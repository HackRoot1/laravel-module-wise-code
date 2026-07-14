# Permissions CRUD (Spatie `laravel-permission`)

Documents the existing `PermissionController` (standard resource CRUD over the `permissions` table via `spatie/laravel-permission`) plus matching basic blade views.

**Known limitations of the controller below (not fixed here, just flagged):**
- No authorization/middleware gate — any authenticated user can hit these routes.
- `destroy()` doesn't check if the permission is still attached to roles/users before deleting.
- `index()` uses `Permission::all()` — no pagination, won't scale.
- Uses manual `findOrFail($id)` instead of Laravel 12 implicit route-model binding.

If you want these fixed, say so explicitly — this README documents the code as provided.

---

## Prerequisite

```bash
composer require spatie/laravel-permission
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

---

## Routes

**File:** `routes/web.php`

```php
use App\Http\Controllers\PermissionController;

Route::resource('permissions', PermissionController::class);
```

Generates:

| Method | URI                        | Action  | Route Name          |
|--------|----------------------------|---------|----------------------|
| GET    | /permissions                | index   | permissions.index    |
| GET    | /permissions/create          | create  | permissions.create   |
| POST   | /permissions                | store   | permissions.store    |
| GET    | /permissions/{id}            | show    | permissions.show     |
| GET    | /permissions/{id}/edit       | edit    | permissions.edit     |
| PUT    | /permissions/{id}            | update  | permissions.update   |
| DELETE | /permissions/{id}            | destroy | permissions.destroy  |

---

## Controller

**File:** `app/Http/Controllers/PermissionController.php`

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Spatie\Permission\Models\Permission;

class PermissionController extends Controller
{
    public function index()
    {
        $permissions = Permission::all();
        return view('permissions.index', compact('permissions'));
    }

    public function create()
    {
        return view('permissions.create');
    }

    public function store(Request $request)
    {
        $request->validate([
            'name' => 'required|unique:permissions,name',
        ]);

        Permission::create(['name' => $request->name]);

        return redirect()->route('permissions.index')
            ->with('success', 'Permission created successfully.');
    }

    public function show(string $id)
    {
        $permission = Permission::findOrFail($id);
        return view('permissions.view', compact('permission'));
    }

    public function edit(string $id)
    {
        $permission = Permission::findOrFail($id);
        return view('permissions.edit', compact('permission'));
    }

    public function update(Request $request, string $id)
    {
        $request->validate([
            'name' => 'required|unique:permissions,name,' . $id,
        ]);

        $permission = Permission::findOrFail($id);
        $permission->name = $request->name;
        $permission->save();

        return redirect()->route('permissions.index')
            ->with('success', 'Permission updated successfully.');
    }

    public function destroy(string $id)
    {
        $permission = Permission::findOrFail($id);
        $permission->delete();

        return redirect()->route('permissions.index')
            ->with('success', 'Permission deleted successfully.');
    }
}
```

---

## Views (no styling, structure only)

### `resources/views/permissions/index.blade.php`

```blade
<h1>Permissions</h1>

@if (session('success'))
    <p>{{ session('success') }}</p>
@endif

<a href="{{ route('permissions.create') }}">Create Permission</a>

<table>
    <thead>
        <tr>
            <th>#</th>
            <th>Name</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody>
        @foreach ($permissions as $permission)
            <tr>
                <td>{{ $permission->id }}</td>
                <td>{{ $permission->name }}</td>
                <td>
                    <a href="{{ route('permissions.show', $permission->id) }}">View</a>
                    <a href="{{ route('permissions.edit', $permission->id) }}">Edit</a>
                    <form action="{{ route('permissions.destroy', $permission->id) }}" method="POST" style="display:inline">
                        @csrf
                        @method('DELETE')
                        <button type="submit" onclick="return confirm('Delete this permission?')">Delete</button>
                    </form>
                </td>
            </tr>
        @endforeach
    </tbody>
</table>
```

### `resources/views/permissions/create.blade.php`

```blade
<h1>Create Permission</h1>

<form action="{{ route('permissions.store') }}" method="POST">
    @csrf

    <label>Name</label>
    <input type="text" name="name" value="{{ old('name') }}" required>

    @error('name')
        <div style="color:red">{{ $message }}</div>
    @enderror

    <button type="submit">Save</button>
</form>

<a href="{{ route('permissions.index') }}">Back</a>
```

### `resources/views/permissions/edit.blade.php`

```blade
<h1>Edit Permission</h1>

<form action="{{ route('permissions.update', $permission->id) }}" method="POST">
    @csrf
    @method('PUT')

    <label>Name</label>
    <input type="text" name="name" value="{{ old('name', $permission->name) }}" required>

    @error('name')
        <div style="color:red">{{ $message }}</div>
    @enderror

    <button type="submit">Update</button>
</form>

<a href="{{ route('permissions.index') }}">Back</a>
```

### `resources/views/permissions/view.blade.php`

```blade
<h1>Permission Details</h1>

<p><strong>ID:</strong> {{ $permission->id }}</p>
<p><strong>Name:</strong> {{ $permission->name }}</p>
<p><strong>Guard:</strong> {{ $permission->guard_name }}</p>

<a href="{{ route('permissions.index') }}">Back</a>
```