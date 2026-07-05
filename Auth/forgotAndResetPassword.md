# Laravel 12 - Forgot Password & Reset Password

This document explains how to implement the **Forgot Password** functionality in Laravel 12 using Laravel's built-in Password Broker.

---

# Project Structure

```text
app/
├── Http/
│   └── Controllers/
│       └── AuthController.php
│
resources/
└── views/
    └── auth/
        ├── login.blade.php
        └── forgot-password.blade.php
        └── reset-password.blade.php

routes/
└── web.php

config/
└── mail.php

.env
```



# Step 1 : Configure Mail

Laravel sends password reset links through its mail system.

Update your `.env` file.

```env
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=your-email@gmail.com
MAIL_FROM_NAME="${APP_NAME}"
```

> **Note:** If you use Gmail, generate an **App Password** instead of using your account password.


# Step 2 : Create Routes

**File**

```text
routes/web.php
```

```php
use App\Http\Controllers\AuthController;
use Illuminate\Support\Facades\Route;

Route::middleware('guest')->group(function () {

    /*
    |--------------------------------------------------------------------------
    | Forgot Password Page
    |--------------------------------------------------------------------------
    */

    Route::get('/forgot-password', [AuthController::class, 'showForgotPassword'])
        ->name('password.request');

    /*
    |--------------------------------------------------------------------------
    | Send Password Reset Link
    |--------------------------------------------------------------------------
    */

    Route::post('/forgot-password', [AuthController::class, 'forgotPassword'])
        ->middleware('throttle:5,1')
        ->name('password.email');

    /*
    |--------------------------------------------------------------------------
    | Reset Password Page
    |--------------------------------------------------------------------------
    */

    Route::get('/reset-password/{token}', [AuthController::class, 'showResetPassword'])
        ->name('password.reset');

    /*
    |--------------------------------------------------------------------------
    | Update Password
    |--------------------------------------------------------------------------
    */

    Route::post('/reset-password', [AuthController::class, 'resetPassword'])
        ->middleware('throttle:5,1')
        ->name('password.update');

});
```

---

# Step 3 : Controller

**File**

```text
app/Http/Controllers/AuthController.php
```

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Password;
use Illuminate\Validation\Rules\Password as PasswordRule;
use Illuminate\Auth\Events\PasswordReset;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

class AuthController extends Controller
{
    /**
     * Show Forgot Password Page
     */
    public function showForgotPassword()
    {
        return view('auth.forgot-password');
    }

    /**
     * Send Password Reset Link
     */
    public function forgotPassword(Request $request)
    {
        $request->validate([
            'email' => ['required', 'email'],
        ]);

        $status = Password::sendResetLink(
            $request->only('email')
        );

        return $status === Password::RESET_LINK_SENT
            ? back()->with('status', __($status))
            : back()->withErrors([
                'email' => __($status),
            ]);
    }

     
    /**
     * Show Reset Password Page
     */
    public function showResetPassword(Request $request, string $token)
    {
        return view('auth.reset-password', [
            'token' => $token,
            'email' => $request->email,
        ]);
    }


    /**
     * Reset Password
     */
    public function resetPassword(Request $request)
    {
        $request->validate([
            'token' => ['required'],
            'email' => ['required', 'email'],
            'password' => [
                'required',
                'confirmed',
                PasswordRule::defaults(),
            ],
        ]);
    
        $status = Password::reset(
            $request->only(
                'email',
                'password',
                'password_confirmation',
                'token'
            ),
            function ($user, $password) {
    
                $user->forceFill([
                    'password' => Hash::make($password),
                    'remember_token' => Str::random(60),
                ])->save();
    
                event(new PasswordReset($user));
            }
        );
    
        return $status === Password::PASSWORD_RESET
            ? redirect()->route('login.form')
                ->with('status', __($status))
            : back()->withErrors([
                'email' => __($status),
            ]);
    }

}
```


# Step 4 : Create Forgot Password & Reset Password Blade

**File**

```
resources/views/auth/forgot-password.blade.php
```

```blade
<form action="{{ route('password.email') }}" method="POST">

    @csrf

    <div>

        <label>Email</label>

        <input
            type="email"
            name="email"
            value="{{ old('email') }}"
            autocomplete="email"
            required
            autofocus>

        @error('email')
            <div>{{ $message }}</div>
        @enderror

    </div>

    <br>

    <button type="submit">
        Send Reset Password Link
    </button>

</form>

@if (session('status'))
    <div>
        {{ session('status') }}
    </div>
@endif
```



**File**

```
resources/views/auth/reset-password.blade.php
```

```blade
<form action="{{ route('password.update') }}" method="POST">

    @csrf

    <input
        type="hidden"
        name="token"
        value="{{ $token }}">

    <div>

        <label>Email</label>

        <input
            type="email"
            name="email"
            value="{{ old('email', $email) }}"
            required>

        @error('email')
            <div>{{ $message }}</div>
        @enderror

    </div>

    <br>

    <div>

        <label>New Password</label>

        <input
            type="password"
            id="password"
            name="password"
            autocomplete="new-password"
            required>

        @error('password')
            <div>{{ $message }}</div>
        @enderror

    </div>

    <br>

    <div>

        <label>Confirm Password</label>

        <input
            type="password"
            name="password_confirmation"
            id="password_confirmation"
            autocomplete="new-password"
            required>

    </div>

    <br>

    <button type="submit">
        Reset Password
    </button>

</form>

<script>

const password = document.getElementById("password");

password.addEventListener("input", function () {

    const strongPassword =
        /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z0-9]).{8,}$/;

    if (strongPassword.test(this.value)) {

        this.setCustomValidity("");

    } else {

        this.setCustomValidity(
            "Password must contain uppercase, lowercase, number and special character."
        );

    }

});

</script>
```

> **Important:** Frontend validation can be bypassed. Always keep the backend validation in place.
