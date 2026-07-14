# Laravel 12 + Spatie Roles & Permissions + Teams + Super Admin (Corrected)

**Change from the version reviewed:** Step 3's config keys were wrong. `team_foreign_key` is not a top-level key in the current `spatie/laravel-permission` config — it's nested inside `column_names`. As written, the original config would not actually apply the custom key at all (the package reads `column_names.team_foreign_key`, not a top-level `team_foreign_key`). Fixed below. Everything else in the original held up against current package source/docs.

---

## 1. Install the Package

```bash
composer require spatie/laravel-permission
```

---

## 2. Publish the Configuration & Migrations

```bash
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
```

This publishes:

* `config/permission.php`
* Permission migrations

---

## 3. Enable Teams (Before Migrating)

> **Important:** the `teams` config must be enabled **before** you run the migration for the first time. If you've already migrated, see the "Upgrading an Existing Install" note below.

Open `config/permission.php` and set:

```php
// config/permission.php
'teams' => true,

'column_names' => [
    // ...other keys already in this array, leave them...
    'team_foreign_key' => 'team_id', // this is already the default value — only change if you have a naming conflict
],
```

**Corrected from original:** `team_foreign_key` must live inside the `column_names` array. Placing it top-level (as in the reviewed version) means the package silently ignores it and falls back to its default.

### Upgrading an Existing (Already Migrated) Install

If you already ran `php artisan migrate` before enabling teams, run:

```bash
php artisan permission:setup-teams
php artisan migrate
```

This generates and runs an `add_teams_fields` migration that adds the `team_id` column to the roles/permissions pivot tables.

---

## 4. Run Migrations

```bash
php artisan migrate
```

(Optional)

```bash
php artisan optimize:clear
```

---

## 5. Add HasRoles Trait to User Model

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasRoles;

    // ...
}
```

If you're using a custom guard, make sure `guard_name` matches the guard configured in `config/auth.php`.

---

## 6. Create a Team Model

Teams don't need `HasRoles` on the Team model itself — that trait goes on `User`. Just create a normal Eloquent model for your team/tenant:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Team extends Model
{
    protected $fillable = ['name'];

    public function users()
    {
        return $this->belongsToMany(User::class);
        // or hasMany, depending on your team-membership structure
    }
}
```

Make sure you have a `teams` table (and, typically, a `team_user` pivot table) migrated for your own team/membership logic — this is separate from the `team_id` column Spatie adds to its own tables.

---

## 7. Register Middleware (Laravel 12)

Laravel 11/12 **does not use `Kernel.php`** for middleware aliases.

Open:

```
bootstrap/app.php
```

Register the middleware aliases:

```php
use Illuminate\Foundation\Configuration\Middleware;

->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'role' => \Spatie\Permission\Middleware\RoleMiddleware::class,
        'permission' => \Spatie\Permission\Middleware\PermissionMiddleware::class,
        'role_or_permission' => \Spatie\Permission\Middleware\RoleOrPermissionMiddleware::class,
    ]);
})
```

---

## 8. Set the Active Team on Every Request

With teams enabled, Spatie needs to know **which team's** roles/permissions to check on each request. The standard approach:

1. Store the active `team_id` in the session when the user logs in / switches team.
2. Add a middleware that calls `setPermissionsTeamId()` before any role/permission check happens.

Create the middleware:

```bash
php artisan make:middleware SetPermissionsTeam
```

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class SetPermissionsTeam
{
    public function handle(Request $request, Closure $next)
    {
        if (auth()->check()) {
            // team_id set on login / team switch, e.g. session(['team_id' => $team->id]);
            setPermissionsTeamId(session('team_id'));
        }

        return $next($request);
    }
}
```

Register it and, critically, give it priority **before** `SubstituteBindings` so route-model-bound permission checks see the correct team:

```php
// bootstrap/app.php
use App\Http\Middleware\SetPermissionsTeam;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Routing\Middleware\SubstituteBindings;

->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'role' => \Spatie\Permission\Middleware\RoleMiddleware::class,
        'permission' => \Spatie\Permission\Middleware\PermissionMiddleware::class,
        'role_or_permission' => \Spatie\Permission\Middleware\RoleOrPermissionMiddleware::class,
    ]);

    $middleware->web(append: [
        SetPermissionsTeam::class,
    ]);

    $middleware->priority([
        SetPermissionsTeam::class,
        SubstituteBindings::class,
        // ...any other middleware that must run after, e.g. \Illuminate\Auth\Middleware\Authorize::class,
    ]);
})
```

> **Using Livewire?** You may also need to register this middleware as persistent. See [Livewire docs: Configuring Persistent Middleware](https://livewire.laravel.com/docs/security#configuring-persistent-middleware).

---

## 9. Configure Super Admin

With teams enabled, a "global" Super Admin role (one that applies across every team) needs `team_id = null`, and must still be explicitly assigned per team — see step 13.

Spatie's own documentation recommends `Gate::before()` for this. Place it in `app/Providers/AppServiceProvider.php`:

```php
use Illuminate\Support\Facades\Gate;

public function boot(): void
{
    Gate::before(function ($user, string $ability) {
        return $user->hasRole('Super Admin') ? true : null;
    });
}
```

> **Note:** Laravel 12 projects do not include `AuthServiceProvider` by default. Using `AppServiceProvider` is fine unless you've created an `AuthServiceProvider` yourself.

**Worth knowing:** `Gate::before` only intercepts calls that go through Laravel's Gate — `can()`, `@can`, `authorize()`. It does **not** intercept direct calls like `hasPermissionTo()`. That's why step 13 still explicitly assigns the "Super Admin" role per team — role-based middleware (`role:Super Admin`) and `@hasrole`/`@role` blade directives check the role directly, not through the Gate, so the role assignment still has to be real, not just implied by the bypass.

---

## 10. Create Roles (Global vs Team-Scoped)

With teams enabled, roles can either be **global** (`team_id = null`, usable/assignable on any team) or **team-scoped** (tied to one specific `team_id`, and different teams can each have their own role with the same name).

```php
use Spatie\Permission\Models\Role;

// Global role — not tied to any specific team (e.g. Super Admin)
Role::firstOrCreate([
    'name' => 'Super Admin',
    'team_id' => null,
    'guard_name' => 'web',
]);

// Team-scoped roles — created per team, e.g. inside a "create team" flow
$teamId = $team->id;

foreach (['Admin', 'Member'] as $roleName) {
    Role::firstOrCreate([
        'name' => $roleName,
        'team_id' => $teamId,
        'guard_name' => 'web',
    ]);
}
```

> Role/permission lookups always use whatever `team_id` is currently active via `setPermissionsTeamId()`, so make sure it's set correctly before creating or querying team-scoped roles (e.g. wrap seeding logic in `setPermissionsTeamId($teamId)` / restore afterwards).

---

## 11. Create Permissions

Permissions work the same way — they're usually global (shared definitions like `'view staff'`), while it's the **role-to-permission** and **user-to-role** assignments that get scoped to a team.

```php
use Spatie\Permission\Models\Permission;

$permissions = [
    'view staff',
    'create staff',
    'update staff',
    'delete staff',
];

foreach ($permissions as $permission) {
    Permission::firstOrCreate([
        'name' => $permission,
        'guard_name' => 'web',
    ]);
}
```

---

## 12. Assign Permissions to Roles

```php
$teamId = $team->id;
setPermissionsTeamId($teamId);

$admin = Role::where('name', 'Admin')->where('team_id', $teamId)->first();

$admin->givePermissionTo([
    'view staff',
    'create staff',
    'update staff',
    'delete staff',
]);
```

Super Admin doesn't need explicit permissions to pass `can()`/`@can` checks — `Gate::before` handles those. It still needs the role itself actually assigned per team (see note in step 9) for role-based middleware/directives to work.

---

## 13. Assign Role to User (Within a Team)

Role assignment is scoped to whatever `team_id` is currently active, so set it first:

```php
$user = User::find(1);
$teamId = $team->id;

setPermissionsTeamId($teamId);
$user->unsetRelation('roles')->unsetRelation('permissions'); // clear cached relations before assigning

$user->assignRole('Admin');
```

### Assigning a Global Super Admin to a New Team

Because global roles still require a `team_id` to be linked through the pivot table, a "Super Admin" must be explicitly (re)assigned whenever a new team is created:

```php
// e.g. inside a Team model's "created" event
protected static function booted(): void
{
    static::created(function (Team $team) {
        $previousTeamId = getPermissionsTeamId();

        setPermissionsTeamId($team->id);
        User::find(1)->assignRole('Super Admin');

        setPermissionsTeamId($previousTeamId);
    });
}
```

---

## 14. Switching the Active Team

Whenever a user switches teams (e.g. via a team switcher in the UI), update the session and reset the active team, then clear cached relations so the next role/permission check re-queries for the new team:

```php
session(['team_id' => $newTeamId]);
setPermissionsTeamId($newTeamId);

$user->unsetRelation('roles')->unsetRelation('permissions');
```

---

## 15. Protect Routes

By Role

```php
Route::middleware(['auth', 'role:Admin|Super Admin'])
    ->group(function () {
        //
    });
```

By Permission

```php
Route::middleware(['auth', 'permission:delete staff'])
    ->group(function () {
        //
    });
```

> As long as `SetPermissionsTeam` middleware runs before these (per step 8's priority list), the role/permission middleware will automatically check against the currently active team — no need to pass a team ID in the route middleware string. Remember: this middleware checks `hasRole()` directly, so `Super Admin` needs the role actually assigned for the current team, not just the `Gate::before` bypass.

---

## 16. Use in Controllers

```php
if (auth()->user()->can('update staff')) {
    // Allowed
}
```

or

```php
$this->authorize('update staff');
```

---

## 17. Use in Blade

```blade
@role('Super Admin')
    ...
@endrole
```

```blade
@hasrole('Admin')
    ...
@endhasrole
```

```blade
@can('view staff')
    ...
@endcan
```

```blade
@can('update staff')
    ...
@endcan
```

```blade
@can('delete staff')
    ...
@endcan
```

---

## 18. Clear Permission Cache (Important)

Whenever you add or update roles or permissions:

```bash
php artisan permission:cache-reset
```

or

```bash
php artisan optimize:clear
```

> With teams, Spatie's cache is keyed per-team automatically, but it's still good practice to reset the cache after seeding or bulk role/permission changes.

---

## Notes & Gotchas

- **Config structure matters:** `team_foreign_key` goes inside `column_names` in `config/permission.php`, not top-level. Putting it top-level means it's silently ignored.
- **Order matters:** enable `'teams' => true` in `config/permission.php` *before* the first migration. Retrofitting an existing install means running `php artisan permission:setup-teams` first.
- **`Gate::before` doesn't cover everything:** it only intercepts Gate-routed checks (`can()`, `@can`, `authorize()`). Role-based middleware and `@role`/`@hasrole` still check the role directly — Super Admin needs the role actually assigned per team.
- **Always set the team before checking:** any `hasRole()`, `can()`, `hasPermissionTo()`, etc. call reads from whatever `team_id` `setPermissionsTeamId()` last set — if this isn't set (e.g. in a queued job or console command), calls will silently check against the "no team" / null scope.
- **Unset relations after switching teams:** `$user->roles` and `$user->permissions` are cached Eloquent relations — always call `unsetRelation('roles')->unsetRelation('permissions')` after `setPermissionsTeamId()` changes, or you'll see stale data from the previous team.
- **Queued jobs / console commands:** these don't have your web middleware, so you must call `setPermissionsTeamId($teamId)` manually at the start of the job/command before doing any role/permission work.