# Exceptions

The Web Push library uses a hierarchy of specific exceptions to provide clear error messages and enable precise error handling.

## Exception Hierarchy

```
WebPushException (interface)
└── AbstractWebPushException
    ├── ValidationException
    │   ├── InvalidTopicException
    │   ├── InvalidTTLException
    │   ├── InvalidUrgencyException
    │   └── InvalidPayloadException
    └── OperationException (deprecated, use specific exceptions)
```

## Validation Exceptions

All validation exceptions extend `ValidationException` and expose the problematic value as a `public readonly` property, making debugging easier.

### InvalidTopicException

Thrown when a notification topic is invalid.

**Property:** `public readonly string $topic`

**Causes:**
- Empty topic
- Topic exceeds 32 characters
- Topic contains invalid characters (only `a-z`, `A-Z`, `0-9`, `-`, `.`, `_`, `~` allowed)

```php
use WebPush\Exception\InvalidTopicException;
use WebPush\Notification;

try {
    $notification = Notification::create()
        ->withTopic('invalid@topic'); // @ is not allowed
} catch (InvalidTopicException $e) {
    echo "Invalid topic: {$e->topic}\n";
    echo "Error: {$e->getMessage()}\n";
    // Output:
    // Invalid topic: invalid@topic
    // Error: Topic must contain only URL-safe characters (a-z, A-Z, 0-9, -, ., _, ~)
}
```

### InvalidTTLException

Thrown when a notification TTL (Time-To-Live) is invalid.

**Property:** `public readonly int $ttl`

**Causes:**
- Negative TTL value

```php
use WebPush\Exception\InvalidTTLException;
use WebPush\Notification;

try {
    $notification = Notification::create()
        ->withTTL(-1); // TTL cannot be negative
} catch (InvalidTTLException $e) {
    echo "Invalid TTL: {$e->ttl}\n";
    echo "Error: {$e->getMessage()}\n";
    // Output:
    // Invalid TTL: -1
    // Error: Invalid TTL
}
```

### InvalidUrgencyException

Thrown when a notification urgency is invalid.

**Property:** `public readonly string $urgency`

**Causes:**
- Urgency value not in: `very-low`, `low`, `normal`, `high`

```php
use WebPush\Exception\InvalidUrgencyException;
use WebPush\Notification;

try {
    $notification = Notification::create()
        ->withUrgency('urgent'); // Invalid urgency
} catch (InvalidUrgencyException $e) {
    echo "Invalid urgency: {$e->urgency}\n";
    echo "Error: {$e->getMessage()}\n";
    // Output:
    // Invalid urgency: urgent
    // Error: Invalid urgency parameter
}
```

### InvalidPayloadException

Thrown when a notification payload is invalid.

**Property:** `public readonly int $size`

**Causes:**
- Payload exceeds maximum size

```php
use WebPush\Exception\InvalidPayloadException;
use WebPush\Notification;

try {
    $notification = Notification::create()
        ->withPayload($hugePayload); // Payload too large
} catch (InvalidPayloadException $e) {
    echo "Payload size: {$e->size} bytes\n";
    echo "Error: {$e->getMessage()}\n";
}
```

## Error Handling Strategies

### Strategy 1: Catch Specific Exceptions

Handle each type of validation error differently:

```php
use WebPush\Exception\InvalidTopicException;
use WebPush\Exception\InvalidTTLException;
use WebPush\Exception\InvalidUrgencyException;
use WebPush\Notification;

function createNotification(array $data): Notification
{
    try {
        $notification = Notification::create();

        if (isset($data['topic'])) {
            $notification->withTopic($data['topic']);
        }

        if (isset($data['ttl'])) {
            $notification->withTTL($data['ttl']);
        }

        if (isset($data['urgency'])) {
            $notification->withUrgency($data['urgency']);
        }

        return $notification;

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

### Strategy 2: Catch All Validation Errors

Use the base `ValidationException` to catch any validation error:

```php
use WebPush\Exception\ValidationException;
use WebPush\Notification;
use Psr\Log\LoggerInterface;

class NotificationFactory
{
    public function __construct(
        private LoggerInterface $logger
    ) {}

    public function createFromUserInput(array $input): ?Notification
    {
        try {
            return Notification::create()
                ->withTopic($input['topic'] ?? 'default')
                ->withTTL($input['ttl'] ?? Notification::TTL_ONE_HOUR)
                ->withUrgency($input['urgency'] ?? Notification::URGENCY_NORMAL)
                ->withPayload($input['message'] ?? '');

        } catch (ValidationException $e) {
            $this->logger->error('Notification validation failed', [
                'error' => $e->getMessage(),
                'exception' => get_class($e)
            ]);
            return null;
        }
    }
}
```

### Strategy 3: Provide User Feedback

Use exception properties to give specific feedback to users:

```php
use WebPush\Exception\InvalidTopicException;
use WebPush\Exception\ValidationException;
use WebPush\Notification;

class NotificationController
{
    public function create(array $request): array
    {
        try {
            $notification = Notification::create()
                ->withTopic($request['topic'])
                ->withPayload($request['message']);

            return ['success' => true];

        } catch (InvalidTopicException $e) {
            return [
                'success' => false,
                'field' => 'topic',
                'value' => $e->topic,
                'message' => match (true) {
                    strlen($e->topic) > 32 => "Topic is too long (max 32 characters)",
                    $e->topic === '' => "Topic cannot be empty",
                    default => "Topic contains invalid characters (only a-z, A-Z, 0-9, -, ., _, ~ allowed)"
                }
            ];
        } catch (ValidationException $e) {
            return [
                'success' => false,
                'message' => $e->getMessage()
            ];
        }
    }
}
```

### Strategy 4: Logging with Context

Log validation errors with full context for debugging:

```php
use WebPush\Exception\InvalidTopicException;
use WebPush\Exception\InvalidTTLException;
use WebPush\Exception\ValidationException;
use Psr\Log\LoggerInterface;

class NotificationLogger
{
    public function __construct(
        private LoggerInterface $logger
    ) {}

    public function logValidationError(ValidationException $e): void
    {
        $context = [
            'exception' => get_class($e),
            'message' => $e->getMessage()
        ];

        // Add specific context based on exception type
        if ($e instanceof InvalidTopicException) {
            $context['topic'] = $e->topic;
            $context['topic_length'] = strlen($e->topic);
        } elseif ($e instanceof InvalidTTLException) {
            $context['ttl'] = $e->ttl;
        }

        $this->logger->warning('Notification validation failed', $context);
    }
}
```

## Best Practices

### 1. Catch Specific Exceptions

Prefer catching specific exceptions over the generic `ValidationException` when you need different handling:

```php
// ✅ Good - specific handling
try {
    $notification->withTopic($userInput);
} catch (InvalidTopicException $e) {
    return "Please use only letters, numbers, and dashes in the topic";
}

// ❌ Less ideal - generic handling
try {
    $notification->withTopic($userInput);
} catch (ValidationException $e) {
    return $e->getMessage(); // Generic error message
}
```

### 2. Use Exception Properties

Leverage the `public readonly` properties for debugging and logging:

```php
// ✅ Good - use the property
catch (InvalidTopicException $e) {
    $logger->error('Invalid topic', [
        'topic' => $e->topic,
        'length' => strlen($e->topic)
    ]);
}

// ❌ Less ideal - parse the message
catch (InvalidTopicException $e) {
    // Don't parse error messages!
    if (str_contains($e->getMessage(), 'too long')) {
        // ...
    }
}
```

### 3. Validate Early

Validate user input as early as possible:

```php
// ✅ Good - validate before processing
public function scheduleNotification(array $data): void
{
    try {
        $notification = Notification::create()
            ->withTopic($data['topic'])
            ->withTTL($data['ttl']);
    } catch (ValidationException $e) {
        throw new InvalidRequestException($e->getMessage());
    }

    // Continue with valid notification
    $this->queue->push($notification);
}
```

### 4. Provide Helpful Error Messages

Transform technical exceptions into user-friendly messages:

```php
public function createNotificationFromForm(array $form): Notification
{
    try {
        return Notification::create()
            ->withTopic($form['topic'])
            ->withTTL((int) $form['ttl']);

    } catch (InvalidTopicException $e) {
        throw new \DomainException(
            "The topic '{$e->topic}' is invalid. " .
            "Topics can only contain letters, numbers, and these characters: - . _ ~"
        );
    } catch (InvalidTTLException $e) {
        throw new \DomainException(
            "The TTL must be a positive number (you provided: {$e->ttl})"
        );
    }
}
```

## Common Scenarios

### Form Validation

```php
class NotificationFormValidator
{
    public function validate(array $form): array
    {
        $errors = [];

        // Validate topic
        if (!empty($form['topic'])) {
            try {
                Notification::create()->withTopic($form['topic']);
            } catch (InvalidTopicException $e) {
                $errors['topic'] = match (true) {
                    strlen($e->topic) > 32 => 'Topic must be 32 characters or less',
                    !preg_match('/^[a-zA-Z0-9\-._~]+$/', $e->topic) =>
                        'Topic can only contain letters, numbers, and: - . _ ~',
                    default => 'Invalid topic'
                };
            }
        }

        // Validate TTL
        if (isset($form['ttl'])) {
            try {
                Notification::create()->withTTL((int) $form['ttl']);
            } catch (InvalidTTLException $e) {
                $errors['ttl'] = 'TTL must be a positive number';
            }
        }

        return $errors;
    }
}
```

### API Error Responses

```php
use Symfony\Component\HttpFoundation\JsonResponse;
use WebPush\Exception\ValidationException;

class NotificationApiController
{
    public function create(Request $request): JsonResponse
    {
        try {
            $notification = Notification::create()
                ->withTopic($request->get('topic'))
                ->withTTL($request->get('ttl'))
                ->withPayload($request->get('message'));

            // Send notification...

            return new JsonResponse(['success' => true]);

        } catch (ValidationException $e) {
            return new JsonResponse([
                'success' => false,
                'error' => [
                    'type' => basename(str_replace('\\', '/', get_class($e))),
                    'message' => $e->getMessage()
                ]
            ], 400);
        }
    }
}
```

### Testing

```php
use PHPUnit\Framework\TestCase;
use WebPush\Exception\InvalidTopicException;
use WebPush\Notification;

class NotificationTest extends TestCase
{
    public function testTopicValidation(): void
    {
        $this->expectException(InvalidTopicException::class);

        $notification = Notification::create()
            ->withTopic('invalid@topic');
    }

    public function testTopicExceptionContainsValue(): void
    {
        try {
            Notification::create()->withTopic('bad@topic');
            $this->fail('Expected InvalidTopicException');
        } catch (InvalidTopicException $e) {
            $this->assertSame('bad@topic', $e->topic);
        }
    }
}
```

## Migration from OperationException

If your code currently catches `OperationException`, you can migrate gradually:

### Before (still works)

```php
use WebPush\Exception\OperationException;

try {
    $notification->withTopic($topic);
} catch (OperationException $e) {
    // Handle error
}
```

### After (recommended)

```php
use WebPush\Exception\InvalidTopicException;
use WebPush\Exception\ValidationException;

try {
    $notification->withTopic($topic);
} catch (InvalidTopicException $e) {
    // Handle topic-specific error with access to $e->topic
} catch (ValidationException $e) {
    // Handle any validation error
}
```

## Summary

✅ **Specific Exceptions** - Each validation error has its own exception type
✅ **Contextual Properties** - Access problematic values via `public readonly` properties
✅ **Type-Safe** - IDEs can autocomplete exception types and properties
✅ **Better Debugging** - Log exact values that caused errors
✅ **Granular Handling** - Catch and handle specific error types differently
✅ **User-Friendly** - Transform technical errors into helpful messages

## Next Steps

- Learn about [Notifications](the-notification.md) and their properties
- Understand [Status Reports](the-status-report.md) for delivery errors
- See [VAPID](vapid.md) authentication setup
