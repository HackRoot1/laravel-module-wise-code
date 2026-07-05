# Laravel 12 - Change Password
- Suppose User is already logged in and try to change their password

## 1. Routes

```php
use App\Http\Controllers\AuthController;

Route::middleware('auth')->group(function () {

    /*
    |--------------------------------------------------------------------------
    | Change Password Page
    |--------------------------------------------------------------------------
    */

    Route::get('/change-password', [AuthController::class, 'showChangePassword'])
        ->name('password.change');

    /*
    |--------------------------------------------------------------------------
    | Update Password
    |--------------------------------------------------------------------------
    */

    Route::post('/change-password', [AuthController::class, 'changePassword'])
        ->middleware('throttle:5,1')
        ->name('password.change.update');

});
```

---

## 2. Controller

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Auth\Events\PasswordReset;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;
use Illuminate\Validation\Rules\Password as PasswordRule;

class AuthController extends Controller
{
    /**
     * Show Change Password Page
     */
    public function showChangePassword()
    {
        return view('auth.change-password');
    }

    /**
     * Change Password
     */
    public function changePassword(Request $request)
    {
        $request->validate([
            'current_password' => [
                'required',
                'current_password',
            ],

            'password' => [
                'required',
                'confirmed',
                PasswordRule::defaults(),
            ],
        ]);

        DB::transaction(function () use ($request) {

            $user = $request->user();

            $user->forceFill([
                'password' => Hash::make($request->password),
                'remember_token' => Str::random(60),
            ])->save();

            /*
            |--------------------------------------------------------------------------
            | Logout From All Devices
            |--------------------------------------------------------------------------
            */

            DB::table('sessions')
                ->where('user_id', $user->id)
                ->delete();

            event(new PasswordReset($user));

        });

        Auth::logout();

        $request->session()->invalidate();

        $request->session()->regenerateToken();

        return redirect()
            ->route('login.form')
            ->with('status', 'Password changed successfully. Please login again.');
    }
}
```

---

## 3. Change Password Blade

```blade
<form
    action="{{ route('password.change.update') }}"
    method="POST">

    @csrf

    <div>

        <label>Current Password</label>

        <input
            type="password"
            name="current_password"
            autocomplete="current-password"
            required>

        @error('current_password')
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
            id="password_confirmation"
            name="password_confirmation"
            autocomplete="new-password"
            required>

    </div>

    <br>

    <button type="submit">
        Change Password
    </button>

</form>

@if(session('status'))
    <div>
        {{ session('status') }}
    </div>
@endif

<script>

const password = document.getElementById("password");

password.addEventListener("input", function () {

    const strongPassword =
        /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z0-9]).{8,}$/;

    if (strongPassword.test(this.value)) {

        this.setCustomValidity("");

    } else {

        this.setCustomValidity(
            "Password must contain at least 8 characters, one uppercase letter, one lowercase letter, one number and one special character."
        );

    }

});

</script>
```
