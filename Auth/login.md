# Laravel 12 - Login Module

This document explains how to implement a secure login module in Laravel 12 using the built-in authentication system.


---

# Project Structure

```
app/
└── Http/
    └── Controllers/
        └── AuthController.php

resources/
└── views/
    └── auth/
        └── login.blade.php

routes/
└── web.php
```

---

# Step 1 : Create Routes

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

# Step 2 : Create Controller

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
     * Login User
     */
    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required', 'string'],
        ]);

        if (! Auth::attempt(
            $credentials,
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

# Step 3 : Create Login Blade

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

