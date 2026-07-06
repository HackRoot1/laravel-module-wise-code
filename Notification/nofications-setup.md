# Laravel Notification Module — Implementation Guide

A standalone, enterprise-grade Notification Module for your Laravel 12 boilerplate.
Keeps **PHP Flasher** for post-redirect flash messages, and uses this module for
everything else: in-app bell, email, push, and real-time broadcasts.

---

## 0. Corrections to the original plan

Before implementing, note these fixes to the architecture as originally described:

1. **`NotificationService::send()` should not be a bare static method.**
   Bind it as a singleton in a service provider and resolve it via the container
   (or a thin facade). A static-only class can't be swapped/mocked in tests.
2. **Don't create a custom `notifications` migration.** Laravel ships an artisan
   command for this — use it (`php artisan notifications:table`).
3. **Custom channels (Firebase/OneSignal) must implement a `send($notifiable, $notification)`
   method** — this was implied but not shown in the original plan, and it's the part
   most people get wrong.
4. **Preference checks belong inside each Notification's `via()` method**, not
   scattered across listeners — otherwise you end up checking twice or missing a channel.
5. **Queueing is opt-in per notification class.** Adding `QUEUE_CONNECTION=redis` to
   `.env` does nothing unless the notification class itself implements `ShouldQueue`.

---

## 1. Directory structure

```bash
mkdir -p app/Notifications/{User,Order,Payment,System,Admin}
mkdir -p app/Services
mkdir -p app/Channels
mkdir -p app/Traits
mkdir -p app/Events
mkdir -p app/Listeners
```

Final layout:

```
app/
├── Notifications/
│   ├── User/UserRegisteredNotification.php
│   ├── Order/OrderPlacedNotification.php
│   └── ...
├── Services/NotificationService.php
├── Channels/FirebaseChannel.php
├── Channels/OneSignalChannel.php
├── Traits/SendsNotifications.php
├── Events/UserRegistered.php
└── Listeners/SendWelcomeNotifications.php
```

---

## 2. Database notifications table (built-in, don't hand-roll it)

```bash
php artisan notifications:table
php artisan migrate
```

This creates the standard `notifications` table Laravel's `DatabaseChannel` expects.

---

## 3. Notification preferences table

```bash
php artisan make:migration create_notification_preferences_table
```

```php
// database/migrations/xxxx_xx_xx_create_notification_preferences_table.php
public function up(): void
{
    Schema::create('notification_preferences', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->boolean('email')->default(true);
        $table->boolean('push')->default(true);
        $table->boolean('database')->default(true);
        $table->boolean('broadcast')->default(true);
        $table->boolean('sms')->default(false);
        $table->boolean('marketing')->default(false);
        $table->timestamps();
    });
}
```

```bash
php artisan make:model NotificationPreference
```

```php
// app/Models/NotificationPreference.php
class NotificationPreference extends Model
{
    protected $fillable = ['user_id','email','push','database','broadcast','sms','marketing'];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

Add relation on `User`:

```php
public function notificationPreference(): HasOne
{
    return $this->hasOne(NotificationPreference::class);
}
```

---

## 4. The trait every model uses

```php
// app/Traits/SendsNotifications.php
namespace App\Traits;

use Illuminate\Notifications\Notifiable;

trait SendsNotifications
{
    use Notifiable;

    public function wantsChannel(string $channel): bool
    {
        $pref = $this->notificationPreference;
        return $pref ? (bool) ($pref->{$channel} ?? true) : true;
    }
}
```

Apply it on `User`, `Admin`, `Vendor`, etc.:

```php
class User extends Authenticatable
{
    use SendsNotifications;
}
```

---

## 5. A real notification class (with preference-aware `via()`)

```bash
php artisan make:notification User/UserRegisteredNotification
```

```php
// app/Notifications/User/UserRegisteredNotification.php
namespace App\Notifications\User;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;
use Illuminate\Notifications\Messages\MailMessage;
use App\Channels\FirebaseChannel;

class UserRegisteredNotification extends Notification implements ShouldQueue
{
    use Queueable;

    public function via($notifiable): array
    {
        $channels = ['database'];

        if ($notifiable->wantsChannel('email'))  $channels[] = 'mail';
        if ($notifiable->wantsChannel('push'))    $channels[] = FirebaseChannel::class;
        if ($notifiable->wantsChannel('broadcast')) $channels[] = 'broadcast';

        return $channels;
    }

    public function toMail($notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('Welcome!')
            ->line('Your account has been created successfully.');
    }

    public function toDatabase($notifiable): array
    {
        return ['message' => 'Welcome to the platform!'];
    }

    public function toFirebase($notifiable): array
    {
        return ['title' => 'Welcome!', 'body' => 'Your account is ready.'];
    }

    public function toBroadcast($notifiable): array
    {
        return ['message' => 'Welcome to the platform!'];
    }
}
```

> `ShouldQueue` is what actually makes this queued — the `.env` queue driver alone does nothing.

---

## 6. Custom channel (Firebase example)

```php
// app/Channels/FirebaseChannel.php
namespace App\Channels;

use Illuminate\Notifications\Notification;
use Kreait\Firebase\Messaging\CloudMessage;
use Kreait\Firebase\Contract\Messaging;

class FirebaseChannel
{
    public function __construct(private Messaging $messaging) {}

    // REQUIRED method — this is what the original plan omitted
    public function send($notifiable, Notification $notification): void
    {
        if (!method_exists($notification, 'toFirebase')) {
            return;
        }

        $token = $notifiable->fcm_token ?? null;
        if (!$token) return;

        $data = $notification->toFirebase($notifiable);

        $message = CloudMessage::withTarget('token', $token)
            ->withNotification([
                'title' => $data['title'],
                'body'  => $data['body'],
            ]);

        $this->messaging->send($message);
    }
}
```

Register the binding in `AppServiceProvider::register()`:

```php
$this->app->bind(\App\Channels\FirebaseChannel::class, function ($app) {
    return new \App\Channels\FirebaseChannel(
        app('firebase.messaging')
    );
});
```

Same pattern for `OneSignalChannel` (call OneSignal's REST API in `send()` with Guzzle/HTTP client instead).

---

## 7. NotificationService (bound as a singleton — not static)

```php
// app/Services/NotificationService.php
namespace App\Services;

use Illuminate\Notifications\Notification;

class NotificationService
{
    public function send($notifiable, Notification $notification): void
    {
        $notifiable->notify($notification);
    }

    public function sendNow($notifiable, Notification $notification): void
    {
        // bypasses the queue, for urgent/security notifications
        \Illuminate\Support\Facades\Notification::sendNow($notifiable, $notification);
    }
}
```

Bind it in `app/Providers/AppServiceProvider.php`:

```php
public function register(): void
{
    $this->app->singleton(NotificationService::class, fn () => new NotificationService());
}
```

Usage anywhere in the app:

```php
app(NotificationService::class)->send($user, new UserRegisteredNotification());
```

(Or inject it via constructor/method DI — preferred over `app()` helper in real controllers.)

---

## 8. Event → Listener flow (keeps controllers clean)

```bash
php artisan make:event UserRegistered
php artisan make:listener SendWelcomeNotifications --event=UserRegistered
```

```php
// app/Events/UserRegistered.php
class UserRegistered
{
    public function __construct(public User $user) {}
}
```

```php
// app/Listeners/SendWelcomeNotifications.php
class SendWelcomeNotifications
{
    public function __construct(private NotificationService $notifications) {}

    public function handle(UserRegistered $event): void
    {
        $this->notifications->send(
            $event->user,
            new UserRegisteredNotification()
        );
    }
}
```

Register in `app/Providers/EventServiceProvider.php`:

```php
protected $listen = [
    UserRegistered::class => [SendWelcomeNotifications::class],
];
```

Controller now just does:

```php
event(new UserRegistered($user));
```

No knowledge of email/push/database inside the controller.

---

## 9. Queues

```env
QUEUE_CONNECTION=redis
```

```bash
php artisan queue:work redis --tries=3
```

In production, run this under Supervisor (or Horizon if using Redis).

---

## 10. Broadcasting (real-time bell)

```bash
composer require laravel/reverb
php artisan reverb:install
```

`.env`:

```env
BROADCAST_CONNECTION=reverb
REVERB_APP_ID=...
REVERB_APP_KEY=...
REVERB_APP_SECRET=...
```

Frontend (Laravel Echo):

```js
Echo.private(`App.Models.User.${userId}`)
    .notification((notification) => {
        console.log(notification);
    });
```

---

## 11. Testing

```php
use Illuminate\Support\Facades\Notification;

public function test_user_gets_welcome_notification(): void
{
    Notification::fake();

    $user = User::factory()->create();
    event(new UserRegistered($user));

    Notification::assertSentTo($user, UserRegisteredNotification::class);
}
```

---

## 12. Where PHP Flasher still fits

| Purpose                                    | Technology                     |
|---------------------------------------------|---------------------------------|
| "Profile updated" flash after redirect       | PHP Flasher                    |
| Notification bell inside the app             | Laravel Database Notifications |
| Email alerts                                 | Laravel Notifications          |
| Push notifications                           | Firebase / OneSignal channels  |
| Real-time browser notifications              | Broadcasting (Reverb)          |
| Background sending                           | Queues                         |
| User notification settings                   | notification_preferences table |

Nothing about PHP Flasher changes — this module lives entirely alongside it.

---

## Build order (do it in this sequence)

1. Directory structure (`Section 1`)
2. `notifications:table` migration (`Section 2`)
3. `notification_preferences` table + model (`Section 3`)
4. `SendsNotifications` trait on your models (`Section 4`)
5. First notification class end-to-end (`Section 5`)
6. Custom channel(s) if you need push (`Section 6`)
7. `NotificationService` binding (`Section 7`)
8. Event/Listener wiring per feature (`Section 8`)
9. Queue worker running (`Section 9`)
10. Broadcasting, only once the rest works (`Section 10`)
11. Tests for each notification (`Section 11`)
