# FileHelper — Setup & Usage Guide

Config-driven file upload/update/delete helper. Drivers: `public`, `storage` (public disk), `s3`. `firebase` is a stub that throws — not implemented.

## Step 1: Create the config file

`config/fileupload.php`

```php
<?php

return [
    /*
    | Supported: "public", "storage", "s3", "firebase" (throws — not implemented).
    | Changing this only affects NEW uploads. Existing files stored under a
    | different driver are NOT migrated automatically — see Limitations below.
    */
    'driver' => env('FILE_UPLOAD_DRIVER', 'public'),
];
```

## Step 2: Add the driver to `.env`

```env
FILE_UPLOAD_DRIVER=public
# public | storage | s3
```

## Step 3: Create the migration

```bash
php artisan make:migration create_file_histories_table
```

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('file_histories', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('user_id')->nullable();
            $table->string('model_name')->nullable();
            $table->unsignedBigInteger('model_id')->nullable();
            $table->string('column_name')->nullable();
            $table->text('old_path');
            $table->string('disk')->nullable(); // records which disk the old file lived on
            $table->enum('action', ['deleted', 'updated']);
            $table->timestamps();
            $table->index(['model_name', 'model_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('file_histories');
    }
};
```

Run it:

```bash
php artisan migrate
```

## Step 4: Create the FileHistory model

`app/Models/FileHistory.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class FileHistory extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id', 'model_name', 'model_id',
        'column_name', 'old_path', 'disk', 'action',
    ];
}
```

## Step 5: Create the FileHelper class

`app/Helpers/FileHelper.php`

```php
<?php

namespace App\Helpers;

use App\Models\FileHistory;
use Exception;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\File;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class FileHelper
{
    public static function fileUpload(UploadedFile $file, string $folder): array
    {
        $driver = config('fileupload.driver');
        $filename = Str::orderedUuid() . '.' . $file->extension();

        switch ($driver) {
            case 'public':
                $directory = public_path($folder);
                if (!File::exists($directory)) {
                    File::makeDirectory($directory, 0755, true, true);
                }
                $file->move($directory, $filename);
                $path = $folder . '/' . $filename;

                return [
                    'path' => $path,
                    'disk' => self::driverToDiskName('public'),
                    'url' => asset($path),
                    'filename' => $filename,
                    'size' => $file->getSize(),
                    'mime' => $file->getMimeType(),
                ];

            case 'storage':
                $path = $file->storeAs($folder, $filename, 'public');

                return [
                    'path' => $path,
                    'disk' => self::driverToDiskName('storage'),
                    'url' => Storage::disk('public')->url($path),
                    'filename' => $filename,
                    'size' => $file->getSize(),
                    'mime' => $file->getMimeType(),
                ];

            case 's3':
                $path = $file->storeAs($folder, $filename, 's3');

                return [
                    'path' => $path,
                    'disk' => self::driverToDiskName('s3'),
                    'url' => Storage::disk('s3')->url($path),
                    'filename' => $filename,
                    'size' => $file->getSize(),
                    'mime' => $file->getMimeType(),
                ];

            case 'firebase':
                throw new Exception('Firebase upload driver is not implemented.');

            default:
                throw new Exception("Invalid upload driver: [{$driver}].");
        }
    }

    public static function fileUpdate(
        UploadedFile $file,
        string $folder,
        $model,
        string $column
    ): array {
        $oldPath = $model->{$column};

        // Assumes old file lives on the CURRENTLY configured driver. Wrong if
        // FILE_UPLOAD_DRIVER changed since the old file was uploaded — see
        // "Known limitations" below.
        $oldDisk = self::driverToDiskName(config('fileupload.driver'));

        // Upload first. If this throws, old file and DB row are untouched.
        $new = self::fileUpload($file, $folder);

        DB::transaction(function () use ($model, $column, $new, $oldPath, $oldDisk) {
            if ($oldPath) {
                FileHistory::create([
                    'user_id' => auth()->id(),
                    'model_name' => get_class($model),
                    'model_id' => $model->id,
                    'column_name' => $column,
                    'old_path' => $oldPath,
                    'disk' => $oldDisk,
                    'action' => 'updated',
                ]);
            }
            $model->{$column} = $new['path'];
            $model->save();
        });

        // Delete old file last — only after DB is committed to the new path.
        if ($oldPath) {
            try {
                self::deleteFile($oldPath, $oldDisk);
            } catch (Exception $e) {
                report($e);
            }
        }

        return $new;
    }

    public static function deleteFile(string $path, ?string $disk = null): void
    {
        $target = $disk ?? self::driverToDiskName(config('fileupload.driver'));

        switch ($target) {
            case 'public_path':
                $fullPath = public_path($path);
                if (File::exists($fullPath)) {
                    File::delete($fullPath);
                }
                break;

            case 'storage_public':
                Storage::disk('public')->delete($path);
                break;

            case 's3':
                Storage::disk('s3')->delete($path);
                break;

            case 'firebase':
                throw new Exception('Firebase delete driver is not implemented.');

            default:
                throw new Exception("Cannot delete file: unknown disk [{$target}].");
        }
    }

    private static function driverToDiskName(string $driver): string
    {
        return match ($driver) {
            'public' => 'public_path',
            'storage' => 'storage_public',
            's3' => 's3',
            'firebase' => 'firebase',
            default => $driver,
        };
    }
}
```

## Step 6: Create the global helper functions

`app/helpers.php`

```php
<?php

use App\Helpers\FileHelper;

if (!function_exists('fileUpload')) {
    function fileUpload($file, string $folder): array
    {
        return FileHelper::fileUpload($file, $folder);
    }
}

if (!function_exists('fileUpdate')) {
    function fileUpdate($file, string $folder, $model, string $column): array
    {
        return FileHelper::fileUpdate($file, $folder, $model, $column);
    }
}
```

## Step 7: Autoload the helpers file

`composer.json`:

```json
"autoload": {
    "files": ["app/helpers.php"]
}
```

```bash
composer dump-autoload
```

## Step 8: Use it

**Upload:**

```php
$result = fileUpload($request->file('profile'), 'uploads/profile');
$user->profile = $result['path'];
$user->save();
```

**Update:**

```php
$result = fileUpdate($request->file('profile'), 'uploads/profile', $user, 'profile');
```

Return shape (always, from both calls):

```php
[
    'path' => 'uploads/profile/0199f2...-a1b2.jpg',
    'disk' => 'storage_public',
    'url' => 'https://.../storage/uploads/profile/....jpg',
    'filename' => '0199f2...-a1b2.jpg',
    'size' => 48213,
    'mime' => 'image/jpeg',
]
```

**Manual delete:**

```php
FileHelper::deleteFile($path, $disk); // $disk optional
```

## What this fixes vs. the original draft

- Original `fileUpdate()` deleted the old file *before* confirming the new upload succeeded — a failed upload meant permanent data loss. Fixed: upload happens first, delete happens last.
- Original firebase delete branch silently did nothing and returned as if it succeeded. Fixed: throws instead of lying about success.
- Original filenames used `uniqid()`, which collides under concurrent uploads. Fixed: `Str::orderedUuid()`.
- Original returned a bare path string, inconsistent with its own stated goal of API-usable output. Fixed: returns a metadata DTO.
- Original had a driver/disk naming collision (`'public'` meant two different things in different branches). Fixed: namespaced disk identifiers via `driverToDiskName()`.

## Known limitations — read before treating this as done

**Switching `FILE_UPLOAD_DRIVER` does not migrate existing files, and `fileUpdate()` cannot reliably delete an old file if the driver changed since it was uploaded.** The code guesses the old file's disk from the *current* config value. If you never change drivers, this is fine. If you do, deletes on old files can silently miss or throw. Real fix requires storing the disk per-file on the model itself (not just in history) — not implemented here, because it's a schema decision you should make deliberately, not one to bury in a helper class.

**No file validation inside the helper.** Mime/size/extension checks are the caller's job via Form Requests. Don't assume this class protects you from bad uploads.

**No async delete.** For S3 at scale, wrap `deleteFile()` in a queued job yourself if delete latency matters.
