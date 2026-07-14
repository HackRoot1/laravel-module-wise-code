# Roles CRUD (Spatie `laravel-permission`)

Documents the existing `RoleController` (resource CRUD + permission assignment) over the `roles` table via `spatie/laravel-permission`, plus matching basic blade views.

**Known limitations of the controller below (not fixed here, just flagged):**
- No authorization/middleware gate — any authenticated user can hit these routes.
- `destroy()` doesn't check if users are still assigned this role before deleting.
- `update()` allows renaming any role, including ones referenced by name elsewhere in your app (e.g. `hasRole('super-admin')` checks) — renaming breaks those silently.
- `index()` uses `Role::all()` — no pagination.

If you want these fixed, say so explicitly — this README documents the code as provided.

---

## Prerequisite

Same package as permissions — if already set up for that, no further install needed.

```bash
composer require spatie/laravel-permission
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

---

## Routes

**File:** `routes/web.php`

```php
use App\Http\Controllers\RoleController;

Route::resource('roles', RoleController::class);

// Not a standard resource action — assign permissions to a role
Route::put('roles/{id}/assign-permissions', [RoleController::class, 'assignPermissions'])
    ->name('roles.assignPermissions');
```

Generates (resource part):

| Method | URI                    | Action  | Route Name    |
|--------|------------------------|---------|----------------|
| GET    | /roles                  | index   | roles.index    |
| GET    | /roles/create           | create  | roles.create   |
| POST   | /roles                  | store   | roles.store    |
| GET    | /roles/{id}             | show    | roles.show     |
| GET    | /roles/{id}/edit        | edit    | roles.edit     |
| PUT    | /roles/{id}             | update  | roles.update   |
| DELETE | /roles/{id}             | destroy | roles.destroy  |

Plus the manually declared:

| Method | URI                              | Action            | Route Name              |
|--------|-----------------------------------|-------------------|---------------------------|
| PUT    | /roles/{id}/assign-permissions      | assignPermissions | roles.assignPermissions   |

---

## Controller

**File:** `app/Http/Controllers/RoleController.php`

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Spatie\Permission\Models\Permission;
use Spatie\Permission\Models\Role;

class RoleController extends Controller
{
    public function index()
    {
        $roles = Role::all();
        return view('roles.index', compact('roles'));
    }

    public function create()
    {
        return view('roles.create');
    }

    public function store(Request $request)
    {
        $request->validate([
            'name' => 'required|unique:roles,name',
        ]);

        Role::create(['name' => $request->name]);

        return redirect()->route('roles.index')
            ->with('success', 'Role created successfully.');
    }

    public function show(string $id)
    {
        $role = Role::findOrFail($id);
        $permissions = Permission::orderBy('name')->get();
        $assignedPermissionIds = $role->permissions()->pluck('id')->all();

        return view('roles.view', compact('role', 'permissions', 'assignedPermissionIds'));
    }

    public function assignPermissions(Request $request, string $id)
    {
        $role = Role::findOrFail($id);

        $data = $request->validate([
            'permissions' => ['nullable', 'array'],
            'permissions.*' => ['integer', 'exists:permissions,id'],
        ]);

        $permissionIds = $data['permissions'] ?? [];
        $permissions = Permission::where('guard_name', $role->guard_name)
            ->whereIn('id', $permissionIds)
            ->get();

        $role->syncPermissions($permissions);

        return redirect()->route('roles.show', $role->id)
            ->with('success', 'Permissions assigned successfully.');
    }

    public function edit(string $id)
    {
        $role = Role::findOrFail($id);
        return view('roles.edit', compact('role'));
    }

    public function update(Request $request, string $id)
    {
        $request->validate([
            'name' => 'required|unique:roles,name,' . $id,
        ]);

        $role = Role::findOrFail($id);
        $role->name = $request->name;
        $role->save();

        return redirect()->route('roles.index')
            ->with('success', 'Role updated successfully.');
    }

    public function destroy(string $id)
    {
        $role = Role::findOrFail($id);
        $role->delete();

        return redirect()->route('roles.index')
            ->with('success', 'Role deleted successfully.');
    }
}
```

---

## Views (no styling, structure only)

### `resources/views/roles/index.blade.php`

```blade
<h1>Roles</h1>

@if (session('success'))
    <p>{{ session('success') }}</p>
@endif

<a href="{{ route('roles.create') }}">Create Role</a>

<table>
    <thead>
        <tr>
            <th>#</th>
            <th>Name</th>
            <th>Guard</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        @foreach ($roles as $role)
            <tr>
                <td>{{ $role->id }}</td>
                <td>{{ $role->name }}</td>
                <td>{{ $role->guard_name }}</td>
                <td>
                    <a href="{{ route('roles.show', $role->id) }}">View / Assign Permissions</a>
                    <a href="{{ route('roles.edit', $role->id) }}">Edit</a>
                    <form action="{{ route('roles.destroy', $role->id) }}" method="POST" style="display:inline">
                        @csrf
                        @method('DELETE')
                        <button type="submit" onclick="return confirm('Delete this role?')">Delete</button>
                    </form>
                </td>
            </tr>
        @endforeach
    </tbody>
</table>
```

### `resources/views/roles/create.blade.php`

```blade
<h1>Create Role</h1>

<form action="{{ route('roles.store') }}" method="POST">
    @csrf

    <label>Name</label>
    <input type="text" name="name" value="{{ old('name') }}" required>

    @error('name')
        <div style="color:red">{{ $message }}</div>
    @enderror

    <button type="submit">Save</button>
</form>

<a href="{{ route('roles.index') }}">Back</a>
```

### `resources/views/roles/edit.blade.php`

```blade
<h1>Edit Role</h1>

<form action="{{ route('roles.update', $role->id) }}" method="POST">
    @csrf
    @method('PUT')

    <label>Name</label>
    <input type="text" name="name" value="{{ old('name', $role->name) }}" required>

    @error('name')
        <div style="color:red">{{ $message }}</div>
    @enderror

    <button type="submit">Update</button>
</form>

<a href="{{ route('roles.index') }}">Back</a>
```

### `resources/views/roles/view.blade.php`

Shows role details and a checkbox form to assign permissions.

```blade
<h1>Role Details</h1>

<p><strong>ID:</strong> {{ $role->id }}</p>
<p><strong>Name:</strong> {{ $role->name }}</p>
<p><strong>Guard:</strong> {{ $role->guard_name }}</p>

<h2>Assign Permissions</h2>

<form action="{{ route('roles.assignPermissions', $role->id) }}" method="POST">
    @csrf
    @method('PUT')

    @foreach ($permissions as $permission)
        <label>
            <input
                type="checkbox"
                name="permissions[]"
                value="{{ $permission->id }}"
                @checked(in_array($permission->id, $assignedPermissionIds))
            >
            {{ $permission->name }}
        </label>
        <br>
    @endforeach

    <button type="submit">Save Permissions</button>
</form>

<a href="{{ route('roles.index') }}">Back</a>
```