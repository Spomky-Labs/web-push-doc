# The Web Push Service

The WebPush object requires a [HTTP Client](https://symfony.com/doc/current/http\_client.html) and an [Extension Manager](advanced-service.md).

```php
use Symfony\Component\HttpClient\HttpClient;
use WebPush\WebPush;

$client = HttpClient::create();

$service = new WebPush($client, $extensionManager);
```

The service is now ready to send Notifications to the Subscriptions. The StatusReport object that is returned [is explained here](../common-concepts/the-status-report.md).

```php
<?php

use WebPush\Subscription;
use WebPush\Notification;

$subscription = Subscription::createFromString('{"endpoint":"https://updates.push.services.mozilla.com/wpush/v2/AAAAAAAA[…]AAAAAAAAA","keys":{"auth":"XXXXXXXXXXXXXX","p256dh":"YYYYYYYY[…]YYYYYYYYYYYYY"}}');
$notification = Notification::create()
    ->withPayload('Hello world')
;

$statusReport = $service->send($notification, $subscription);
```

{% hint style="info" %}
In this example, we load the Subscription object from a string, but usually to retrieve the Subscription objects from a database or a dedicated storage.
{% endhint %}
