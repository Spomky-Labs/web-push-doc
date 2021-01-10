# The Web Push Service

The WebPush object requires a PSR-17 Request Factory, a PSR-18 Http Client and an [Extension Manager](advanced-service.md).

```php
use Nyholm\Psr7\Factory\Psr17Factory;
use Symfony\Component\HttpClient\Psr18Client;
use WebPush\WebPush;

$client = new Psr18Client();
$requestFactory = new Psr17Factory();

$service = new WebPush($client, $requestFactory, $extensionManager);
```

The service is now ready to send Notifications to the Subscriptions. The StatusReport object that is returned [is explained here](../common-concepts/the-satus-report.md).

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

