# The Subscription

The subscription is created on client side when the end-user allows your application to send push messages.

A subscription is a unique identifier that represents the user's device and browser. It contains:
- An **endpoint URL** provided by the push service
- **Encryption keys** for securing the message payload
- **Supported content encodings** for payload encryption

## Creating a Subscription

On client side (Javascript), you can simply send to your server the object you receive using `JSON.stringify`.

{% hint style="info" %}
Javascript examples to get a Subscription from the web browser are not provided here. Please refer to other resources such as blog posts or library documentation.
{% endhint %}

A subscription object will look like:

```javascript
{
 "endpoint":"https://updates.push.services.mozilla.com/wpush/v2/AAAAAAAA[…]AAAAAAAAA",
 "keys":{
 "auth":"XXXXXXXXXXXXXX",
 "p256dh":"YYYYYYYY[…]YYYYYYYYYYYYY"
 }
}
```

### Understanding the Subscription Components

- **endpoint**: The unique URL where your server sends notifications. Each subscription has a different endpoint.
- **keys.auth**: Authentication secret for message encryption
- **keys.p256dh**: Public key for Elliptic Curve Diffie-Hellman key agreement

## Server-Side Processing

On server side, you can get a `WebPush\Subscription` object from the JSON string using the dedicated method `WebPush\Subscription::createFromString`.

```php
<?php

use WebPush\Subscription;

$subscription = Subscription::createFromString('{"endpoint":"https://updates.push.services.mozilla.com/wpush/v2/AAAAAAAA[…]AAAAAAAAA","keys":{"auth":"XXXXXXXXXXXXXX","p256dh":"YYYYYYYY[…]YYYYYYYYYYYYY"}}');
```

## Supported Content Encodings

By default, the content encoding `aesgcm` will be used. This encoding indicates how the payload of the notification should be formatted. The PushManager object from the Push API may list all acceptable encodings. In this case, it could be interesting to set these encodings to the Subscription object.

The two standard encodings are:
- **aes128gcm** (recommended): Newer, more efficient encoding defined in RFC 8291
- **aesgcm** (legacy): Older encoding for backwards compatibility

```javascript
// Retrieve the supported content encodings
const supportedContentEncodings = PushManager.supportedContentEncodings || ['aesgcm'];

// Assign the encodings to the subscription object
const jsonSubscription = Object.assign(
    subscription.toJSON(),
    { supportedContentEncodings }
);

// Send the subscription object to the application server
fetch('/subscription/add', {
    method: 'POST',
    body: JSON.stringify(jsonSubscription),
});
```

This will result in something like the following:

```javascript
{
 "endpoint":"https://updates.push.services.mozilla.com/wpush/v2/AAAAAAAA[…]AAAAAAAAA",
 "keys":{
 "auth":"XXXXXXXXXXXXXX",
 "p256dh":"YYYYYYYY[…]YYYYYYYYYYYYY"
 },
 "supportedContentEncodings":["aes128gcm","aesgcm"]
}
```

{% hint style="warning" %}
The order of `supportedContentEncodings` is important. First supported item will be used. If possible, `AES128GCM` should be used as preferred content encoding.
{% endhint %}

## Subscription Lifecycle

### 1. Creation
A subscription is created when the user grants notification permission and your service worker subscribes to push notifications.

### 2. Storage
Your application server must store the subscription to send notifications later. Store:
- The complete subscription object
- Associated user information
- Creation timestamp (useful for cleanup)

### 3. Usage
Use the stored subscription to send notifications at any time, even when the user is not on your website.

### 4. Expiration
Subscriptions can expire for several reasons:
- User revokes notification permission
- User clears browser data
- Push service expires old subscriptions
- Subscription endpoint becomes invalid

### 5. Cleanup
Always handle expired subscriptions:
- Remove them from your database when you receive a 404 or 410 error
- Implement a strategy to detect and clean abandoned subscriptions

## Best Practices

### Store Essential Information
```php
// Minimum data to store
$subscriptionData = [
    'user_id' => $userId,
    'endpoint' => $subscription->getEndpoint(),
    'keys' => $subscription->getKeys(),
    'encodings' => $subscription->getSupportedContentEncodings(),
    'created_at' => new DateTime(),
];
```

### Handle Multiple Subscriptions per User
A single user may have multiple subscriptions (different devices/browsers). Store all of them:

```php
// Send to all user's subscriptions
foreach ($userSubscriptions as $subscription) {
    $report = $webPush->send($notification, $subscription);

    if ($report->isSubscriptionExpired()) {
        // Remove this specific subscription
        $repository->remove($subscription);
    }
}
```

### Implement Subscription Refresh
If a subscription expires, prompt the user to resubscribe:

```javascript
// Check if subscription is still valid
const subscription = await registration.pushManager.getSubscription();
if (!subscription) {
    // Subscription was revoked, show UI to resubscribe
    showResubscribePrompt();
}
```

### Security Considerations

1. **Validate Endpoint URLs**: Ensure endpoints are from known push services
2. **Store Securely**: Treat subscriptions as sensitive data
3. **Rate Limiting**: Implement rate limits to prevent abuse
4. **User Association**: Always associate subscriptions with authenticated users

```php
// Validate endpoint before storing
$endpoint = $subscription->getEndpoint();
$allowedDomains = [
    'fcm.googleapis.com',
    'updates.push.services.mozilla.com',
    'web.push.apple.com',
];

$isValid = false;
foreach ($allowedDomains as $domain) {
    if (str_contains($endpoint, $domain)) {
        $isValid = true;
        break;
    }
}

if (!$isValid) {
    throw new \InvalidArgumentException('Invalid push service endpoint');
}
```

## Subscription Uniqueness

Each subscription is unique to:
- A specific browser
- A specific device
- A specific user profile (if the browser has multiple profiles)
- A specific origin (your website domain)

This means:
- The same user on Chrome desktop and Firefox desktop will have 2 different subscriptions
- The same user on desktop and mobile will have 2 different subscriptions
- If a user clears their browser data, they will get a new subscription

## Testing Subscriptions

Always test your subscription handling:

```php
// Test with a real subscription
$testSubscription = Subscription::createFromString($jsonFromBrowser);

// Verify it has all required components
assert($testSubscription->getEndpoint() !== '');
assert(count($testSubscription->getKeys()) >= 2);
assert(in_array('auth', array_keys($testSubscription->getKeys())));
assert(in_array('p256dh', array_keys($testSubscription->getKeys())));
```

## Performance Optimization

### Caching Subscriptions

When sending notifications to multiple users or sending frequently, implement caching strategies to improve performance:

#### Application-Level Caching

Use your application's cache layer (Redis, Memcached, etc.) to cache subscription lookups:

```php
use Psr\Cache\CacheItemPoolInterface;
use WebPush\Subscription;

class CachedSubscriptionRepository
{
    public function __construct(
        private SubscriptionRepository $repository,
        private CacheItemPoolInterface $cache
    ) {}

    public function findByUserId(string $userId): ?Subscription
    {
        $cacheKey = "subscription.user.{$userId}";
        $item = $this->cache->getItem($cacheKey);

        if ($item->isHit()) {
            $json = $item->get();
            return Subscription::createFromString($json);
        }

        $subscription = $this->repository->findByUserId($userId);
        if ($subscription !== null) {
            $item->set($subscription->toString());
            $item->expiresAfter(3600); // Cache for 1 hour
            $this->cache->save($item);
        }

        return $subscription;
    }

    public function invalidate(string $userId): void
    {
        $cacheKey = "subscription.user.{$userId}";
        $this->cache->deleteItem($cacheKey);
    }
}
```

#### Doctrine Query Result Cache

If using Doctrine, leverage query result caching:

```php
use Doctrine\ORM\EntityRepository;

class UserRepository extends EntityRepository
{
    public function findWithSubscription(int $userId): ?User
    {
        return $this->createQueryBuilder('u')
            ->leftJoin('u.subscription', 's')
            ->addSelect('s')
            ->where('u.id = :userId')
            ->setParameter('userId', $userId)
            ->getQuery()
            ->useResultCache(true, 3600, "user_subscription_{$userId}")
            ->getOneOrNullResult();
    }
}
```

#### Batch Loading

When sending to multiple users, load subscriptions in batches to reduce database queries:

```php
use WebPush\Notification;
use WebPush\WebPushService;

class BulkNotificationSender
{
    public function __construct(
        private WebPushService $webPush,
        private EntityManagerInterface $em
    ) {}

    public function sendToUsers(Notification $notification, array $userIds): array
    {
        // Load all subscriptions in one query
        $subscriptions = $this->em->createQueryBuilder()
            ->select('s')
            ->from(UserSubscription::class, 's')
            ->where('s.userId IN (:userIds)')
            ->setParameter('userIds', $userIds)
            ->getQuery()
            ->getResult();

        // Send to all subscriptions
        return $this->webPush->sendToMultiple($notification, $subscriptions);
    }
}
```

#### Cache Invalidation Strategy

Always invalidate the cache when subscriptions change:

```php
class SubscriptionService
{
    public function __construct(
        private SubscriptionRepository $repository,
        private CacheItemPoolInterface $cache
    ) {}

    public function updateSubscription(string $userId, string $subscriptionJson): void
    {
        $subscription = Subscription::createFromString($subscriptionJson);
        $this->repository->save($userId, $subscription);

        // Invalidate cache
        $this->cache->deleteItem("subscription.user.{$userId}");
    }

    public function removeExpiredSubscription(string $userId): void
    {
        $this->repository->remove($userId);

        // Invalidate cache
        $this->cache->deleteItem("subscription.user.{$userId}");
    }
}
```

### Best Practices for High-Volume Sending

When sending to thousands of users:

1. **Use Background Jobs**: Queue notifications for asynchronous processing
2. **Batch Subscriptions**: Load subscriptions in batches of 100-1000
3. **Cache Aggressively**: Cache subscription lookups for 1-4 hours
4. **Monitor Cache Hit Rate**: Aim for >80% hit rate
5. **Clean Up Expired**: Remove expired subscriptions immediately to reduce database size

```php
use Symfony\Component\Messenger\MessageBusInterface;

class NotificationDispatcher
{
    public function __construct(
        private MessageBusInterface $bus,
        private CacheItemPoolInterface $cache
    ) {}

    public function sendToAllUsers(Notification $notification): void
    {
        // Get cached user count
        $userCount = $this->getCachedUserCount();
        $batchSize = 1000;

        // Dispatch batch jobs
        for ($offset = 0; $offset < $userCount; $offset += $batchSize) {
            $this->bus->dispatch(new SendNotificationBatch(
                $notification,
                $offset,
                $batchSize
            ));
        }
    }

    private function getCachedUserCount(): int
    {
        $item = $this->cache->getItem('user.count');
        if ($item->isHit()) {
            return $item->get();
        }

        $count = $this->countUsers();
        $item->set($count);
        $item->expiresAfter(300); // Cache for 5 minutes
        $this->cache->save($item);

        return $count;
    }
}
```

{% hint style="info" %}
**Note**: The web-push library itself does not provide caching functionality, as this is the responsibility of your application layer. The examples above show recommended patterns for implementing caching in your application.
{% endhint %}

## Common Issues

### Subscription Not Received
- Check that the user granted permission
- Verify HTTPS is enabled
- Ensure service worker is properly registered

### Subscription Immediately Expires
- Check VAPID keys match between client and server
- Verify the push service endpoint is accessible
- Ensure proper payload encryption

### Multiple Subscriptions for Same User
This is normal and expected. Users can have multiple devices and browsers.

## Next Steps

- Learn how to create [Notifications](the-notification.md) to send to your subscriptions
- Understand [Status Reports](the-status-report.md) to handle delivery results
- Set up [VAPID](vapid.md) authentication for secure sending
