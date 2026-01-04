# The Notification

To reach the client (web browser), you need to send a Notification to the Subscription.

```php
<?php
use WebPush\Notification;

$notification = Notification::create();
```

The Notification should have a payload. In this case, the payload will be encrypted on server side and decrypted by the client.

That payload may be a string or a JSON object. The structure of the latter is described in the next section.

```php
<?php
use WebPush\Notification;

$notification = Notification::create()
    ->withPayload('Hello world')
;
```

## TTL (Time-To-Live)

With this feature, a value in seconds is added to the notification. It suggests how long a push message is retained by the push service. A value of 0 (zero) indicates the notification is delivered immediately.

```php
<?php
use WebPush\Notification;

$notification = Notification::create()
    ->withTTL(3600)
;
```

## Topic

A push message that has been stored by the push service can be replaced with new content. If the user agent is offline during the time the push messages are sent, updating a push message avoids the situation where outdated or redundant messages are sent to the user agent.

Only push messages that have been assigned a topic can be replaced. A push message with a topic replaces any outstanding push message with an identical topic.

```php
<?php
use WebPush\Notification;

$notification = Notification::create()
    ->withTopic('user-account-updated')
;
```

## Urgency

For a device that is battery-powered, it is often critical it remains dormant for extended periods.

Radio communication in particular consumes significant power and limits the length of time the device can operate.

To avoid consuming resources to receive trivial messages, it is helpful if an application server can communicate the urgency of a message and if the user agent can request that the push server only forwards messages of a specific urgency.

| Urgency  | Device State               | Examples                                    |
| -------- | -------------------------- | ------------------------------------------- |
| very-low | On power and Wi-Fi         | Advertisements                              |
| low      | On either power or Wi-Fi   | Topic updates                               |
| normal   | On neither power nor Wi-Fi | Chat or Calendar Message                    |
| high     | Low battery                | Incoming phone call or time-sensitive alert |

{% hint style="warning" %}
Be careful with the `very-low` urgency: it is not recognized by all Web-Push services
{% endhint %}

```php
<?php
use WebPush\Notification;

$notification = Notification::create()
    ->veryLowUrgency()
    ->lowUrgency()
    ->normalUrgency()
    ->highUrgency()
;
```

## Asynchronous Response

Your application may prefer asynchronous responses to request confirmation from the push service when a push message is delivered and then acknowledged by the user agent. The push service MUST support delivery confirmations to use this feature.

```php
<?php
use WebPush\Notification;

$notification = Notification::create()
    ->async() // Prefer async response
    ->sync() // Prefer sync response (default)
;
```

{% hint style="warning" %}
The `async` mode is not recognized by all Web Push services. In case of failure, you should try sending `sync` notifications.
{% endhint %}

## JSON Messages

As mentioned in the overview section, the specification [defines a structure for the payload](https://developer.mozilla.org/en-US/docs/Web/API/Notification/Notification#parameters). This structure contains properties that the client should understand and render in an appropriate way.

The library provides a `WebPush\Message` class with convenient methods to ease the creation of a message.

```php
<?php
use WebPush\Action;
use WebPush\Message;
use WebPush\Notification;

$message = Message::create('This is the title')
    ->mute() // Silent
    ->unmute() // Not silent (default)

    ->auto() //Direction = auto (default)
    ->ltr() //Direction = left to right
    ->rtl() //Direction = right to left

    ->addAction(Action::create('alert', 'Click me!'))

    ->interactionRequired()
    ->noInteraction()

    ->renotify()
    ->doNotRenotify() // Default

    ->withBody('Hello World!')

    ->withIcon('https://…')
    ->withImage('https://…')
    ->withData(['foo' => 'BAR']) // Arbitrary data
    ->withBadge('badge1')
    ->withLang('fr-FR')
    ->withTimestamp(time())
    ->withTag('foo')

    ->vibrate(300, 100, 400)

    ->toString() // Converts the Message object into a string
;

$notification = Notification::create()
    ->withPayload($message)
;
```

The resulting notification payload will look as follows:

```javascript
{
    "title":"This is the title",
    "options":{
        "actions":[
            {
                "action":"alert",
                "title":"Click me!"
            }
        ],
        "badge":"badge1",
        "body":"Hello World!",
        "data":{
            "foo":"BAR"
        },
        "dir":"rtl",
        "icon":"https://…",
        "image":"https://…",
        "lang":"fr-FR",
        "renotify":false,
        "requireInteraction":false,
        "silent":false,
        "tag":"foo",
        "timestamp":1629145424,
        "vibrate":[
            300,
            100,
            400
        ]
    }
}
```

On client side, you can easily load that payload and display the notification:

```javascript
self.addEventListener('push', function(event) {
    const data = event.data.json();
    const {title, ...options} = data;

    event.waitUntil(
        self.registration.showNotification(title, options)
    );
});
```

## Best Practices

### Choose Appropriate TTL Values

The Time-To-Live (TTL) determines how long notifications are retained if the user is offline:

```php
// Time-sensitive: expires quickly
$urgentNotification = Notification::create()
    ->withPayload('Flash sale ends in 10 minutes!')
    ->withTTL(600); // 10 minutes

// Important but not urgent
$normalNotification = Notification::create()
    ->withPayload('New message from John')
    ->withTTL(86400); // 24 hours

// Persistent notification
$persistentNotification = Notification::create()
    ->withPayload('New feature available')
    ->withTTL(604800); // 7 days
```

### Use Topics Wisely

Topics prevent notification spam by replacing old notifications with new ones:

```php
// Weather updates - only show the latest
$weatherUpdate = Notification::create()
    ->withTopic('weather-alert')
    ->withPayload($latestWeatherData);

// Stock price updates - replace with latest price
$stockUpdate = Notification::create()
    ->withTopic('stock-AAPL')
    ->withPayload($currentPrice);

// User mentions - don't replace, show all
$mention = Notification::create()
    // No topic - each mention is shown separately
    ->withPayload('Sarah mentioned you in a comment');
```

### Set Appropriate Urgency

Match urgency to content to optimize battery life:

```php
// High urgency - user needs to act immediately
Notification::create()
    ->highUrgency()
    ->withPayload('Your bank account requires immediate attention')
    ->withTTL(3600);

// Normal urgency - standard messages
Notification::create()
    ->normalUrgency()
    ->withPayload('New comment on your post')
    ->withTTL(86400);

// Low urgency - can wait
Notification::create()
    ->lowUrgency()
    ->withPayload('Weekly summary available')
    ->withTTL(604800);

// Very low urgency - promotional content
Notification::create()
    ->veryLowUrgency()
    ->withPayload('Check out our new products')
    ->withTTL(2592000); // 30 days
```

### Craft Effective Messages

Good notification messages are:
- **Clear**: User understands the content immediately
- **Actionable**: User knows what to do next
- **Concise**: Get to the point quickly
- **Relevant**: Personalized to the user

```php
use WebPush\Message;
use WebPush\Action;

// Example: E-commerce order notification
$message = Message::create('Order Shipped!')
    ->withBody('Your order #12345 is on its way')
    ->withIcon('/icons/shipping.png')
    ->withBadge('/icons/badge.png')
    ->withData([
        'orderId' => '12345',
        'trackingUrl' => 'https://example.com/track/12345'
    ])
    ->addAction(Action::create('track', 'Track Package'))
    ->addAction(Action::create('view', 'View Order'))
    ->interactionRequired(); // User must interact

$notification = Notification::create()
    ->withPayload($message->toString())
    ->normalUrgency()
    ->withTTL(86400);
```

### Optimize for Mobile

Mobile devices have limited battery and screen space:

```php
// Mobile-optimized notification
$mobileNotification = Message::create('New Message')
    ->withBody('John: Can we meet tomorrow?')
    ->withIcon('/icons/chat-64.png') // Small icon
    ->withBadge('/icons/unread-badge.png')
    ->withData(['chatId' => '123', 'userId' => '456'])
    ->addAction(Action::create('reply', 'Reply'))
    ->addAction(Action::create('view', 'View'))
    ->withTag('chat-123') // Group related notifications
    ->renotify() // Alert user even if previous notification exists
    ->vibrate(200, 100, 200); // Short vibration pattern
```

## Common Notification Patterns

### 1. Chat Message

```php
$chatMessage = Message::create($senderName)
    ->withBody($messagePreview)
    ->withIcon($senderAvatar)
    ->withBadge('/icons/message-badge.png')
    ->withData([
        'chatId' => $chatId,
        'senderId' => $senderId,
        'messageId' => $messageId
    ])
    ->addAction(Action::create('reply', 'Reply'))
    ->addAction(Action::create('view', 'View'))
    ->withTag("chat-{$chatId}") // Group by conversation
    ->renotify()
    ->withTimestamp(time());

$notification = Notification::create()
    ->withPayload($chatMessage->toString())
    ->highUrgency()
    ->withTTL(3600);
```

### 2. System Alert

```php
$alert = Message::create('System Maintenance')
    ->withBody('Scheduled maintenance in 30 minutes')
    ->withIcon('/icons/warning.png')
    ->withBadge('/icons/alert-badge.png')
    ->interactionRequired()
    ->withTag('system-maintenance')
    ->vibrate(300, 200, 300);

$notification = Notification::create()
    ->withPayload($alert->toString())
    ->highUrgency()
    ->withTTL(1800);
```

### 3. News Update

```php
$news = Message::create('Breaking News')
    ->withBody($headline)
    ->withIcon('/icons/news.png')
    ->withImage($articleImage)
    ->withData(['articleId' => $articleId])
    ->addAction(Action::create('read', 'Read Article'))
    ->withTag('news')
    ->withTimestamp(time());

$notification = Notification::create()
    ->withPayload($news->toString())
    ->normalUrgency()
    ->withTopic('breaking-news')
    ->withTTL(86400);
```

### 4. Silent Background Sync

```php
// Silent notification for background sync
$syncNotification = Notification::create()
    ->withPayload(json_encode(['type' => 'sync', 'data' => $syncData]))
    ->async()
    ->lowUrgency()
    ->withTTL(0); // Deliver immediately or not at all
```

## Handling Delivery Failures

Always handle failures gracefully:

```php
use WebPush\Notification;
use WebPush\WebPushService;

/** @var WebPushService $webPush */
/** @var Subscription $subscription */

try {
    $notification = Notification::create()
        ->withPayload($message->toString())
        ->withTTL(86400);

    $report = $webPush->send($notification, $subscription);

    if (!$report->isSuccess()) {
        // Log the failure
        $logger->error('Notification delivery failed', [
            'subscription' => $subscription->getEndpoint(),
            'error' => $report->getLocation()
        ]);

        // Handle expired subscriptions
        if ($report->isSubscriptionExpired()) {
            $subscriptionRepository->remove($subscription);
        }
    }
} catch (\Throwable $e) {
    $logger->error('Exception sending notification', [
        'message' => $e->getMessage(),
        'subscription' => $subscription->getEndpoint()
    ]);
}
```

## Testing Notifications

Always test your notifications before sending to production:

```php
// Create a test notification
$testNotification = Message::create('Test Notification')
    ->withBody('This is a test')
    ->withIcon('/test-icon.png')
    ->toString();

// Send to your own subscription
$report = $webPush->send(
    Notification::create()->withPayload($testNotification),
    $yourTestSubscription
);

// Verify delivery
assert($report->isSuccess(), 'Test notification failed to send');
```

## Performance Considerations

### Batch Sending

When sending to many subscriptions, batch efficiently:

```php
$subscriptions = $repository->getAllActive();
$notification = Notification::create()->withPayload($message);

foreach ($subscriptions as $subscription) {
    // Send asynchronously when possible
    $report = $webPush->send($notification, $subscription);

    // Cleanup expired subscriptions immediately
    if ($report->isSubscriptionExpired()) {
        $repository->remove($subscription);
    }
}
```

### Respect Rate Limits

Push services have rate limits. Implement throttling:

```php
$rateLimit = 100; // notifications per second
$delay = 1000000 / $rateLimit; // microseconds

foreach ($subscriptions as $subscription) {
    $webPush->send($notification, $subscription);
    usleep($delay);
}
```

## Next Steps

- Understand [Status Reports](the-status-report.md) to handle delivery results
- Learn about [Subscriptions](the-subscription.md) management
- Set up [VAPID](vapid.md) for secure authentication
