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

## Validation Exceptions

When creating notifications from user input or configuration, validation exceptions provide clear error messages with contextual properties:

{% code title="src/Service/NotificationFactory.php" %}
```php
<?php

declare(strict_types=1);

namespace App\Service;

use Psr\Log\LoggerInterface;
use WebPush\Exception\InvalidTopicException;
use WebPush\Exception\InvalidTTLException;
use WebPush\Exception\ValidationException;
use WebPush\Notification;

final readonly class NotificationFactory
{
    public function __construct(
        private LoggerInterface $logger
    ) {}

    public function createFromRequest(array $data): ?Notification
    {
        try {
            return Notification::create()
                ->withTopic($data['topic'] ?? 'default')
                ->withTTL($data['ttl'] ?? Notification::TTL_ONE_HOUR)
                ->withPayload($data['message'] ?? '');

        } catch (InvalidTopicException $e) {
            $this->logger->error('Invalid topic in request', [
                'topic' => $e->topic,
                'error' => $e->getMessage()
            ]);
            return null;

        } catch (InvalidTTLException $e) {
            $this->logger->error('Invalid TTL in request', [
                'ttl' => $e->ttl,
                'error' => $e->getMessage()
            ]);
            return null;

        } catch (ValidationException $e) {
            $this->logger->error('Validation error', [
                'error' => $e->getMessage()
            ]);
            return null;
        }
    }
}
```
{% endcode %}

See the [Exceptions](../common-concepts/exceptions.md) documentation for complete error handling strategies.
