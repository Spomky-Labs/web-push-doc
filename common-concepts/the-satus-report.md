# The Satus Report

After sending a notification, you will receive a StatusReport object.

This status report has the following properties:

* The [notification](the-notification.md)
* The [subscription](the-subscription.md)
* The PSR-7 request
* The PSR-7 response

{% hint style="warning" %}
Because of the presence of the Request and Response object, the StatusReport object cannot be serialized.
{% endhint %}

Depending on the status code, you will be able to know if it is a success or not. In case of success, you can directly access the management link \(`location` header parameter\) or the links entity fields in case of asynchronous call. In case of failure, the response code indicates the main reason for rejection \(invalid authorization token, expired endpoint...\)

```php
<?php
use WebPush\Subscription;
use WebPush\Notification;
use WebPush\WebPushService;

/** @var Notification $notification */
/** @var Subscription $subscription */
/** @var WebPushService $webPushService */
$statusReport = $webPushService->send($notification, $subscription);

if(!$statusReport->isSuccess()) {
    //Something went wrong
} else {
    $statusReport->getLocation();
    $statusReport->getLinks();
}
```

One of the failure reasons could be the expiration of the subscription \(too old or cancelled by the end-user\). This can be checked with the method `hasExpired()`. In this case, you should simply delete the subscription as it will always fail.

```php
<?php

if($statusReport->hasExpired()) {
    $this->subscriptionRepository->remove($statusReport);
}
```

