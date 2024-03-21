# The Web Push Service

The bundle provides a public Web Push service that you can inject this service into your application components.

In the following example, let's imagine that a notification is dispatched using the Symfony Messanger component and catched by an event handler. This handler will fetch all subscriptions and send the notification.

{% hint style="info" %}
The SubscriptionRepository class is totally fictive
{% endhint %}

{% code title="src/MessageHandler/SendNotification.php" %}
```php
<?php

declare(strict_types=1);

namespace App\MessageHandler;

use App\Message\SubscriptionExpired;
use Symfony\Component\Messenger\Handler\MessageHandlerInterface;
use Symfony\Component\Messenger\MessageBusInterface;
use WebPush\Notification;
use WebPush\WebPush;

final class SendPushNotifications implements MessageHandlerInterface
{
    private MessageBusInterface $messageBus;
    private SubscriptionRepository $repository;
    private WebPush $webPush;

    public function __construct(MessageBusInterface $messageBus, SubscriptionRepository $repository, WebPush $webPush)
    {
        $this->messageBus = $messageBus;
        $this->repository = $repository;
        $this->webPush = $webPush;
    }

    public function __invoke(Notification $notification): void
    {
        // Fetch all subscriptions
        $subscriptions = $this->repository->fetchAllSubscriptions();
        foreach ($subscriptions as $subscription) {
            //Sends the notification to the subscriber
            $report = $this->webPush->send($notification, $subscription);

            //If the subscription expired
            if ($report->subscriptionExpired()) {
                //We dispatch a new message and expect for
                // the subscription to be deleted
                $this->messageBus->dispatch(
                    new SubscriptionExpired($subscription)
                );
            }
        }
    }
}
```
{% endcode %}
