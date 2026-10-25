# The Status Report

After sending a notification, you will receive a StatusReport object.

This status report has the following properties:

* The [notification](the-notification.md)
* The [subscription](the-subscription.md)
* The status code
* The notification URL (refers to the push service provider)
* The links for push notification management

Depending on the status code, you will be able to know if it is a success or not. In case of success, you can directly access the management link (`location` header parameter) or the links entity fields in case of asynchronous call. In case of failure, the response code indicates the main reason for rejection (invalid authorization token, expired endpoint...)

## Basic Usage

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

## Helper Methods

The StatusReport provides several helper methods to simplify error handling:

```php
// Get the HTTP status code
$code = $statusReport->getStatusCode(); // 200, 404, 500, etc.

// Check error types
$statusReport->isSuccess();           // true for 2xx codes
$statusReport->isClientError();       // true for 4xx codes
$statusReport->isServerError();       // true for 5xx codes
$statusReport->isRateLimited();       // true for 429
$statusReport->isSubscriptionExpired(); // true for 404 or 410
$statusReport->isTransportError();    // true if no HTTP response (network error)

// Check if the error is retryable
$statusReport->isRetryable();         // true for 5xx or 429

// Get a human-readable error message
$message = $statusReport->getErrorMessage();
// Returns: "Rate limit exceeded", "Subscription expired", etc.
```

### Simplified Error Handling

```php
$report = $webPush->send($notification, $subscription);

if ($report->isSuccess()) {
    // Notification delivered successfully
    return;
}

// Handle subscription expiration
if ($report->isSubscriptionExpired()) {
    $repository->remove($subscription);
    return;
}

// Log error with readable message
$logger->error('Failed to send notification', [
    'endpoint' => $subscription->getEndpoint(),
    'error' => $report->getErrorMessage(),
    'status_code' => $report->getStatusCode()
]);

// Retry if the error is retryable
if ($report->isRetryable()) {
    // Queue for retry with exponential backoff
    $queue->retry($notification, $subscription);
}
```

## Understanding Status Codes

The status code follows HTTP standards and indicates the result of the push notification delivery.

### Success Codes (2xx)

#### 201 Created
The notification was successfully sent and queued for delivery.

```php
if ($statusReport->isSuccess()) {
    // Notification accepted by push service
    $location = $statusReport->getLocation();
    // Store location for tracking if needed
}
```

### Client Error Codes (4xx)

#### 400 Bad Request
The request was malformed or invalid.

**Common causes:**
- Invalid payload encryption
- Malformed subscription
- Missing required headers

```php
if ($report->getStatusCode() === 400) {
    $logger->error('Bad request', [
        'subscription' => $subscription->getEndpoint()
    ]);
    // Check your payload encryption and subscription format
}
```

#### 401 Unauthorized
The VAPID authentication failed.

**Common causes:**
- Invalid VAPID signature
- Expired VAPID token
- Mismatched VAPID public key

```php
if ($report->getStatusCode() === 401) {
    $logger->critical('VAPID authentication failed', [
        'endpoint' => $subscription->getEndpoint()
    ]);
    // Check your VAPID keys and token generation
}
```

#### 404 Not Found
The subscription endpoint no longer exists.

**Action required:** Remove the subscription from your database.

```php
if ($report->getStatusCode() === 404) {
    $logger->info('Subscription not found', [
        'endpoint' => $subscription->getEndpoint()
    ]);
    $subscriptionRepository->remove($subscription);
}
```

#### 410 Gone
The subscription has expired and will never be valid again.

**Action required:** Remove the subscription from your database.

```php
if ($report->getStatusCode() === 410) {
    $logger->info('Subscription expired', [
        'endpoint' => $subscription->getEndpoint()
    ]);
    $subscriptionRepository->remove($subscription);
}
```

{% hint style="info" %}
One of the failure reasons could be the expiration of the subscription (too old or cancelled by the end-user). This can be checked with the method `isSubscriptionExpired()`. In this case, you should simply delete the subscription as it is not possible to send notifications anymore.
{% endhint %}

```php
<?php

if($statusReport->isSubscriptionExpired()) {
    $this->subscriptionRepository->remove($subscription);
}
```

#### 413 Payload Too Large
The notification payload exceeds the size limit.

**Maximum sizes:**
- Chrome/Edge: 4096 bytes
- Firefox: 4096 bytes
- Safari: 4096 bytes

```php
if ($report->getStatusCode() === 413) {
    $logger->warning('Payload too large', [
        'size' => strlen($notification->getPayload())
    ]);
    // Reduce your notification payload size
}
```

#### 429 Too Many Requests
You've exceeded the rate limit for the push service.

**Action required:** Implement rate limiting and exponential backoff.

```php
if ($report->getStatusCode() === 429) {
    $logger->warning('Rate limit exceeded');
    // Wait and retry with exponential backoff
    sleep(60); // Wait 1 minute before retrying
}
```

### Server Error Codes (5xx)

#### 500 Internal Server Error
The push service encountered an error.

**Action:** Retry with exponential backoff.

#### 502 Bad Gateway
The push service is temporarily unavailable.

**Action:** Retry with exponential backoff.

#### 503 Service Unavailable
The push service is temporarily overloaded.

**Action:** Retry later.

## Handling Errors

### Comprehensive Error Handling

```php
use WebPush\Notification;
use WebPush\WebPushService;
use Psr\Log\LoggerInterface;

function sendWithErrorHandling(
    WebPushService $webPush,
    Notification $notification,
    Subscription $subscription,
    LoggerInterface $logger,
    SubscriptionRepository $repository
): bool {
    try {
        $report = $webPush->send($notification, $subscription);

        // Success
        if ($report->isSuccess()) {
            $logger->info('Notification sent successfully', [
                'endpoint' => $subscription->getEndpoint()
            ]);
            return true;
        }

        // Subscription expired
        if ($report->isSubscriptionExpired()) {
            $logger->info('Subscription expired, removing', [
                'endpoint' => $subscription->getEndpoint()
            ]);
            $repository->remove($subscription);
            return false;
        }

        // Handle specific error codes
        $statusCode = $report->getStatusCode();
        switch ($statusCode) {
            case 400:
                $logger->error('Bad request - check payload format');
                break;

            case 401:
                $logger->critical('VAPID authentication failed');
                break;

            case 413:
                $logger->warning('Payload too large');
                break;

            case 429:
                $logger->warning('Rate limit exceeded, will retry later');
                // Implement retry logic
                break;

            case 500:
            case 502:
            case 503:
                $logger->error('Push service error, will retry', [
                    'code' => $statusCode
                ]);
                // Implement retry with exponential backoff
                break;

            default:
                $logger->error('Unknown error', [
                    'code' => $statusCode
                ]);
        }

        return false;

    } catch (\Throwable $e) {
        $logger->error('Exception sending notification', [
            'message' => $e->getMessage(),
            'endpoint' => $subscription->getEndpoint()
        ]);
        return false;
    }
}
```

### Retry Strategy with Exponential Backoff

```php
function sendWithRetry(
    WebPushService $webPush,
    Notification $notification,
    Subscription $subscription,
    int $maxRetries = 3
): bool {
    $attempt = 0;
    $delay = 1; // Initial delay in seconds

    while ($attempt < $maxRetries) {
        $report = $webPush->send($notification, $subscription);

        if ($report->isSuccess()) {
            return true;
        }

        // Don't retry on client errors (except rate limit)
        $code = $report->getStatusCode();
        if ($code >= 400 && $code < 500 && $code !== 429) {
            return false;
        }

        // Exponential backoff
        $attempt++;
        if ($attempt < $maxRetries) {
            sleep($delay);
            $delay *= 2; // Double the delay for next retry
        }
    }

    return false;
}
```

## Batch Processing with Error Handling

### Using sendToMultiple()

The recommended way to send notifications to multiple subscriptions is using the `sendToMultiple()` method:

```php
use WebPush\Notification;
use WebPush\WebPush;
use WebPush\StatusReport;

$notification = Notification::create()
    ->withPayload('{"title":"Update","body":"New content available"}')
    ->withTTL(Notification::TTL_ONE_HOUR);

// Send to multiple subscriptions
$reports = $webPush->sendToMultiple($notification, $subscriptions);

// Process results
foreach ($reports as $report) {
    if ($report->isSubscriptionExpired()) {
        $repository->remove($report->getSubscription());
    } elseif (!$report->isSuccess()) {
        $logger->error('Failed to send', [
            'endpoint' => $report->getSubscription()->getEndpoint(),
            'error' => $report->getErrorMessage()
        ]);
    }
}
```

{% hint style="info" %}
The `sendToMultiple()` method does not throw exceptions for individual failures. Instead, it attempts to send to all subscriptions and returns a StatusReport for each one, allowing you to inspect successes and failures.
{% endhint %}

### Filtering Reports

Use the static helper methods to filter and analyze batch results:

```php
// Filter successful deliveries
$successful = StatusReport::filterSuccessful($reports);

// Filter failed deliveries
$failed = StatusReport::filterFailed($reports);

// Filter expired subscriptions
$expired = StatusReport::filterExpired($reports);

// Filter retryable errors
$retryable = StatusReport::filterRetryable($reports);

// Get statistics
$stats = StatusReport::getStatistics($reports);
// Returns: ['total' => 100, 'successful' => 85, 'failed' => 15, 'expired' => 5, 'retryable' => 3]
```

### Complete Batch Processing Example

```php
use WebPush\Notification;
use WebPush\WebPushService;
use WebPush\StatusReport;

function sendToMultipleSubscriptions(
    WebPushService $webPush,
    Notification $notification,
    array $subscriptions,
    SubscriptionRepository $repository,
    LoggerInterface $logger
): array {
    // Send to all subscriptions
    $reports = $webPush->sendToMultiple($notification, $subscriptions);

    // Get statistics
    $stats = StatusReport::getStatistics($reports);
    $logger->info('Batch send completed', $stats);

    // Remove expired subscriptions
    $expired = StatusReport::filterExpired($reports);
    foreach ($expired as $report) {
        $repository->remove($report->getSubscription());
    }

    // Queue retryable errors
    $retryable = StatusReport::filterRetryable($reports);
    foreach ($retryable as $report) {
        $queue->retry($report->getNotification(), $report->getSubscription());
    }

    // Log permanent failures
    $failed = StatusReport::filterFailed($reports);
    foreach ($failed as $report) {
        if (!$report->isRetryable() && !$report->isSubscriptionExpired()) {
            $logger->error('Permanent failure', [
                'endpoint' => $report->getSubscription()->getEndpoint(),
                'error' => $report->getErrorMessage(),
                'code' => $report->getStatusCode()
            ]);
        }
    }

    return $stats;
}
```

## Monitoring and Metrics

Track your notification delivery metrics:

```php
class NotificationMetrics
{
    public function __construct(
        private MetricsCollector $metrics
    ) {}

    public function recordDelivery(StatusReport $report, Subscription $subscription): void
    {
        // Record success/failure
        if ($report->isSuccess()) {
            $this->metrics->increment('notifications.sent.success');
        } else {
            $this->metrics->increment('notifications.sent.failed');
            $this->metrics->increment("notifications.sent.failed.{$report->getStatusCode()}");
        }

        // Record expired subscriptions
        if ($report->isSubscriptionExpired()) {
            $this->metrics->increment('subscriptions.expired');
        }

        // Record by push service
        $endpoint = $subscription->getEndpoint();
        if (str_contains($endpoint, 'fcm.googleapis.com')) {
            $this->metrics->increment('notifications.sent.fcm');
        } elseif (str_contains($endpoint, 'mozilla.com')) {
            $this->metrics->increment('notifications.sent.mozilla');
        } elseif (str_contains($endpoint, 'apple.com')) {
            $this->metrics->increment('notifications.sent.apple');
        }
    }
}
```

## Debugging Failed Deliveries

When debugging failures, collect all relevant information:

```php
function debugFailedDelivery(StatusReport $report, Notification $notification, Subscription $subscription): void
{
    $debugInfo = [
        'status_code' => $report->getStatusCode(),
        'location' => $report->getLocation(),
        'links' => $report->getLinks(),
        'endpoint' => $subscription->getEndpoint(),
        'supported_encodings' => $subscription->getSupportedContentEncodings(),
        'payload_size' => strlen($notification->getPayload() ?? ''),
        'ttl' => $notification->getTTL(),
        'urgency' => $notification->getUrgency(),
        'topic' => $notification->getTopic(),
    ];

    error_log('Failed notification delivery: ' . json_encode($debugInfo, JSON_PRETTY_PRINT));
}
```

## Best Practices

1. **Always Check Success**: Don't assume notifications are delivered
2. **Handle Expired Subscriptions**: Clean up your database immediately
3. **Implement Retry Logic**: Use exponential backoff for server errors
4. **Log Everything**: Track success rates and error patterns
5. **Monitor Metrics**: Keep track of delivery rates by push service
6. **Respect Rate Limits**: Implement throttling to avoid 429 errors
7. **Validate Early**: Check subscription format before sending
8. **Test Thoroughly**: Test with real subscriptions from different browsers

## Common Issues and Solutions

### High Failure Rate
- Check VAPID keys are correctly configured
- Verify payload encryption is working
- Ensure subscriptions are fresh and valid

### Subscriptions Expiring Quickly
- User might be clearing browser data frequently
- Check if you're using the correct public key
- Verify the subscription format is correct

### Random Failures
- Implement retry logic with exponential backoff
- Monitor push service status pages
- Check for network connectivity issues

## Next Steps

- Learn about [Notifications](the-notification.md) structure and options
- Understand [Subscriptions](the-subscription.md) lifecycle
- Set up [VAPID](vapid.md) authentication properly
