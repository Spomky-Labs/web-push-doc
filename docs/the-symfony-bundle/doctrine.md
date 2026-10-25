# Doctrine

This section explains how to store `Subscription` objects using Doctrine ORM in your Symfony application.

## Creating a Subscription Entity

To persist subscriptions in your database, you need to create a Doctrine entity. There are two approaches:

1. **Extend the base `WebPush\Subscription` class** (recommended for simplicity)
2. **Implement the `WebPush\SubscriptionInterface` interface** (more flexibility)

### Approach 1: Extending the Base Class

In this example, we create a Subscription entity that extends the base `WebPush\Subscription` class. We also associate one or more Subscription entities to a specific user (Many-To-One relationship).

{% code title="src/Entity/Subscription.php" %}
```php
<?php

declare(strict_types=1);

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use WebPush\Subscription as WebPushSubscription;

#[ORM\Table(name: 'subscriptions')]
#[ORM\Entity]
class Subscription extends WebPushSubscription
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    private ?int $id = null;

    #[ORM\ManyToOne(targetEntity: User::class, cascade: ['persist'], inversedBy: 'subscriptions')]
    #[ORM\JoinColumn(name: 'user_id', referencedColumnName: 'id', nullable: true)]
    
    private ?User $user;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getUser(): ?User
    {
        return $this->user;
    }

    public function setUser(?User $user): self
    {
        $this->user = $user;

        return $this;
    }

    // We need to override this method as it returns a WebPush\Subscription and we want an entity
    public static function createFromString(string $input): self
    {
        $base = parent::createFromString($input);
        $object = new self($base->getEndpoint());
        $object->withContentEncodings($base->getSupportedContentEncodings());
        foreach ($base->getKeys() as $k => $v) {
            $object->setKey($k, $v);
        }

        return $object;
    }
}
```
{% endcode %}

{% hint style="info" %}
In this example, we assume you already have a valid User entity class.
{% endhint %}

### The `User` Entity

Now, to have a bidirectional relationship between this class and the User entity class, we will add this relationship to the User class.

{% code title="src/Entity/User.php" %}
```php
<?php

declare(strict_types=1);

namespace App\Entity;

use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Table(name="users")
 * @ORM\Entity
 */
#[ORM\Table(name: 'users')]
#[ORM\Entity]
class User //Usual interface here
{
    //Usual user stuff here

    #[ORM\OneToMany(targetEntity: Subscription::class, mappedBy: 'user')]
    private Collection $subscriptions;

    public function __construct()
    {
        $this->subscriptions = new ArrayCollection();
    }

    /**
     * @return Subscription[]
     */
    public function getSubscriptions(): array
    {
        return $this->subscriptions->toArray();
    }

    public function addSubscription(Subscription $subscription): self
    {
        $subscription->setUser($this);
        $this->subscriptions->add($subscription);

        return $this;
    }

    public function removeSubscription(Subscription $subscription): self
    {
        $subscription->setUser(null);
        $this->subscriptions->removeElement($subscription);

        return $this;
    }
}
```
{% endcode %}

## Sending Notifications To A User

Now that your entities are set, you can register Subcriptions and assign them to your users. To send a Notification to a specific user, you just have to get all subscriptions using `$user->getSubscriptions()`.

{% code title="" %}
```php
$subscriptions = $user->getSubscriptions();
foreach ($subscriptions as $subscription) {
    $report = $this->webPush->send($notification, $subscription);
    if ($report->isSubscriptionExpired()) {
        //...Remove this subscription
    }
}
```
{% endcode %}

### Approach 2: Implementing the Interface Directly

Instead of extending the `WebPush\Subscription` class, you can create your own entity class that implements the `WebPush\SubscriptionInterface` interface. This approach gives you more flexibility in how you structure your entity.

```php
<?php

declare(strict_types=1);

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use WebPush\SubscriptionInterface;

#[ORM\Table(name: 'subscriptions')]
#[ORM\Entity]
class Subscription implements SubscriptionInterface
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    private ?int $id = null;

    #[ORM\Column(type: 'string')]
    private string $endpoint;

    #[ORM\Column(type: 'json')]
    private array $keys = [];

    #[ORM\Column(type: 'json')]
    private array $supportedContentEncodings = ['aesgcm'];

    #[ORM\Column(type: 'integer', nullable: true)]
    private ?int $expirationTime = null;

    #[ORM\ManyToOne(targetEntity: User::class, cascade: ['persist'], inversedBy: 'subscriptions')]
    #[ORM\JoinColumn(name: 'user_id', referencedColumnName: 'id', nullable: true)]
    private ?User $user;

    public function __construct(string $endpoint)
    {
        $this->endpoint = $endpoint;
    }

    // Implement all methods from SubscriptionInterface
    public function getEndpoint(): string
    {
        return $this->endpoint;
    }

    public function getKeys(): array
    {
        return $this->keys;
    }

    public function hasKey(string $key): bool
    {
        return isset($this->keys[$key]);
    }

    public function getKey(string $key): string
    {
        return $this->keys[$key] ?? throw new \RuntimeException('Key not found');
    }

    public function getSupportedContentEncodings(): array
    {
        return $this->supportedContentEncodings;
    }

    public function getExpirationTime(): ?int
    {
        return $this->expirationTime;
    }

    public function jsonSerialize(): array
    {
        return [
            'endpoint' => $this->endpoint,
            'keys' => $this->keys,
            'supportedContentEncodings' => $this->supportedContentEncodings,
        ];
    }

    // Additional methods for Doctrine
    public function getId(): ?int
    {
        return $this->id;
    }

    public function getUser(): ?User
    {
        return $this->user;
    }

    public function setUser(?User $user): self
    {
        $this->user = $user;
        return $this;
    }
}
```

Both approaches (extending the class or implementing the interface) are valid and can be used depending on your needs.
