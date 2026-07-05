# Laravel 12 - Logout Module

This document explains how to implement a secure logout module in Laravel 12 using the built-in authentication system.
---

# Project Structure

```
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

# Step 1 : Create Logout Route

**File**

```
routes/web.php
```

```php
use App\Http\Controllers\AuthController;
use Illuminate\Support\Facades\Route;

Route::middleware('auth')->group(function () {

    Route::post('/logout', [AuthController::class, 'logout'])
        ->name('logout');

});
```
---

# Step 2 : Create Logout Method

**File**

```
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
     * Logout User
     */
    public function logout(Request $request)
    {
        // Remove authenticated user
        Auth::logout();

        // Destroy current session
        $request->session()->invalidate();

        // Generate new CSRF token
        $request->session()->regenerateToken();

        // Redirect to Login Page
        return redirect()->route('login.form');
    }
}
```
---

# Step 3 : Create Logout Button

**File**

```
resources/views/profile.blade.php
```

```blade
<form
    action="{{ route('logout') }}"
    method="POST"
    onsubmit="return confirm('Are you sure you want to logout?')">

    @csrf

    <button type="submit">
        Logout
    </button>

</form>
```

---
