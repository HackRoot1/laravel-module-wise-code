# Adding `rappasoft/laravel-authentication-log` (v6.x) to an Existing Laravel Project

Tracks login/logout attempts, IP, device fingerprint, and location per user; can send new-device/suspicious-login notifications.

**Requirements:** PHP 8.1+, Laravel 11.x/12.x/13.x. If you're on Laravel 10.x, use `composer require rappasoft/laravel-authentication-log:^3.0` instead — the steps below are for the current v6 line.

---

## 1. Install

```bash
composer require rappasoft/laravel-authentication-log
```

Optional — only if you want IP-to-location resolution:

```bash
composer require torann/geoip
```

Optional — only if you want SMS notifications via Vonage for new-device alerts:

```bash
composer require laravel/vonage-notification-channel
```

Skip both if you only want basic login/logout/failed-attempt logging.

---

## 2. Publish Config, Migrations, and (optionally) Views

```bash
php artisan vendor:publish --provider="Rappasoft\LaravelAuthenticationLog\LaravelAuthenticationLogServiceProvider" --tag="authentication-log-config"

php artisan vendor:publish --provider="Rappasoft\LaravelAuthenticationLog\LaravelAuthenticationLogServiceProvider" --tag="authentication-log-migrations"

# Only needed if you plan to use the package's pre-built device-management views
php artisan vendor:publish --provider="Rappasoft\LaravelAuthenticationLog\LaravelAuthenticationLogServiceProvider" --tag="authentication-log-views"
```

## 3. Run Migrations

```bash
php artisan migrate
```

Creates an `authentication_log` table with columns for `ip_address`, `user_agent`, `location`, `login_at`, `logout_at`, `device_id`, `device_name`, `is_trusted`, `last_activity_at`, `is_suspicious`, `suspicious_reason`, plus a polymorphic `authenticatable` relation.

---

## 4. Model Changes

**File:** `app/Models/User.php`

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Rappasoft\LaravelAuthenticationLog\Traits\AuthenticationLoggable;

class User extends Authenticatable
{
    use Notifiable, AuthenticationLoggable;

    // ...your existing traits, fillable, casts, etc.
}
```

`Notifiable` is required alongside `AuthenticationLoggable` — the package sends its new-device/suspicious-login notifications through Laravel's standard notification system, so without `Notifiable` those notifications have nowhere to dispatch to.

No new migration needed on the `users` table itself — everything the package tracks lives in its own `authentication_log` table, linked polymorphically. This is the cleanly separated part; nothing here touches your existing `users` schema.

---

## 5. Config Highlights

`config/authentication-log.php` (after publishing) lets you control, among other things:

- Which events are logged (`Login`, `Logout`, `Failed`, `OtherDeviceLogout` by default — overridable if you have custom auth events)
- Notification channels for new-device alerts (`mail`, `slack`, `vonage` for SMS)
- Whether suspicious-activity detection is enabled
- Database connection to use for the `authentication_log` table (defaults to your app's default connection)

Review it before going to production — the defaults notify on every new device, which may be noisy if your users frequently switch networks/browsers.

---

## 6. Usage Examples

**Get login stats for a user:**
```php
$user = User::find(1);
$stats = $user->getLoginStats();
// total_logins, failed_logins, last_login_at, etc.
```

**Query scopes for filtering logs:**
```php
use Rappasoft\LaravelAuthenticationLog\Models\AuthenticationLog;

AuthenticationLog::successful()->get();
AuthenticationLog::failed()->get();
AuthenticationLog::suspicious()->get();
AuthenticationLog::recent()->get();
AuthenticationLog::forIp('203.0.113.10')->get();
```

**Protect a route so only trusted devices can access it (e.g. account settings):**
```php
Route::middleware(['auth', 'trusted.device'])->group(function () {
    Route::get('/account/security', [AccountController::class, 'security']);
});
```
(Confirm the exact middleware alias name in the published config — it may need registering in `bootstrap/app.php` the same way you did for Spatie's role/permission middleware, since Laravel 11/12 doesn't use `Kernel.php` for this.)

**Export logs (CSV/JSON) for a compliance request:**
```php
$user->authentications()->recent()->get()->toCsv(); // check exact export method name in docs for your installed version — this varies between v5 and v6
```

---

## What This Doesn't Touch

Same isolation pattern as the 2FA and Activity Log packages: no changes to your `AuthController`/login logic are required. The package hooks into Laravel's built-in `Login`, `Logout`, and `Failed` auth events automatically — it listens, it doesn't intercept. If your existing login flow uses `Auth::attempt()` (as in your current `AuthController`), those events already fire, so this "just works" without touching that controller.