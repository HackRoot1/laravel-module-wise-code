# Laravel 12 - User Registration

## 1. Install Required Packages

### Install Roles & Permissions

```bash
composer require spatie/laravel-permission
```

Publish

```bash
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
```

Run Migration

```bash
php artisan migrate
```

---

### Install Avatar Package

```bash
composer require laravolt/avatar
```

Publish Configuration

```bash
php artisan vendor:publish --provider="Laravolt\Avatar\ServiceProvider"
```

---

# 2. Update Users Migration

Add the following columns.

```php
$table->string('username')->unique();

$table->string('phone', 20)->unique();

$table->string('avatar')->nullable();
```

Run Migration

```bash
php artisan migrate
```

---

# 3. User Model

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasRoles;

    protected $fillable = [
        'name',
        'username',
        'phone',
        'email',
        'password',
        'avatar',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
        ];
    }
}
```

---

# 4. Create Customer Role

```bash
php artisan make:seeder RoleSeeder
```

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Spatie\Permission\Models\Role;

class RoleSeeder extends Seeder
{
    public function run(): void
    {
        Role::firstOrCreate([
            'name' => 'customer',
        ]);
    }
}
```

Run Seeder

```bash
php artisan db:seed --class=RoleSeeder
```

---

# 5. Routes

```php
use App\Http\Controllers\AuthController;

Route::middleware('guest')->group(function () {

    /*
    |--------------------------------------------------------------------------
    | Registration Page
    |--------------------------------------------------------------------------
    */

    Route::get('/register', [AuthController::class, 'showRegister'])
        ->name('register.form');

    /*
    |--------------------------------------------------------------------------
    | Register User
    |--------------------------------------------------------------------------
    */

    Route::post('/register', [AuthController::class, 'register'])
        ->middleware('throttle:5,1')
        ->name('register');

});
```

---

# 6. Controller

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Auth;
use Illuminate\Validation\Rules\Password as PasswordRule;
use Laravolt\Avatar\Facade as Avatar;

class AuthController extends Controller
{
    /**
     * Show Registration Page
     */
    public function showRegister()
    {
        return view('auth.register');
    }

    /**
     * Register User
     */
    public function register(Request $request)
    {
        $validated = $request->validate([

            'name' => [
                'required',
                'string',
                'max:255',
            ],

            'username' => [
                'required',
                'string',
                'max:50',
                'alpha_dash',
                'unique:users,username',
            ],

            'phone' => [
                'required',
                'digits_between:10,15',
                'unique:users,phone',
            ],

            'email' => [
                'required',
                'email',
                'unique:users,email',
            ],

            'password' => [
                'required',
                'confirmed',
                PasswordRule::defaults(),
            ],

        ]);

        DB::transaction(function () use ($validated, &$user) {

            $avatarName = time().'.png';

            Avatar::create($validated['name'])
                ->save(storage_path(
                    'app/public/avatars/'.$avatarName
                ));

            $user = User::create([

                'name' => $validated['name'],

                'username' => $validated['username'],

                'phone' => $validated['phone'],

                'email' => $validated['email'],

                'password' => $validated['password'],

                'avatar' => 'avatars/'.$avatarName,

            ]);

            $user->assignRole('customer');

        });

        Auth::login($user);

        $request->session()->regenerate();

        return redirect()->route('dashboard');
    }
}
```

---

# 7. Registration Blade

```blade
<form
    action="{{ route('register') }}"
    method="POST">

    @csrf

    <div>

        <label>Full Name</label>

        <input
            type="text"
            name="name"
            value="{{ old('name') }}"
            required>

        @error('name')
            <div>{{ $message }}</div>
        @enderror

    </div>

    <br>

    <div>

        <label>Username</label>

        <input
            type="text"
            name="username"
            value="{{ old('username') }}"
            required>

        @error('username')
            <div>{{ $message }}</div>
        @enderror

    </div>

    <br>

    <div>

        <label>Phone Number</label>

        <input
            type="tel"
            name="phone"
            value="{{ old('phone') }}"
            required>

        @error('phone')
            <div>{{ $message }}</div>
        @enderror

    </div>

    <br>

    <div>

        <label>Email</label>

        <input
            type="email"
            name="email"
            value="{{ old('email') }}"
            autocomplete="email"
            required>

        @error('email')
            <div>{{ $message }}</div>
        @enderror

    </div>

    <br>

    <div>

        <label>Password</label>

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

        Register

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
            "Password must contain at least 8 characters, one uppercase letter, one lowercase letter, one number and one special character."
        );

    }

});

</script>
```

---

# 8. Storage Link

```bash
php artisan storage:link
```

---

# 9. Dashboard Route

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

# 10. Flow

```text
Open Registration Page
        │
        ▼
Enter Full Name
        │
        ▼
Enter Username
        │
        ▼
Enter Phone Number
        │
        ▼
Enter Email
        │
        ▼
Enter Password
        │
        ▼
Confirm Password
        │
        ▼
Frontend Validation
        │
        ▼
Backend Validation
        │
        ▼
Generate Avatar
        │
        ▼
Create User
        │
        ▼
Assign Customer Role
        │
        ▼
Login User
        │
        ▼
Regenerate Session
        │
        ▼
Redirect To Dashboard
```
