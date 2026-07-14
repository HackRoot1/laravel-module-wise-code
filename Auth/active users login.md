# Laravel 12 - Login Module (Active Users Only)

This document explains how to implement a secure login module in Laravel 12 using the built-in authentication system, restricted to users with `is_active = true`.

**Security note on approach:** Inactive-user rejection is done via the `Auth::attempt()` credentials array, not a post-login check. This means an inactive user gets the same generic "invalid credentials" error as a wrong password — no enumeration leak, but also no explicit "your account is disabled" message. If you want that explicit message instead, you need the post-attempt check variant (different tradeoff — flagged at the bottom).

---

# Project Structure

```
app/
└── Http/
    └── Controllers/
        └── AuthController.php

database/
└── migrations/
    └── xxxx_xx_xx_add_is_active_to_users_table.php

resources/
└── views/
    └── auth/
        └── login.blade.php

routes/
└── web.php
```

---

# Step 1 : Add `is_active` column (new migration — don't edit the original users migration)

```bash
php artisan make:migration add_is_active_to_users_table --table=users
```

```php
Schema::table('users', function (Blueprint $table) {
    $table->boolean('is_active')->default(true)->after('email_verified_at');
});
```

```bash
php artisan migrate
```

Add to `$casts` in `app/Models/User.php`:
```php
protected $casts = [
    'is_active' => 'boolean',
];
```

---

# Step 2 : Create Routes

**File**

```
routes/web.php
```

```php
use App\Http\Controllers\AuthController;
use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| Guest Routes
|--------------------------------------------------------------------------
*/

Route::middleware('guest')->group(function () {

    // Login Form
    Route::get('/login', [AuthController::class, 'showLogin'])
        ->name('login.form');

    // Login
    Route::post('/login', [AuthController::class, 'login'])
        ->middleware('throttle:5,1')
        ->name('login');
});

/*
|--------------------------------------------------------------------------
| Protected Routes
|--------------------------------------------------------------------------
*/

Route::middleware('auth')->group(function () {

    Route::get('/dashboard', function () {
        return view('dashboard');
    })->name('dashboard');

});
```

# Step 3 : Create Controller

**File**

```
app/Http/Controllers/AuthController.php
```

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    /**
     * Show Login Form
     */
    public function showLogin()
    {
        return view('auth.login');
    }

    /**
     * Login User (active users only)
     */
    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required', 'string'],
        ]);

        // is_active added directly to the attempt query.
        // Inactive users fail exactly like a wrong password — no distinct error message.
        $attemptCredentials = array_merge($credentials, ['is_active' => true]);

        if (! Auth::attempt(
            $attemptCredentials,
            $request->boolean('remember')
        )) {
            throw ValidationException::withMessages([
                'email' => __('auth.failed'),
            ]);
        }

        // Prevent Session Fixation
        $request->session()->regenerate();

        return redirect()->intended(route('dashboard'));
    }
}
```

# Step 4 : Create Login Blade

**File**

```
resources/views/auth/login.blade.php
```

```blade
<!DOCTYPE html>
<html>

<head>
    <title>Login</title>
</head>

<body>

    <h2>Login</h2>

    <form action="{{ route('login') }}" method="POST">

        @csrf

        <div>

            <label>Email</label>

            <input
                type="email"
                name="email"
                value="{{ old('email') }}"
                placeholder="Enter Email"
                required
                autofocus>

            @error('email')
                <div style="color:red">
                    {{ $message }}
                </div>
            @enderror

        </div>

        <br>

        <div>

            <label>Password</label>

            <input
                type="password"
                name="password"
                placeholder="Enter Password"
                required>

            @error('password')
                <div style="color:red">
                    {{ $message }}
                </div>
            @enderror

        </div>

        <br>

        <div>

            <label>

                <input
                    type="checkbox"
                    name="remember">

                Remember Me

            </label>

        </div>

        <br>

        <button type="submit">
            Login
        </button>

    </form>

</body>

</html>
```

---

# Alternative: Explicit "Account Disabled" Message

If you'd rather tell inactive users *why* they can't log in (worse for security, better for UX/support load), replace the controller logic with:

```php
public function login(Request $request)
{
    $credentials = $request->validate([
        'email' => ['required', 'email'],
        'password' => ['required', 'string'],
    ]);

    if (! Auth::attempt($credentials, $request->boolean('remember'))) {
        throw ValidationException::withMessages([
            'email' => __('auth.failed'),
        ]);
    }

    if (! Auth::user()->is_active) {
        Auth::logout();
        $request->session()->invalidate();

        throw ValidationException::withMessages([
            'email' => 'Your account has been disabled. Contact support.',
        ]);
    }

    $request->session()->regenerate();

    return redirect()->intended(route('dashboard'));
}
```

This confirms to anyone attempting login that the email/password combo is valid even when blocked — decide if that tradeoff is acceptable for your use case before using this version.