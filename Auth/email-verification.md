# Laravel 12 - Email Verification

Laravel provides a **built-in Email Verification** feature that is secure, production-ready, and recommended over implementing a custom OTP verification flow.

---

## 1. User Model

Implement the `MustVerifyEmail` interface.

```php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable implements MustVerifyEmail
{
    //
}
```

---

## 2. Routes

```php
use App\Http\Controllers\AuthController;
use Illuminate\Foundation\Auth\EmailVerificationRequest;
use Illuminate\Http\Request;

Route::middleware('auth')->group(function () {

    /*
    |--------------------------------------------------------------------------
    | Email Verification Notice
    |--------------------------------------------------------------------------
    */

    Route::get('/email/verify', [AuthController::class, 'showEmailVerification'])
        ->name('verification.notice');

    /*
    |--------------------------------------------------------------------------
    | Send Verification Email
    |--------------------------------------------------------------------------
    */

    Route::post('/email/verification-notification', [AuthController::class, 'sendVerificationEmail'])
        ->middleware('throttle:6,1')
        ->name('verification.send');

    /*
    |--------------------------------------------------------------------------
    | Verify Email
    |--------------------------------------------------------------------------
    */

    Route::get('/email/verify/{id}/{hash}', function (EmailVerificationRequest $request) {

        $request->fulfill();

        return redirect()->route('dashboard');

    })->middleware([
        'signed',
        'throttle:6,1',
    ])->name('verification.verify');

});
```

---

## 3. Controller

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class AuthController extends Controller
{
    /**
     * Show Email Verification Page
     */
    public function showEmailVerification()
    {
        return view('auth.verify-email');
    }

    /**
     * Send Verification Email
     */
    public function sendVerificationEmail(Request $request)
    {
        if ($request->user()->hasVerifiedEmail()) {

            return redirect()
                ->route('dashboard');

        }

        $request->user()
            ->sendEmailVerificationNotification();

        return back()->with(
            'status',
            'Verification link has been sent successfully.'
        );
    }
}
```

---

## 4. Verify Email Blade

```blade
@if(session('status'))

<div>
    {{ session('status') }}
</div>

@endif

@if(auth()->user()->hasVerifiedEmail())

<div>

    Your email has already been verified.

</div>

@else

<p>

Please verify your email address.

</p>

<form
    action="{{ route('verification.send') }}"
    method="POST">

    @csrf

    <button type="submit">

        Send Verification Email

    </button>

</form>

@endif
```

---

## 5. Protect Verified Routes

```php
Route::middleware([
    'auth',
    'verified',
])->group(function () {

    Route::get('/dashboard', function () {

        return view('dashboard');

    })->name('dashboard');

});
```

---

## 6. Profile Page

```blade
@if(auth()->user()->hasVerifiedEmail())

<span>

Email Verified

</span>

@else

<a href="{{ route('verification.notice') }}">

Verify Email

</a>

@endif
```
