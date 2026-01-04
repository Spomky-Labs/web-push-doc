# Example

This page provides a complete example of implementing Web Push notifications using the standalone library (without Symfony).

## Complete Working Example

### Step 1: Installation

```bash
composer require spomky-labs/web-push-lib
composer require symfony/http-client
composer require symfony/clock
```

### Step 2: Generate VAPID Keys

```bash
# Generate private key
openssl ecparam -genkey -name prime256v1 -out private_key.pem

# Extract public key
openssl ec -in private_key.pem -pubout -outform DER|tail -c 65|base64|tr -d '=' |tr '/+' '_-' > public_key.txt

# Extract private key
openssl ec -in private_key.pem -outform DER|tail -c +8|head -c 32|base64|tr -d '=' |tr '/+' '_-' > private_key.txt
```

### Step 3: Create the Web Push Service

{% code title="src/WebPushServiceFactory.php" %}
```php
<?php

declare(strict_types=1);

namespace App;

use Psr\Clock\ClockInterface;
use Symfony\Component\Clock\NativeClock;
use Symfony\Component\HttpClient\HttpClient;
use WebPush\ExtensionManager;
use WebPush\Payload\AES128GCM;
use WebPush\Payload\AESGCM;
use WebPush\Payload\PayloadExtension;
use WebPush\PreferAsyncExtension;
use WebPush\TopicExtension;
use WebPush\TTLExtension;
use WebPush\UrgencyExtension;
use WebPush\VAPID\LcobucciProvider;
use WebPush\VAPID\VAPIDExtension;
use WebPush\VAPID\WebTokenProvider;
use WebPush\WebPush;

class WebPushServiceFactory
{
    public static function create(
        string $vapidPublicKey,
        string $vapidPrivateKey,
        string $vapidSubject
    ): WebPush {
        $clock = new NativeClock();

        // Create HTTP client
        $httpClient = HttpClient::create();

        // Create Extension Manager
        $extensionManager = self::createExtensionManager($clock, $vapidPublicKey, $vapidPrivateKey, $vapidSubject);

        // Create and return WebPush service
        return WebPush::create($httpClient, $extensionManager);
    }

    private static function createExtensionManager(
        ClockInterface $clock,
        string $vapidPublicKey,
        string $vapidPrivateKey,
        string $vapidSubject
    ): ExtensionManager {
        // Create VAPID extension (choose one provider)
        // Option 1: Using web-token/jwt
        $jwsProvider = WebTokenProvider::create($vapidPublicKey, $vapidPrivateKey);

        // Option 2: Using lcobucci/jwt (uncomment to use)
        // $jwsProvider = LcobucciProvider::create($vapidPublicKey, $vapidPrivateKey);

        $vapidExtension = VAPIDExtension::create($vapidSubject, $jwsProvider, $clock);

        // Create payload extension
        $payloadExtension = PayloadExtension::create()
            ->addContentEncoding(AESGCM::create($clock))
            ->addContentEncoding(AES128GCM::create($clock));

        // Create extension manager with all extensions
        return ExtensionManager::create()
            ->add(TTLExtension::create())
            ->add(UrgencyExtension::create())
            ->add(TopicExtension::create())
            ->add(PreferAsyncExtension::create())
            ->add($payloadExtension)
            ->add($vapidExtension);
    }
}
```
{% endcode %}

### Step 4: Create a Subscription Manager

{% code title="src/SubscriptionManager.php" %}
```php
<?php

declare(strict_types=1);

namespace App;

use WebPush\Subscription;

class SubscriptionManager
{
    private array $subscriptions = [];

    public function __construct(
        private readonly string $storagePath
    ) {
        $this->load();
    }

    public function add(Subscription $subscription, string $userId): void
    {
        if (!isset($this->subscriptions[$userId])) {
            $this->subscriptions[$userId] = [];
        }

        $endpoint = $subscription->getEndpoint();
        $this->subscriptions[$userId][$endpoint] = $subscription;
        $this->save();
    }

    public function remove(string $endpoint, string $userId): void
    {
        if (isset($this->subscriptions[$userId][$endpoint])) {
            unset($this->subscriptions[$userId][$endpoint]);
            $this->save();
        }
    }

    /**
     * @return Subscription[]
     */
    public function getByUser(string $userId): array
    {
        return $this->subscriptions[$userId] ?? [];
    }

    public function getAll(): array
    {
        $all = [];
        foreach ($this->subscriptions as $userSubscriptions) {
            $all = array_merge($all, array_values($userSubscriptions));
        }
        return $all;
    }

    private function load(): void
    {
        if (!file_exists($this->storagePath)) {
            return;
        }

        $data = json_decode(file_get_contents($this->storagePath), true);

        foreach ($data as $userId => $subscriptions) {
            foreach ($subscriptions as $subscriptionData) {
                $subscription = Subscription::createFromString(json_encode($subscriptionData));
                $this->subscriptions[$userId][$subscription->getEndpoint()] = $subscription;
            }
        }
    }

    private function save(): void
    {
        $data = [];
        foreach ($this->subscriptions as $userId => $subscriptions) {
            $data[$userId] = array_map(
                fn(Subscription $s) => $s->jsonSerialize(),
                array_values($subscriptions)
            );
        }

        file_put_contents($this->storagePath, json_encode($data, JSON_PRETTY_PRINT));
    }
}
```
{% endcode %}

### Step 5: Handle Subscription from Browser

{% code title="public/subscribe.php" %}
```php
<?php

declare(strict_types=1);

require_once __DIR__ . '/../vendor/autoload.php';

use App\SubscriptionManager;
use WebPush\Subscription;

// Get the posted data
$data = json_decode(file_get_contents('php://input'), true);

if (!isset($data['endpoint'])) {
    http_response_code(400);
    echo json_encode(['error' => 'Invalid subscription data']);
    exit;
}

// Create subscription from browser data
$subscription = Subscription::createFromString(json_encode($data));

// Get user ID (in real app, get from session)
$userId = $_SESSION['user_id'] ?? 'anonymous';

// Save subscription
$manager = new SubscriptionManager(__DIR__ . '/../data/subscriptions.json');
$manager->add($subscription, $userId);

http_response_code(201);
echo json_encode(['message' => 'Subscription saved']);
```
{% endcode %}

### Step 6: Send Notifications

{% code title="src/NotificationSender.php" %}
```php
<?php

declare(strict_types=1);

namespace App;

use Psr\Log\LoggerInterface;
use Psr\Log\NullLogger;
use WebPush\Message;
use WebPush\Notification;
use WebPush\Subscription;
use WebPush\WebPush;

class NotificationSender
{
    private LoggerInterface $logger;

    public function __construct(
        private readonly WebPush $webPush,
        private readonly SubscriptionManager $subscriptionManager,
        ?LoggerInterface $logger = null
    ) {
        $this->logger = $logger ?? new NullLogger();
    }

    public function sendToUser(string $userId, string $title, string $body, array $data = []): void
    {
        $subscriptions = $this->subscriptionManager->getByUser($userId);

        if (empty($subscriptions)) {
            $this->logger->info("No subscriptions found for user {$userId}");
            return;
        }

        $message = Message::create($title)
            ->withBody($body)
            ->withData($data)
            ->withIcon('/icon-192.png')
            ->withBadge('/badge-72.png');

        $notification = Notification::create()
            ->withPayload($message->toString())
            ->withTTL(3600);

        foreach ($subscriptions as $subscription) {
            $this->sendNotification($notification, $subscription, $userId);
        }
    }

    public function sendToAll(string $title, string $body, array $data = []): void
    {
        $subscriptions = $this->subscriptionManager->getAll();

        if (empty($subscriptions)) {
            $this->logger->info("No subscriptions found");
            return;
        }

        $message = Message::create($title)
            ->withBody($body)
            ->withData($data);

        $notification = Notification::create()
            ->withPayload($message->toString())
            ->withTTL(3600);

        foreach ($subscriptions as $subscription) {
            $this->sendNotification($notification, $subscription);
        }
    }

    private function sendNotification(
        Notification $notification,
        Subscription $subscription,
        ?string $userId = null
    ): void {
        try {
            $report = $this->webPush->send($notification, $subscription);

            if ($report->isSubscriptionExpired()) {
                $this->logger->info('Subscription expired, removing', [
                    'endpoint' => $subscription->getEndpoint()
                ]);

                if ($userId) {
                    $this->subscriptionManager->remove($subscription->getEndpoint(), $userId);
                }
            } elseif (!$report->isSuccess()) {
                $this->logger->error('Failed to send notification', [
                    'endpoint' => $subscription->getEndpoint()
                ]);
            } else {
                $this->logger->info('Notification sent successfully', [
                    'endpoint' => $subscription->getEndpoint()
                ]);
            }
        } catch (\Throwable $e) {
            $this->logger->error('Exception while sending notification', [
                'endpoint' => $subscription->getEndpoint(),
                'error' => $e->getMessage()
            ]);
        }
    }
}
```
{% endcode %}

### Step 7: Usage Example

{% code title="examples/send-notification.php" %}
```php
<?php

declare(strict_types=1);

require_once __DIR__ . '/../vendor/autoload.php';

use App\NotificationSender;
use App\SubscriptionManager;
use App\WebPushServiceFactory;

// Configuration
$vapidPublicKey = file_get_contents(__DIR__ . '/../keys/public_key.txt');
$vapidPrivateKey = file_get_contents(__DIR__ . '/../keys/private_key.txt');
$vapidSubject = 'mailto:admin@example.com';

// Create services
$webPush = WebPushServiceFactory::create(
    trim($vapidPublicKey),
    trim($vapidPrivateKey),
    $vapidSubject
);

$subscriptionManager = new SubscriptionManager(__DIR__ . '/../data/subscriptions.json');
$notificationSender = new NotificationSender($webPush, $subscriptionManager);

// Send notification to specific user
$notificationSender->sendToUser(
    'user123',
    'Hello!',
    'This is a test notification',
    ['url' => 'https://example.com/notifications']
);

// Or send to all users
$notificationSender->sendToAll(
    'Important Update',
    'We have an important announcement for all users!',
    ['url' => 'https://example.com/announcement']
);

echo "Notifications sent!\n";
```
{% endcode %}

### Step 8: Client-side JavaScript

{% code title="public/js/push.js" %}
```javascript
// Configuration
const API_BASE_URL = '/api';

// Request notification permission
async function requestNotificationPermission() {
    const permission = await Notification.requestPermission();
    if (permission !== 'granted') {
        console.error('Notification permission denied');
        return false;
    }
    return true;
}

// Subscribe to push notifications
async function subscribeToPush() {
    if (!('serviceWorker' in navigator) || !('PushManager' in window)) {
        console.error('Push notifications are not supported');
        return;
    }

    try {
        // Register service worker
        const registration = await navigator.serviceWorker.register('/service-worker.js');
        await navigator.serviceWorker.ready;

        // Get VAPID public key from your server
        const response = await fetch(`${API_BASE_URL}/vapid-public-key`);
        const { publicKey } = await response.json();

        // Subscribe
        const subscription = await registration.pushManager.subscribe({
            userVisibleOnly: true,
            applicationServerKey: urlBase64ToUint8Array(publicKey)
        });

        // Get supported content encodings
        const supportedContentEncodings = PushManager.supportedContentEncodings || ['aesgcm'];

        // Prepare subscription data
        const subscriptionData = {
            endpoint: subscription.endpoint,
            keys: {
                p256dh: arrayBufferToBase64(subscription.getKey('p256dh')),
                auth: arrayBufferToBase64(subscription.getKey('auth'))
            },
            supportedContentEncodings: supportedContentEncodings
        };

        // Send to server
        await fetch(`${API_BASE_URL}/subscribe`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(subscriptionData)
        });

        console.log('Successfully subscribed to push notifications');
    } catch (error) {
        console.error('Failed to subscribe:', error);
    }
}

// Unsubscribe from push notifications
async function unsubscribeFromPush() {
    try {
        const registration = await navigator.serviceWorker.ready;
        const subscription = await registration.pushManager.getSubscription();

        if (!subscription) {
            return;
        }

        await fetch(`${API_BASE_URL}/unsubscribe`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ endpoint: subscription.endpoint })
        });

        await subscription.unsubscribe();
        console.log('Successfully unsubscribed from push notifications');
    } catch (error) {
        console.error('Failed to unsubscribe:', error);
    }
}

// Helper functions
function urlBase64ToUint8Array(base64String) {
    const padding = '='.repeat((4 - base64String.length % 4) % 4);
    const base64 = (base64String + padding)
        .replace(/\-/g, '+')
        .replace(/_/g, '/');

    const rawData = window.atob(base64);
    const outputArray = new Uint8Array(rawData.length);

    for (let i = 0; i < rawData.length; ++i) {
        outputArray[i] = rawData.charCodeAt(i);
    }
    return outputArray;
}

function arrayBufferToBase64(buffer) {
    const bytes = new Uint8Array(buffer);
    let binary = '';
    for (let i = 0; i < bytes.byteLength; i++) {
        binary += String.fromCharCode(bytes[i]);
    }
    return window.btoa(binary)
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=+$/, '');
}

// Initialize on page load
document.addEventListener('DOMContentLoaded', async () => {
    const subscribeBtn = document.getElementById('subscribe-btn');
    const unsubscribeBtn = document.getElementById('unsubscribe-btn');

    if (subscribeBtn) {
        subscribeBtn.addEventListener('click', async () => {
            if (await requestNotificationPermission()) {
                await subscribeToPush();
            }
        });
    }

    if (unsubscribeBtn) {
        unsubscribeBtn.addEventListener('click', unsubscribeFromPush);
    }
});
```
{% endcode %}

{% code title="public/service-worker.js" %}
```javascript
self.addEventListener('push', function(event) {
    if (!event.data) {
        return;
    }

    const data = event.data.json();
    const { title, options } = data;

    event.waitUntil(
        self.registration.showNotification(title, options)
    );
});

self.addEventListener('notificationclick', function(event) {
    event.notification.close();

    const urlToOpen = event.notification.data?.url || '/';

    event.waitUntil(
        clients.openWindow(urlToOpen)
    );
});
```
{% endcode %}

### Step 9: Simple HTML Page

{% code title="public/index.html" %}
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Web Push Notifications Example</title>
</head>
<body>
    <h1>Web Push Notifications</h1>

    <div>
        <button id="subscribe-btn">Subscribe to Notifications</button>
        <button id="unsubscribe-btn">Unsubscribe</button>
    </div>

    <script src="/js/push.js"></script>
</body>
</html>
```
{% endcode %}

## Running the Example

1. **Generate VAPID keys** (Step 2)
2. **Create the directory structure**:
   ```bash
   mkdir -p data keys public/js
   ```
3. **Save your keys** in the `keys/` directory
4. **Start a PHP development server**:
   ```bash
   php -S localhost:8000 -t public
   ```
5. **Visit** `http://localhost:8000` and click "Subscribe to Notifications"
6. **Send a test notification**:
   ```bash
   php examples/send-notification.php
   ```

## Notes

- This example uses file-based storage for simplicity. In production, use a database.
- The `WebPush\Subscription` class handles all the complexity of Web Push subscriptions.
- Always handle errors when sending notifications, as subscriptions can expire.
- The service worker must be served from the root of your domain or use the `Service-Worker-Allowed` header.
