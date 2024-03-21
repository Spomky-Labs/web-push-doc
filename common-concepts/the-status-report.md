# The Status Report

After sending a notification, you will receive a StatusReport object.

This status report has the following properties:

* The [notification](the-notification.md)
* The [subscription](the-subscription.md)
* The status code
* The notification URL (refers to the push service provider)
* The links for push notification management

Depending on the status code, you will be able to know if it is a success or not. In case of success, you can directly access the management link (`location` header parameter) or the links entity fields in case of asynchronous call. In case of failure, the response code indicates the main reason for rejection (invalid authorization token, expired endpoint...)

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

One of the failure reasons could be the expiration of the subscription (too old or cancelled by the end-user). This can be checked with the method `isSubscriptionExpired()`. In this case, you should simply delete the subscription as it is not possible to send notifications anymore.

```php
<?php

if($statusReport->isSubscriptionExpired()) {
    $this->subscriptionRepository->remove($subscription);
}
```
