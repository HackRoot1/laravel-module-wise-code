# Users CRUD (with Roles via Spatie `laravel-permission`)

Documents the existing `UserController` plus the model/migration changes it depends on, and matching basic blade views.

**Known issues in the controller below (not fixed here, just flagged):**
- `password` is assigned raw in both `store()` and `update()` — relies entirely on the model cast added below to hash it. If that cast is missing or conflicts with an existing mutator, passwords are stored in plaintext.
- `account_status` has a `deleted` enum option, but `destroy()` hard-deletes the row. Two competing deletion concepts — pick one.
- No check stopping an admin from deleting their own account.
- No authorization/middleware gate on any of these routes.

If you want these fixed, say so explicitly — this README documents the code as provided, plus the minimum model/migration support it needs to function correctly.

---

## Step 1: Migration — add missing columns

`username`, `phone`, `account_status` aren't part of Laravel's default `users` table.

```bash
php artisan make:migration add_profile_fields_to_users_table --table=users
```

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('username', 50)->nullable()->unique()->after('name');
    $table->string('phone', 15)->nullable()->unique()->after('email');
    $table->enum('account_status', ['active', 'inactive', 'suspended', 'delete_requested', 'deleted'])
        ->default('active')
        ->after('phone');
});
```

```bash
php artisan migrate
```

---

## Step 2: Model changes

**File:** `app/Models/User.php`

```php
protected $fillable = [
    'name',
    'email',
    'username',
    'phone',
    'account_status',
    'password',
];

protected $hidden = [
    'password',
    'remember_token',
];

protected $casts = [
    'email_verified_at' => 'datetime',
    'password' => 'hashed', // Required — controller assigns raw password, this cast hashes on save
];
```

If your model already has a `setPasswordAttribute()` mutator that hashes manually, remove it before adding this cast — having both double-hashes and breaks login.

---

## Step 3: Routes

**File:** `routes/web.php`

```php
use App\Http\Controllers\UserController;

Route::resource('users', UserController::class);
```

| Method | URI               | Action  | Route Name    |
|--------|--------------------|---------|----------------|
| GET    | /users              | index   | users.index    |
| GET    | /users/create       | create  | users.create   |
| POST   | /users              | store   | users.store    |
| GET    | /users/{id}         | show    | users.show     |
| GET    | /users/{id}/edit    | edit    | users.edit     |
| PUT    | /users/{id}         | update  | users.update   |
| DELETE | /users/{id}         | destroy | users.destroy  |

---

## Controller

**File:** `app/Http/Controllers/UserController.php`

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Password as PasswordRule;
use Spatie\Permission\Models\Role;

class UserController extends Controller
{
    public function index()
    {
        $users = User::with('roles')->latest()->get();

        return view('users.index', compact('users'));
    }

    public function create()
    {
        $roles = Role::where('guard_name', 'web')->orderBy('name')->get();

        return view('users.create', compact('roles'));
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'max:255', Rule::unique('users', 'email')],
            'username' => ['nullable', 'string', 'max:50', 'alpha_dash', Rule::unique('users', 'username')],
            'phone' => ['nullable', 'digits_between:10,15', Rule::unique('users', 'phone')],
            'account_status' => ['required', Rule::in(['active', 'inactive', 'suspended', 'delete_requested', 'deleted'])],
            'password' => ['required', 'confirmed', PasswordRule::defaults()],
            'roles' => ['nullable', 'array'],
            'roles.*' => ['integer', 'exists:roles,id'],
        ]);

        $user = User::create([
            'name' => $data['name'],
            'email' => $data['email'],
            'username' => $data['username'] ?? null,
            'phone' => $data['phone'] ?? null,
            'account_status' => $data['account_status'],
            'password' => $data['password'],
        ]);

        $roles = Role::where('guard_name', 'web')
            ->whereIn('id', $data['roles'] ?? [])
            ->get();

        $user->syncRoles($roles);

        return redirect()->route('users.index')
            ->with('success', 'User created successfully.');
    }

    public function show(string $id)
    {
        $user = User::with('roles')->findOrFail($id);

        return view('users.view', compact('user'));
    }

    public function edit(string $id)
    {
        $user = User::with('roles')->findOrFail($id);
        $roles = Role::where('guard_name', 'web')->orderBy('name')->get();
        $assignedRoleIds = $user->roles->pluck('id')->all();

        return view('users.edit', compact('user', 'roles', 'assignedRoleIds'));
    }

    public function update(Request $request, string $id)
    {
        $user = User::findOrFail($id);

        $data = $request->validate([
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'max:255', Rule::unique('users', 'email')->ignore($user->id)],
            'username' => ['nullable', 'string', 'max:50', 'alpha_dash', Rule::unique('users', 'username')->ignore($user->id)],
            'phone' => ['nullable', 'digits_between:10,15', Rule::unique('users', 'phone')->ignore($user->id)],
            'account_status' => ['required', Rule::in(['active', 'inactive', 'suspended', 'delete_requested', 'deleted'])],
            'roles' => ['nullable', 'array'],
            'roles.*' => ['integer', 'exists:roles,id'],
        ]);

        if ($request->filled('password')) {
            $request->validate([
                'password' => ['required', 'confirmed', PasswordRule::defaults()],
            ]);

            $data['password'] = $request->password;
        }

        $user->fill([
            'name' => $data['name'],
            'email' => $data['email'],
            'username' => $data['username'] ?? null,
            'phone' => $data['phone'] ?? null,
            'account_status' => $data['account_status'],
        ]);

        if (array_key_exists('password', $data)) {
            $user->password = $data['password'];
        }

        $user->save();

        $roles = Role::where('guard_name', 'web')
            ->whereIn('id', $data['roles'] ?? [])
            ->get();

        $user->syncRoles($roles);

        return redirect()->route('users.index')
            ->with('success', 'User updated successfully.');
    }

    public function destroy(string $id)
    {
        $user = User::findOrFail($id);
        $user->delete();

        return redirect()->route('users.index')
            ->with('success', 'User deleted successfully.');
    }
}
```

---

## Views (no styling, structure only)

### `resources/views/users/index.blade.php`

```blade
<h1>Users</h1>

@if (session('success'))
    <p>{{ session('success') }}</p>
@endif

<a href="{{ route('users.create') }}">Create User</a>

<table>
    <thead>
        <tr>
            <th>#</th>
            <th>Name</th>
            <th>Email</th>
            <th>Status</th>
            <th>Roles</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        @foreach ($users as $user)
            <tr>
                <td>{{ $user->id }}</td>
                <td>{{ $user->name }}</td>
                <td>{{ $user->email }}</td>
                <td>{{ $user->account_status }}</td>
                <td>{{ $user->roles->pluck('name')->join(', ') }}</td>
                <td>
                    <a href="{{ route('users.show', $user->id) }}">View</a>
                    <a href="{{ route('users.edit', $user->id) }}">Edit</a>
                    <form action="{{ route('users.destroy', $user->id) }}" method="POST" style="display:inline">
                        @csrf
                        @method('DELETE')
                        <button type="submit" onclick="return confirm('Delete this user?')">Delete</button>
                    </form>
                </td>
            </tr>
        @endforeach
    </tbody>
</table>
```

### `resources/views/users/create.blade.php`

```blade
<h1>Create User</h1>

<form action="{{ route('users.store') }}" method="POST">
    @csrf

    <label>Name</label>
    <input type="text" name="name" value="{{ old('name') }}" required>
    @error('name') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Email</label>
    <input type="email" name="email" value="{{ old('email') }}" required>
    @error('email') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Username</label>
    <input type="text" name="username" value="{{ old('username') }}">
    @error('username') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Phone</label>
    <input type="text" name="phone" value="{{ old('phone') }}">
    @error('phone') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Account Status</label>
    <select name="account_status" required>
        @foreach (['active', 'inactive', 'suspended', 'delete_requested', 'deleted'] as $status)
            <option value="{{ $status }}" @selected(old('account_status') === $status)>{{ $status }}</option>
        @endforeach
    </select>
    @error('account_status') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Password</label>
    <input type="password" name="password" required>
    @error('password') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Confirm Password</label>
    <input type="password" name="password_confirmation" required>

    <br>

    <label>Roles</label>
    @foreach ($roles as $role)
        <label>
            <input type="checkbox" name="roles[]" value="{{ $role->id }}"
                @checked(is_array(old('roles')) && in_array($role->id, old('roles')))>
            {{ $role->name }}
        </label>
    @endforeach

    <br>

    <button type="submit">Save</button>
</form>

<a href="{{ route('users.index') }}">Back</a>
```

### `resources/views/users/edit.blade.php`

```blade
<h1>Edit User</h1>

<form action="{{ route('users.update', $user->id) }}" method="POST">
    @csrf
    @method('PUT')

    <label>Name</label>
    <input type="text" name="name" value="{{ old('name', $user->name) }}" required>
    @error('name') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Email</label>
    <input type="email" name="email" value="{{ old('email', $user->email) }}" required>
    @error('email') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Username</label>
    <input type="text" name="username" value="{{ old('username', $user->username) }}">
    @error('username') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Phone</label>
    <input type="text" name="phone" value="{{ old('phone', $user->phone) }}">
    @error('phone') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Account Status</label>
    <select name="account_status" required>
        @foreach (['active', 'inactive', 'suspended', 'delete_requested', 'deleted'] as $status)
            <option value="{{ $status }}" @selected(old('account_status', $user->account_status) === $status)>{{ $status }}</option>
        @endforeach
    </select>
    @error('account_status') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>New Password (leave blank to keep current)</label>
    <input type="password" name="password">
    @error('password') <div style="color:red">{{ $message }}</div> @enderror

    <br>

    <label>Confirm New Password</label>
    <input type="password" name="password_confirmation">

    <br>

    <label>Roles</label>
    @foreach ($roles as $role)
        <label>
            <input type="checkbox" name="roles[]" value="{{ $role->id }}"
                @checked(in_array($role->id, $assignedRoleIds))>
            {{ $role->name }}
        </label>
    @endforeach

    <br>

    <button type="submit">Update</button>
</form>

<a href="{{ route('users.index') }}">Back</a>
```

### `resources/views/users/view.blade.php`

```blade
<h1>User Details</h1>

<p><strong>ID:</strong> {{ $user->id }}</p>
<p><strong>Name:</strong> {{ $user->name }}</p>
<p><strong>Email:</strong> {{ $user->email }}</p>
<p><strong>Username:</strong> {{ $user->username ?? '-' }}</p>
<p><strong>Phone:</strong> {{ $user->phone ?? '-' }}</p>
<p><strong>Account Status:</strong> {{ $user->account_status }}</p>
<p><strong>Roles:</strong> {{ $user->roles->pluck('name')->join(', ') ?: '-' }}</p>

<a href="{{ route('users.edit', $user->id) }}">Edit</a>
<a href="{{ route('users.index') }}">Back</a>
```