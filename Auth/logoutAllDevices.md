# Laravel 12 - Logout from All Devices

This document explains how to implement a **Logout from All Devices** feature in Laravel 12.

Unlike a normal logout, this feature signs the user out from **every authenticated device**, including the current one.

---

# Prerequisites

To use `Auth::logoutOtherDevices()`, ensure:

* Passwords are hashed using Laravel's default hashing.
* The `auth.session` middleware (AuthenticateSession) is enabled where required.
* The user provides their current password.

---

# Project Structure

```text
app/
└── Http/
    └── Controllers/
        └── AuthController.php

resources/
└── views/
    └── profile.blade.php

routes/
└── web.php
```

---

# Step 1 : Create Route

**File**

```text
routes/web.php
```

```php
use App\Http\Controllers\AuthController;
use Illuminate\Support\Facades\Route;

Route::middleware('auth')->group(function () {

    Route::post('/all-logout', [AuthController::class, 'allLogout'])
        ->name('all-logout');

});
```
---

# Step 2 : Controller

**File**

```text
app/Http/Controllers/AuthController.php
```

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class AuthController extends Controller
{
    /**
     * Logout from all devices
     */
    public function allLogout(Request $request)
    {
        $request->validate([
            'password' => ['required', 'current_password'],
        ]);

        // Logout from all other authenticated devices
        Auth::logoutOtherDevices($request->password);

        // Logout from current device
        Auth::logout();

        // Destroy current session
        $request->session()->invalidate();

        // Generate new CSRF token
        $request->session()->regenerateToken();

        return redirect()->route('login.form');
    }
}
```

# Step 3 : Blade

**File**

```text
resources/views/profile.blade.php
```

```blade
<form
    action="{{ route('all-logout') }}"
    method="POST"
    onsubmit="return confirm('Are you sure you want to logout from all devices?')">

    @csrf

    <div>
        <label>Current Password</label>

        <input
            type="password"
            name="password"
            required>

        @error('password')
            <div style="color:red">
                {{ $message }}
            </div>
        @enderror
    </div>

    <br>

    <button type="submit">
        Logout from All Devices
    </button>

</form>
```
