# The Web Push Service

The WebPush object requires a [HTTP Client](https://symfony.com/doc/current/http_client.html) and an [Extension Manager](advanced-service.md).

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
In this example, we load the Subscription object from a string, but you usually retrieve the Subscription objects from a database or a dedicated storage.
{% endhint %}

## Sending to Multiple Subscriptions

The `sendToMultiple()` method allows you to send a notification to multiple subscriptions efficiently:

```php
<?php

use WebPush\Notification;
use WebPush\StatusReport;

$notification = Notification::create()
    ->withPayload('{"title":"Breaking News","body":"Check this out"}')
    ->withTTL(Notification::TTL_ONE_HOUR);

// Send to multiple subscriptions
$reports = $service->sendToMultiple($notification, $subscriptions);

// Process results using helper methods
$successful = StatusReport::filterSuccessful($reports);
$failed = StatusReport::filterFailed($reports);
$expired = StatusReport::filterExpired($reports);

// Get statistics
$stats = StatusReport::getStatistics($reports);
// Returns: ['total' => 100, 'successful' => 85, 'failed' => 15, 'expired' => 5, 'retryable' => 3]

// Handle expired subscriptions
foreach ($expired as $report) {
    // Remove from database
    $repository->remove($report->getSubscription());
}

// Handle retryable errors
$retryable = StatusReport::filterRetryable($reports);
foreach ($retryable as $report) {
    // Queue for retry
    $queue->retry($report->getNotification(), $report->getSubscription());
}
```

{% hint style="info" %}
Unlike `send()`, the `sendToMultiple()` method does not throw exceptions for individual failures. It attempts to send to all subscriptions and returns a StatusReport for each one, allowing you to inspect both successes and failures.
{% endhint %}

## Error Handling

The service may throw exceptions for transport or HTTP errors:

```php
<?php

use WebPush\Exception\OperationException;

try {
    $report = $service->send($notification, $subscription);

    if ($report->isSuccess()) {
        // Success!
    } elseif ($report->isSubscriptionExpired()) {
        // Remove expired subscription
        $repository->remove($subscription);
    } elseif ($report->isRetryable()) {
        // Queue for retry (5xx or 429 errors)
        $queue->retry($notification, $subscription);
    } else {
        // Log permanent failure
        $logger->error('Failed to send notification', [
            'error' => $report->getErrorMessage(),
            'status_code' => $report->getStatusCode()
        ]);
    }
} catch (OperationException $e) {
    // Network or transport error
    $logger->error('Transport error', ['message' => $e->getMessage()]);
}
```

## Validation Exceptions

When creating notifications with invalid properties, the library throws specific exceptions. Each exception exposes the problematic value as a `public readonly` property for easy debugging:

```php
<?php

use WebPush\Exception\InvalidTopicException;
use WebPush\Exception\InvalidTTLException;
use WebPush\Exception\InvalidUrgencyException;
use WebPush\Exception\ValidationException;
use WebPush\Notification;

// Example: Validating user input
function createNotificationFromInput(array $input): Notification
{
    try {
        return Notification::create()
            ->withTopic($input['topic'])
            ->withTTL((int) $input['ttl'])
            ->withUrgency($input['urgency'])
            ->withPayload($input['message']);

    } catch (InvalidTopicException $e) {
        throw new \InvalidArgumentException(
            "Invalid topic '{$e->topic}': must be max 32 chars with URL-safe characters only"
        );
    } catch (InvalidTTLException $e) {
        throw new \InvalidArgumentException(
            "Invalid TTL '{$e->ttl}': must be a positive integer"
        );
    } catch (InvalidUrgencyException $e) {
        throw new \InvalidArgumentException(
            "Invalid urgency '{$e->urgency}': must be 'very-low', 'low', 'normal', or 'high'"
        );
    }
}
```

See the [Exceptions](../common-concepts/exceptions.md) documentation for complete details on error handling strategies.
