# Laravel Activity Logs (Spatie `laravel-activitylog` v5)

**Corrected from the reference doc:** the reference was written against package v4. A fresh `composer require` on Laravel 12 pulls v5, which changed namespaces and removed `getDescriptionForEvent()` as a model method (see comparison table above this file's intro, or the `UPGRADING.md` from Spatie if you need the full list). Everything below is v5-accurate.

---

## Installation

```bash
composer require spatie/laravel-activitylog
php artisan config:clear
php artisan vendor:publish --provider="Spatie\Activitylog\ActivitylogServiceProvider" --tag="activitylog-migrations"
php artisan migrate
```

Optional — publish the config file if you want to change the activity model class, pruning window, or default log name:

```bash
php artisan vendor:publish --provider="Spatie\Activitylog\ActivitylogServiceProvider" --tag="activitylog-config"
```

---

## Model Changes

**File:** `app/Models/User.php` (or any model you want tracked)

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Activitylog\Models\Concerns\LogsActivity;
use Spatie\Activitylog\Support\LogOptions;

class User extends Authenticatable
{
    use LogsActivity;

    // ...your existing traits, fillable, etc.

    public function getActivitylogOptions(): LogOptions
    {
        return LogOptions::defaults()
            ->logAll()                          // Watch every attribute
            ->logOnlyDirty()                    // But only record what actually changed
            ->dontLogEmptyChanges()              // Skip saving a record if nothing meaningful changed
            ->logExcept(['password', 'remember_token']) // Never expose these even if they change
            ->useLogName('user_logs')
            ->setDescriptionForEvent(fn (string $eventName) => "User has been {$eventName}");
    }
}
```

**What changed from the reference doc's version:**
- `getDescriptionForEvent()` is no longer a separate model method — it's now `->setDescriptionForEvent()` chained inside `getActivitylogOptions()`.
- Added `logExcept(['password', 'remember_token'])`. The reference doc didn't exclude these, and since it used `logAll()`, a password change would otherwise get written into the `activity_log` table's JSON column — hashed, but still not something you want sitting in a log table by default.
- `dontLogEmptyChanges()` replaces `dontSubmitEmptyLogs()`.

---

## What Actually Gets Stored

Real behavior (not what the reference doc's table showed — it only displayed `attributes`, but an update also stores `old`):

```php
$activity->properties;
// [
//   'attributes' => ['email' => 'new@example.com'],
//   'old'        => ['email' => 'old@example.com'],
// ]
```

| id | log_name | description | subject_id | subject_type | causer_id | causer_type | properties | created_at |
|----|-----------|--------------|-------------|----------------|------------|----------------|--------------|-------------|
| 1 | user_logs | User has been updated | 5 | App\Models\User | 1 | App\Models\User | `{"attributes":{"email":"new@example.com"},"old":{"email":"old@example.com"}}` | 2026-07-14 10:00:00 |

(Note: `causer_type` will normally be `App\Models\User`, not `App\Models\Admin`, unless you have a separate Admin model authenticating requests — the reference doc's example table assumed a second guard/model that may not exist in your app.)

---

## Retrieving Logs

```php
use Spatie\Activitylog\Models\Activity;

// All activity
$activities = Activity::all();

// Filter by log name
$userLogs = Activity::where('log_name', 'user_logs')->latest()->get();

// Everything done to one specific model instance
$logs = $user->activities; // via the subject relation, if using LogsActivity on User

// Everything a specific user caused (requires CausesActivity trait — see below)
$causedLogs = $admin->activitiesAsCauser;
```

If you want a model to also be trackable as a **causer** (e.g. an admin who performs actions), add the companion trait:

```php
use Spatie\Activitylog\Models\Concerns\CausesActivity;

class User extends Authenticatable
{
    use LogsActivity, CausesActivity;
}
```

---

## Additional Options

- `logOnly(['name', 'email'])` — track only these fields instead of everything.
- `logExcept(['password'])` — track everything except these fields (different from `logOnly` — this is a subtraction, not a whitelist).
- `dontLogIfAttributesChangedOnly(['updated_at'])` — skip logging if *only* these fields changed (e.g. ignore touch-only saves).
- `useLogName('user_logs')` — group activity by log name so you can filter per model type.

---

## Cleaning Old Logs

```bash
php artisan activitylog:clean
```

Configurable retention period is in `config/activitylog.php` after publishing the config (see Installation section).

---

## Practical Examples (Where You'd Actually Use This)

**1. Auditing admin actions on user accounts**
Track when an admin changes a user's `account_status`, role, or email — directly relevant to the User/Role CRUD controllers already documented. Add the trait to `User` as shown above, and every `store()`/`update()`/`destroy()` call from `UserController` gets logged automatically with no controller changes needed.

**2. Tracking permission/role changes for compliance**
Add `LogsActivity` to your `Role` or a wrapper model so that `givePermissionTo()` / `syncPermissions()` calls from `RoleController::assignPermissions()` leave a paper trail — useful if you ever need to answer "who gave this role delete access, and when."

**3. E-commerce order status changes**
On an `Order` model, log only the `status` field with `logOnly(['status'])->logOnlyDirty()` so you get a clean history of `pending → paid → shipped → delivered` transitions without noise from every other field.

**4. Content moderation / CMS edit history**
On a `Post` or `Page` model, log `title`, `body`, `status` with `useLogName('content_logs')` so editors can see a revision history — pair with `causer_id` to show "last edited by."

**5. Security-sensitive settings changes**
On a `Settings` model (site config, payment gateway keys, etc.), log everything except actual secret values (`logExcept(['api_secret', 'webhook_key'])`) so you know *that* a setting changed and *who* changed it, without leaking the secret itself into a log table.

**6. Debugging support tickets**
When a customer says "my subscription just changed to the wrong plan," query `Activity::where('subject_type', Subscription::class)->where('subject_id', $id)->get()` to see the exact before/after values and who (or what background job) caused it.