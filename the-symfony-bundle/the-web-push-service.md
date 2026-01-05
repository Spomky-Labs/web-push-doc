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

        // Send to all subscriptions at once
        $reports = $this->webPush->sendToMultiple($notification, $subscriptions);

        // Handle expired subscriptions
        $expired = \WebPush\StatusReport::filterExpired($reports);
        foreach ($expired as $report) {
            // Dispatch a message to delete expired subscription
            $this->messageBus->dispatch(
                new SubscriptionExpired($report->getSubscription())
            );
        }

        // Optionally: handle retryable errors
        $retryable = \WebPush\StatusReport::filterRetryable($reports);
        foreach ($retryable as $report) {
            // Queue for retry (5xx or 429 errors)
            $this->messageBus->dispatch(
                new RetryNotification($report->getNotification(), $report->getSubscription())
            );
        }
    }
}
```
{% endcode %}
