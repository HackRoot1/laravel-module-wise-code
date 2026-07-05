
# Laravel 12 + Spatie Roles & Permissions + Super Admin (Recommended Setup)

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

## 3. Run Migrations

```bash
php artisan migrate
```

(Optional)

```bash
php artisan optimize:clear
```

---

# 4. Add HasRoles Trait to User Model

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

# 5. Register Middleware (Laravel 12)

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

> **This step is missing from your guide and is required in Laravel 12.**

---

# 6. Configure Super Admin

The latest Spatie documentation recommends using `Gate::before()`.

In Laravel 12, you can place this in:

```
app/Providers/AppServiceProvider.php
```

or a dedicated service provider if you have one.

Example:

```php
use Illuminate\Support\Facades\Gate;

public function boot(): void
{
    Gate::before(function ($user, string $ability) {
        return $user->hasRole('Super Admin') ? true : null;
    });
}
```

> **Note:** Laravel 12 projects do not include `AuthServiceProvider` by default. Using `AppServiceProvider` is perfectly fine unless you've created an `AuthServiceProvider` yourself.

---

# 7. Create Roles

Seeder:

```php
use Spatie\Permission\Models\Role;

public function run(): void
{
    $roles = [
        'Super Admin',
        'Admin',
        'Customer',
    ];

    foreach ($roles as $role) {
        Role::firstOrCreate([
            'name' => $role,
            'guard_name' => 'web',
        ]);
    }
}
```

---

# 8. Create Permissions

Example:

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

# 9. Assign Permissions to Roles

```php
$admin = Role::findByName('Admin');

$admin->givePermissionTo([
    'view staff',
    'create staff',
    'update staff',
    'delete staff',
]);
```

Super Admin doesn't need explicit permissions because `Gate::before()` bypasses permission checks.

---

# 10. Assign Role to User

```php
$user = User::find(1);

$user->assignRole('Super Admin');
```

or

```php
$user->assignRole('Customer');
```

---

# 11. Protect Routes

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

---

# 12. Use in Controllers

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

# 13. Use in Blade

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

# 14. Clear Permission Cache (Important)

Whenever you add or update roles or permissions:

```bash
php artisan permission:cache-reset
```

or

```bash
php artisan optimize:clear
```

---
