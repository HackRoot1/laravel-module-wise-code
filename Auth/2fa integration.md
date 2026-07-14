# Adding 2FA to an Existing Laravel Project

Package: `pragmarx/google2fa-laravel` (TOTP-based, no Fortify/Breeze dependency, doesn't hijack your existing auth flow)

Reality check first: you can't make 2FA 100% non-interfering — it has to sit in the login path somewhere. This approach minimizes blast radius by using a separate middleware + separate table, so you're not editing your existing `LoginController`/`AuthenticatedSessionController` logic, just wrapping it.

---

## 1. Install package
```bash
composer require pragmarx/google2fa-laravel
```

## 2. Publish config
```bash
php artisan vendor:publish --provider="PragmaRX\Google2FALaravel\ServiceProvider"
```
Creates `config/google2fa.php`. Leave defaults unless you know why you're changing them.

## 3. Add columns via a new migration (don't touch existing users migration)
```bash
php artisan make:migration add_2fa_to_users_table --table=users
```
```php
Schema::table('users', function (Blueprint $table) {
    $table->string('google2fa_secret')->nullable();
    $table->boolean('google2fa_enabled')->default(false);
    $table->timestamp('google2fa_enabled_at')->nullable();
});
```
Run:
```bash
php artisan migrate
```

## 4. Extend User model (additive only — no changes to existing methods)
Add to `app/Models/User.php`:
```php
protected $casts = [
    'google2fa_enabled' => 'boolean',
];

public function hasTwoFactorEnabled(): bool
{
    return $this->google2fa_enabled;
}
```

## 5. Create a dedicated controller (new file, no edits to existing auth controllers)
```bash
php artisan make:controller Auth/TwoFactorController
```
Handles:
- `showSetup()` – generate QR code + secret, show enable form
- `enable()` – verify first code, set `google2fa_enabled = true`
- `showChallenge()` – 2FA code entry page (post-login)
- `verify()` – check code, mark session as 2FA-passed
- `disable()` – clear secret, set flag false

## 6. Create a dedicated middleware (this is the "wrapper", not a rewrite)
```bash
php artisan make:middleware EnsureTwoFactorIsVerified
```
```php
public function handle($request, Closure $next)
{
    $user = $request->user();

    if ($user && $user->hasTwoFactorEnabled() && !session('2fa_passed')) {
        return redirect()->route('2fa.challenge');
    }

    return $next($request);
}
```
Register as route middleware (`app/Http/Kernel.php`), e.g. alias `2fa`.

## 7. Apply middleware only where needed
Don't touch your global `web` middleware group. Instead, apply `2fa` selectively:
```php
Route::middleware(['auth', '2fa'])->group(function () {
    // protected routes
});
```
This is the only "interference point" — and it's opt-in per route group, not global.

## 8. Add routes (new file: `routes/2fa.php`, require it from `web.php`)
```php
Route::middleware('auth')->group(function () {
    Route::get('/2fa/setup', [TwoFactorController::class, 'showSetup'])->name('2fa.setup');
    Route::post('/2fa/enable', [TwoFactorController::class, 'enable'])->name('2fa.enable');
    Route::get('/2fa/challenge', [TwoFactorController::class, 'showChallenge'])->name('2fa.challenge');
    Route::post('/2fa/verify', [TwoFactorController::class, 'verify'])->name('2fa.verify');
    Route::post('/2fa/disable', [TwoFactorController::class, 'disable'])->name('2fa.disable');
});
```

## 9. QR code generation (setup view)
```php
use PragmaRX\Google2FALaravel\Support\Authenticator;

$google2fa = app('pragmarx.google2fa');
$secret = $google2fa->generateSecretKey();
$qrCodeUrl = $google2fa->getQRCodeUrl(config('app.name'), $user->email, $secret);
```
Render QR via `bacon/bacon-qr-code` (installed as a dependency) or any QR view helper.

## 10. Verify code logic (used in both `enable()` and `verify()`)
```php
$valid = app('pragmarx.google2fa')->verifyKey($user->google2fa_secret, $request->input('code'));
```

## 11. Session flag on successful login (only new code, added where you already fire `Auth::login`)
After existing login succeeds, if `hasTwoFactorEnabled()` is true, don't set `session('2fa_passed')` yet — middleware in step 6 will force redirect to challenge. Once challenge passes, set `session(['2fa_passed' => true])`.

This is the **one unavoidable touch point** in your existing login flow — a single conditional check, not a rewrite.

## 12. Recovery codes (optional but recommended)
- New table `two_factor_recovery_codes` (user_id, code, used_at)
- Generate 8–10 one-time codes on enable, allow use instead of TOTP if lost device

## 13. Test
- Enable 2FA on a test user
- Log out, log back in → should hit `/2fa/challenge`
- Wrong code → rejected
- Correct code → session flagged, access restored
- Disable → confirm middleware no longer blocks

---

### What actually touches existing code
- 1 conditional at your login success point (step 11)
- Middleware registration in `Kernel.php`
- Route middleware applied to protected groups

Everything else (migration, model addition, controller, middleware class, routes, views) is net-new and deletable without breaking existing auth if you ever rip it out.