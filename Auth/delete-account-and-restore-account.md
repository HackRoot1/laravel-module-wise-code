# Laravel 12 - Delete Account & Revoke Account

---

# Delete Account Request

## 1. Routes

```php
use App\Http\Controllers\AuthController;

Route::middleware('auth')->group(function () {

    /*
    |--------------------------------------------------------------------------
    | Delete Account Page
    |--------------------------------------------------------------------------
    */

    Route::get('/delete-account', [AuthController::class, 'showDeleteAccount'])
        ->name('account.delete');

    /*
    |--------------------------------------------------------------------------
    | Delete Account Request
    |--------------------------------------------------------------------------
    */

    Route::post('/delete-account', [AuthController::class, 'deleteAccount'])
        ->middleware('throttle:5,1')
        ->name('account.delete.request');

});
```

---

## 2. Controller

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class AuthController extends Controller
{
    /**
     * Show Delete Account Page
     */
    public function showDeleteAccount()
    {
        return view('auth.delete-account');
    }

    /**
     * Delete Account Request
     */
    public function deleteAccount(Request $request)
    {
        $request->validate([
            'password' => [
                'required',
                'current_password',
            ],

            'reason' => [
                'nullable',
                'string',
                'max:500',
            ],
        ]);

        $user = $request->user();

        $user->update([
            'delete_requested_at' => now(),
            'delete_reason' => $request->reason,
            'account_status' => 'delete_requested',
        ]);

        Auth::logout();

        $request->session()->invalidate();

        $request->session()->regenerateToken();

        return redirect()
            ->route('login.form')
            ->with('status', 'Your account deletion request has been submitted successfully.');
    }
}
```

---

## 3. Delete Account Blade

```blade
<form
    action="{{ route('account.delete.request') }}"
    method="POST"
    onsubmit="return confirm('Are you sure you want to submit your account deletion request?')">

    @csrf

    <div>

        <label>Current Password</label>

        <input
            type="password"
            name="password"
            autocomplete="current-password"
            required>

        @error('password')
            <div>{{ $message }}</div>
        @enderror

    </div>

    <br>

    <div>

        <label>Reason</label>

        <textarea
            name="reason"
            rows="5"></textarea>

        @error('reason')
            <div>{{ $message }}</div>
        @enderror

    </div>

    <br>

    <button type="submit">
        Submit Delete Request
    </button>

</form>

@if(session('status'))

<div>
    {{ session('status') }}
</div>

@endif
```

---

# 10. Revoke Account Request

## 1. Routes

```php
use App\Http\Controllers\AuthController;

Route::middleware('auth')->group(function () {

    /*
    |--------------------------------------------------------------------------
    | Revoke Account Request
    |--------------------------------------------------------------------------
    */

    Route::post('/revoke-account', [AuthController::class, 'revokeAccount'])
        ->middleware('throttle:5,1')
        ->name('account.revoke');

});
```

---

## 2. Controller

```php
/**
 * Revoke Delete Account Request
 */
public function revokeAccount(Request $request)
{
    $user = $request->user();

    if ($user->account_status !== 'delete_requested') {

        return back()->withErrors([
            'account' => 'No pending delete request found.',
        ]);

    }

    $user->update([
        'delete_requested_at' => null,
        'delete_reason' => null,
        'account_status' => 'active',
    ]);

    return back()->with(
        'status',
        'Your account deletion request has been revoked successfully.'
    );
}
```

---

## 3. Revoke Account Blade

```blade
<form
    action="{{ route('account.revoke') }}"
    method="POST"
    onsubmit="return confirm('Are you sure you want to revoke your account deletion request?')">

    @csrf

    <button type="submit">
        Revoke Delete Request
    </button>

</form>

@if(session('status'))

<div>
    {{ session('status') }}
</div>

@endif
```

---

## 4. Required Migration

```bash
php artisan make:migration add_delete_request_columns_to_users_table --table=users
```

```php
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {

        $table->timestamp('delete_requested_at')
            ->nullable();

        $table->text('delete_reason')
            ->nullable();

        $table->enum('account_status', [
            'active',
            'delete_requested',
            'deleted',
        ])->default('active');

    });
}

public function down(): void
{
    Schema::table('users', function (Blueprint $table) {

        $table->dropColumn([
            'delete_requested_at',
            'delete_reason',
            'account_status',
        ]);

    });
}
```

## 5. Update User Model

```php
protected $fillable = [
    'name',
    'email',
    'password',
    // add following
    'delete_requested_at',
    'delete_reason',
    'account_status',
];
```
