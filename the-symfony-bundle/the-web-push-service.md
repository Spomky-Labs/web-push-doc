# The Web Push Service

The bundle provides a public Web Push service that you can inject into your application components.

In the following example, let's imagine that a notification is dispatched using the Symfony Messenger component and caught by an event handler. This handler will fetch all subscriptions and send the notification.

{% hint style="info" %}
The SubscriptionRepository class is totally fictive
{% endhint %}

{% code title="src/MessageHandler/SendNotification.php" %}
```php
<?php

declare(strict_types=1);

namespace App\MessageHandler;

use App\Message\SubscriptionExpired;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;
use Symfony\Component\Messenger\MessageBusInterface;
use WebPush\Notification;
use WebPush\WebPushService;

#[AsMessageHandler]
final readonly class SendPushNotifications
{
    public function __construct(
        private MessageBusInterface $messageBus,
        private SubscriptionRepository $repository,
        private WebPushService $webPush
    ) {
    }

    public function __invoke(Notification $notification): void
    {
        // Fetch all subscriptions
        $subscriptions = $this->repository->fetchAllSubscriptions();
        foreach ($subscriptions as $subscription) {
            // Sends the notification to the subscriber
            $report = $this->webPush->send($notification, $subscription);

            // If the subscription expired
            if ($report->isSubscriptionExpired()) {
                // We dispatch a new message and expect the
                // subscription to be deleted
                $this->messageBus->dispatch(
                    new SubscriptionExpired($subscription)
                );
            }
        }
    }
}
```
{% endcode %}
