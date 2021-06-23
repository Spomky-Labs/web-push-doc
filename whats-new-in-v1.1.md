# What’s New in v1.1?

## The Library

### New Interfaces

The following interfaces have been introduced:

* `WebPush\NotificationInterface`
* `WebPush\SubscriptionInterface`
* `WebPush\StatusReportInterface`

It is no longer required to use the concrete classes. You can use your own objects to be transported and sent.

Methods provided by the WebPush service, the extensions or the extension manager has been updated to always use the interfaces instead of the classes.

### Deprecations

The methods `getResponse` and `getRequest` of the class `WebPush\StatusReport` are now deprecated. They should not used anymore and will be removed in `v2.0`.

The method `notificationExpired` of the class `WebPush\StatusReport` is deprecated. Please use `isSubscriptionExpired` instead.

### The Symfony Bundle

### New Service

The service `web_push.service` has been added. This new service uses the same interface as `WebPush\WebPush`. The main difference is that this service uses the Symfony Http Client and allow you to take advantages of the asynchronous request calls.

{% code title="before.php" %}
```php
use WebPush\WebPush;

$webPush = $this->container->get(WebPush::class);
$reports = [];
foreach ($subscriptions as $subscription) { 
    $reports[] = $webPush->send($notification, $subscription); // Synchronous calls
}

foreach ($reports as $report) { 
    if ($report->isSubscriptionExpired()) {
        // Synchronous calls
    }
}
```
{% endcode %}

{% code title="after.php" %}
```php
use WebPush\WebPush;

$webPush = $this->container->get('web_push.service');
$reports = [];
foreach ($subscriptions as $subscription) { 
    $reports[] = $webPush->send($notification, $subscription); // Asynchronous calls
}

foreach ($reports as $report) { 
    if ($report->isSubscriptionExpired()) {
        // Synchronous calls as usual
    }
}
```
{% endcode %}

 

