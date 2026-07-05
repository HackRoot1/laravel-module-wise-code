# First Get Credentials

## 1. Get OAuth Credentials

### 1.1 Google OAuth credentials

1. Go to **Google Cloud Console**: https://console.cloud.google.com.
2. Create or select a project.  
3. In the left menu, go to **APIs & Services → Credentials**.
4. Click **Create credentials → OAuth client ID**.  
5. If prompted, configure the OAuth consent screen (App name, support email, scopes, etc.).
6. Choose **Web application** as Application type.  
7. Set **Authorized redirect URIs**:

   For local dev (Laravel default host):

   ```text
   http://127.0.0.1:8000/auth/google/callback
   ```

   For production, e.g.:

   ```text
   https://your-domain.com/auth/google/callback
   ```

8. Save and note the **Client ID** and **Client secret**.

### 1.2 Facebook App ID & Secret

1. Go to **Meta for Developers**: https://developers.facebook.com.
2. Click **My Apps → Create App**.  
3. Select **Consumer** app type and continue.
4. Fill in app name, contact email, and create the app.  
5. In **Settings → Basic**, take note of **App ID** (Client ID) and **App Secret**.
6. Add Facebook Login product: in left sidebar, click **Add Product → Facebook Login → Set Up**.
7. Under **Facebook Login → Settings**, set **Valid OAuth Redirect URIs**:

   ```text
   http://127.0.0.1:8000/auth/facebook/callback
   https://your-domain.com/auth/facebook/callback
   ```

8. Put the app in Live mode when ready (after setting privacy and URLs).

### 1.3 GitHub OAuth App credentials

1. Go to GitHub → top right avatar → **Settings**.
2. In the left sidebar, click **Developer settings → OAuth Apps → New OAuth App**.
3. Fill in:
   - **Application name**
   - **Homepage URL**: `http://127.0.0.1:8000` (dev) or your domain
   - **Authorization callback URL**:

     ```text
     http://127.0.0.1:8000/auth/github/callback
     ```

4. Register the application.  
5. Copy **Client ID**, then click **Generate a new client secret** and copy the secret.

***



# Laravel Social Login (Google, Facebook, GitHub)

## 1. Install Socialite

```bash
composer require laravel/socialite
```

---

## 2. Create Social Accounts Table

```bash
php artisan make:model SocialAccount -m
```

Migration

```php
public function up(): void
{
    Schema::create('social_accounts', function (Blueprint $table) {
        $table->id();

        $table->foreignId('user_id')
            ->constrained()
            ->cascadeOnDelete();

        $table->string('provider');

        $table->string('provider_id');

        $table->string('avatar')->nullable();

        $table->timestamps();

        $table->unique([
            'provider',
            'provider_id'
        ]);
    });
}

public function down(): void
{
    Schema::dropIfExists('social_accounts');
}
```

Run Migration

```bash
php artisan migrate
```

---

## 3. Configure .env

```env
APP_URL=http://127.0.0.1:8000

GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=${APP_URL}/auth/google/callback

FACEBOOK_CLIENT_ID=
FACEBOOK_CLIENT_SECRET=
FACEBOOK_REDIRECT_URI=${APP_URL}/auth/facebook/callback

GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GITHUB_REDIRECT_URI=${APP_URL}/auth/github/callback
```

---

## 4. Configure services.php

```php
'google' => [
    'client_id' => env('GOOGLE_CLIENT_ID'),
    'client_secret' => env('GOOGLE_CLIENT_SECRET'),
    'redirect' => env('GOOGLE_REDIRECT_URI'),
],

'facebook' => [
    'client_id' => env('FACEBOOK_CLIENT_ID'),
    'client_secret' => env('FACEBOOK_CLIENT_SECRET'),
    'redirect' => env('FACEBOOK_REDIRECT_URI'),
],

'github' => [
    'client_id' => env('GITHUB_CLIENT_ID'),
    'client_secret' => env('GITHUB_CLIENT_SECRET'),
    'redirect' => env('GITHUB_REDIRECT_URI'),
],
```

---

## 5. Routes

```php
use App\Http\Controllers\Auth\SocialController;

Route::middleware('guest')->group(function () {

    Route::get('/auth/{provider}/redirect', [SocialController::class, 'redirectToProvider'])
        ->whereIn('provider', ['google', 'facebook', 'github'])
        ->name('auth.redirect');

    Route::get('/auth/{provider}/callback', [SocialController::class, 'handleProviderCallback'])
        ->whereIn('provider', ['google', 'facebook', 'github'])
        ->middleware('throttle:10,1')
        ->name('auth.callback');

});
```

---

## 6. User Model

```php
public function socialAccounts()
{
    return $this->hasMany(SocialAccount::class);
}
```

---

## 7. SocialAccount Model

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class SocialAccount extends Model
{
    protected $fillable = [
        'user_id',
        'provider',
        'provider_id',
        'avatar',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

---

## 8. Controller

```bash
php artisan make:controller Auth/SocialController
```

```php
<?php

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\SocialAccount;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Laravel\Socialite\Facades\Socialite;

class SocialController extends Controller
{
    /**
     * Redirect To Provider
     */
    public function redirectToProvider(string $provider)
    {
        validator(
            ['provider' => $provider],
            [
                'provider' => 'required|in:google,facebook,github',
            ]
        )->validate();

        return Socialite::driver($provider)->redirect();
    }

    /**
     * Handle Callback
     */
    public function handleProviderCallback(Request $request, string $provider)
    {
        validator(
            ['provider' => $provider],
            [
                'provider' => 'required|in:google,facebook,github',
            ]
        )->validate();

        try {

            $socialUser = Socialite::driver($provider)->user();

        } catch (\Throwable $e) {

            Log::error($e);

            return redirect()
                ->route('login.form')
                ->withErrors([
                    'provider' => 'Unable to login using '.ucfirst($provider).'.',
                ]);
        }

        if (! $socialUser->getEmail()) {

            return redirect()
                ->route('login.form')
                ->withErrors([
                    'provider' => 'Email address is required.',
                ]);
        }

        DB::transaction(function () use (
            $socialUser,
            $provider,
            $request
        ) {

            $account = SocialAccount::where(
                'provider',
                $provider
            )->where(
                'provider_id',
                $socialUser->getId()
            )->first();

            if ($account) {

                $user = $account->user;

            } else {

                $user = User::firstOrCreate(
                    [
                        'email' => $socialUser->getEmail(),
                    ],
                    [
                        'name' => $socialUser->getName()
                            ?? $socialUser->getNickname(),

                        'email_verified_at' => now(),
                    ]
                );

                SocialAccount::create([
                    'user_id' => $user->id,
                    'provider' => $provider,
                    'provider_id' => $socialUser->getId(),
                    'avatar' => $socialUser->getAvatar(),
                ]);
            }

            Auth::login($user);

            $request->session()->regenerate();
        });

        return redirect()->intended(
            route('dashboard')
        );
    }
}
```

---

## 9. Login Blade

```blade
<hr>

<div>

    <a href="{{ route('auth.redirect', ['provider' => 'google']) }}">
        Login with Google
    </a>

    <br><br>

    <a href="{{ route('auth.redirect', ['provider' => 'facebook']) }}">
        Login with Facebook
    </a>

    <br><br>

    <a href="{{ route('auth.redirect', ['provider' => 'github']) }}">
        Login with GitHub
    </a>

</div>
```

---

## 10. Dashboard Route

```php
Route::middleware('auth')->group(function () {

    Route::get('/dashboard', function () {

        return view('dashboard');

    })->name('dashboard');

});
```
