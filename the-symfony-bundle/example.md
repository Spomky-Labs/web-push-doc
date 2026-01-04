# Example

This page provides a complete example of implementing Web Push notifications in a Symfony application.

## Complete Working Example

### Step 1: Configuration

First, configure the bundle with VAPID authentication:

{% code title="config/packages/webpush.yaml" %}
```yaml
webpush:
  vapid:
    enabled: true
    subject: 'mailto:admin@example.com'
    web_token:
      enabled: true
      public_key: '%env(WEBPUSH_PUBLIC_KEY)%'
      private_key: '%env(WEBPUSH_PRIVATE_KEY)%'
  payload:
    aes128gcm:
      padding: 'recommended'
    aesgcm:
      padding: 'recommended'
  logger: 'monolog.logger'
```
{% endcode %}

{% code title=".env" %}
```bash
# Generate keys with: openssl ecparam -genkey -name prime256v1 -out private_key.pem
# Extract public key: openssl ec -in private_key.pem -pubout -outform DER|tail -c 65|base64|tr -d '=' |tr '/+' '_-'
# Extract private key: openssl ec -in private_key.pem -outform DER|tail -c +8|head -c 32|base64|tr -d '=' |tr '/+' '_-'
WEBPUSH_PUBLIC_KEY=your-public-key-here
WEBPUSH_PRIVATE_KEY=your-private-key-here
```
{% endcode %}

### Step 2: Create the Subscription Entity

{% code title="src/Entity/PushSubscription.php" %}
```php
<?php

declare(strict_types=1);

namespace App\Entity;

use App\Repository\PushSubscriptionRepository;
use Doctrine\ORM\Mapping as ORM;
use WebPush\Subscription as WebPushSubscription;

#[ORM\Entity(repositoryClass: PushSubscriptionRepository::class)]
#[ORM\Table(name: 'push_subscriptions')]
class PushSubscription extends WebPushSubscription
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\ManyToOne(targetEntity: User::class)]
    #[ORM\JoinColumn(nullable: false)]
    private User $user;

    #[ORM\Column]
    private \DateTimeImmutable $createdAt;

    public function __construct(string $endpoint, User $user)
    {
        parent::__construct($endpoint);
        $this->user = $user;
        $this->createdAt = new \DateTimeImmutable();
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getUser(): User
    {
        return $this->user;
    }

    public function getCreatedAt(): \DateTimeImmutable
    {
        return $this->createdAt;
    }

    public static function createFromString(string $input): self
    {
        throw new \RuntimeException('Use createFromRequest instead');
    }
}
```
{% endcode %}

### Step 3: Create the Repository

{% code title="src/Repository/PushSubscriptionRepository.php" %}
```php
<?php

declare(strict_types=1);

namespace App\Repository;

use App\Entity\PushSubscription;
use App\Entity\User;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

class PushSubscriptionRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, PushSubscription::class);
    }

    public function save(PushSubscription $subscription): void
    {
        $this->getEntityManager()->persist($subscription);
        $this->getEntityManager()->flush();
    }

    public function remove(PushSubscription $subscription): void
    {
        $this->getEntityManager()->remove($subscription);
        $this->getEntityManager()->flush();
    }

    /**
     * @return PushSubscription[]
     */
    public function findByUser(User $user): array
    {
        return $this->findBy(['user' => $user]);
    }

    public function findByEndpoint(string $endpoint): ?PushSubscription
    {
        return $this->findOneBy(['endpoint' => $endpoint]);
    }
}
```
{% endcode %}

### Step 4: Create the Subscription Controller

{% code title="src/Controller/PushSubscriptionController.php" %}
```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Entity\PushSubscription;
use App\Repository\PushSubscriptionRepository;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;
use WebPush\Subscription;

#[Route('/api/push')]
class PushSubscriptionController extends AbstractController
{
    public function __construct(
        private readonly PushSubscriptionRepository $repository
    ) {
    }

    #[Route('/subscribe', name: 'push_subscribe', methods: ['POST'])]
    public function subscribe(Request $request): JsonResponse
    {
        $user = $this->getUser();
        if (!$user) {
            return new JsonResponse(['error' => 'Unauthorized'], Response::HTTP_UNAUTHORIZED);
        }

        $data = json_decode($request->getContent(), true);
        if (!isset($data['endpoint'])) {
            return new JsonResponse(['error' => 'Invalid subscription'], Response::HTTP_BAD_REQUEST);
        }

        // Check if subscription already exists
        $existing = $this->repository->findByEndpoint($data['endpoint']);
        if ($existing) {
            return new JsonResponse(['message' => 'Already subscribed'], Response::HTTP_OK);
        }

        // Create subscription from browser data
        $baseSubscription = Subscription::createFromString(json_encode($data));

        $subscription = new PushSubscription($baseSubscription->getEndpoint(), $user);
        $subscription->withContentEncodings($baseSubscription->getSupportedContentEncodings());

        foreach ($baseSubscription->getKeys() as $key => $value) {
            $subscription->setKey($key, $value);
        }

        $this->repository->save($subscription);

        return new JsonResponse(['message' => 'Subscription saved'], Response::HTTP_CREATED);
    }

    #[Route('/unsubscribe', name: 'push_unsubscribe', methods: ['POST'])]
    public function unsubscribe(Request $request): JsonResponse
    {
        $user = $this->getUser();
        if (!$user) {
            return new JsonResponse(['error' => 'Unauthorized'], Response::HTTP_UNAUTHORIZED);
        }

        $data = json_decode($request->getContent(), true);
        $subscription = $this->repository->findByEndpoint($data['endpoint'] ?? '');

        if (!$subscription || $subscription->getUser() !== $user) {
            return new JsonResponse(['error' => 'Subscription not found'], Response::HTTP_NOT_FOUND);
        }

        $this->repository->remove($subscription);

        return new JsonResponse(['message' => 'Subscription removed'], Response::HTTP_OK);
    }

    #[Route('/public-key', name: 'push_public_key', methods: ['GET'])]
    public function getPublicKey(): JsonResponse
    {
        return new JsonResponse([
            'publicKey' => $this->getParameter('env(WEBPUSH_PUBLIC_KEY)')
        ]);
    }
}
```
{% endcode %}

### Step 5: Create the Notification Service

{% code title="src/Service/PushNotificationService.php" %}
```php
<?php

declare(strict_types=1);

namespace App\Service;

use App\Entity\PushSubscription;
use App\Entity\User;
use App\Repository\PushSubscriptionRepository;
use Psr\Log\LoggerInterface;
use WebPush\Message;
use WebPush\Notification;
use WebPush\WebPushService;

final readonly class PushNotificationService
{
    public function __construct(
        private WebPushService $webPush,
        private PushSubscriptionRepository $repository,
        private LoggerInterface $logger
    ) {
    }

    public function sendToUser(User $user, string $title, string $body, array $data = []): void
    {
        $subscriptions = $this->repository->findByUser($user);

        if (empty($subscriptions)) {
            $this->logger->info('No subscriptions found for user', ['user_id' => $user->getId()]);
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
            $this->sendNotification($notification, $subscription);
        }
    }

    private function sendNotification(Notification $notification, PushSubscription $subscription): void
    {
        try {
            $report = $this->webPush->send($notification, $subscription);

            if ($report->isSubscriptionExpired()) {
                $this->logger->info('Subscription expired, removing', [
                    'endpoint' => $subscription->getEndpoint()
                ]);
                $this->repository->remove($subscription);
            } elseif (!$report->isSuccess()) {
                $this->logger->error('Failed to send notification', [
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

### Step 6: Client-side JavaScript

{% code title="public/js/push-notifications.js" %}
```javascript
// Request notification permission
async function requestNotificationPermission() {
    const permission = await Notification.requestPermission();
    if (permission !== 'granted') {
        console.log('Notification permission denied');
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

        // Get public key from server
        const response = await fetch('/api/push/public-key');
        const { publicKey } = await response.json();

        // Subscribe
        const subscription = await registration.pushManager.subscribe({
            userVisibleOnly: true,
            applicationServerKey: urlBase64ToUint8Array(publicKey)
        });

        // Get supported content encodings
        const supportedContentEncodings = PushManager.supportedContentEncodings || ['aesgcm'];

        // Send subscription to server
        const subscriptionData = {
            endpoint: subscription.endpoint,
            keys: {
                p256dh: arrayBufferToBase64(subscription.getKey('p256dh')),
                auth: arrayBufferToBase64(subscription.getKey('auth'))
            },
            supportedContentEncodings: supportedContentEncodings
        };

        await fetch('/api/push/subscribe', {
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

        await fetch('/api/push/unsubscribe', {
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

    event.waitUntil(
        clients.openWindow('/')
    );
});
```
{% endcode %}

### Step 7: Usage Example

{% code title="src/Controller/NotificationTestController.php" %}
```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Service\PushNotificationService;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;

class NotificationTestController extends AbstractController
{
    #[Route('/test/notification', name: 'test_notification')]
    public function sendTestNotification(PushNotificationService $pushService): Response
    {
        $user = $this->getUser();

        $pushService->sendToUser(
            $user,
            'Test Notification',
            'This is a test notification from your Symfony app!',
            ['url' => '/dashboard']
        );

        return new Response('Notification sent!');
    }
}
```
{% endcode %}

## Demo Application

For a complete working demo application, please visit: [https://github.com/Spomky-Labs/web-push-demo](https://github.com/Spomky-Labs/web-push-demo)
