# Doctrine

The bundle provides new Doctrine type and Schema to simplify the way you store the `Subscription` objects with Doctrine.

## Using The Doctrine Mapping

{% hint style="danger" %}
This method does not work with MySQL as the mapping uses the reserved keyword `keys`. This will be fixed for `v2.0` \(with breaking changes\).
{% endhint %}

### Configuration

To enable this feature, the following configuration option  shall be set:

```yaml
webpush:
    doctrine_mapping: true
```

This will tell the bundle to register the Subscription object as a Doctrine mapped-superclass. The DoctrineBundle shall be enabled. No additional configuration is required.

### The `Subscription` Entity

First of all, we need to create a Subscription Entity that extends the Subscription object. In this example, we also need to associate one or more Subscription entities to a specific user \(Many To One relationship\).

{% code title="src/Entity/Subscription.php" %}
```php
<?php

declare(strict_types=1);

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use WebPush\Subscription as WebPushSubscription;

/**
 * @ORM\Table(name="subscriptions")
 * @ORM\Entity
 */
class Subscription extends WebPushSubscription
{
    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private ?int $id = null;
    
    /**
     * @ORM\ManyToOne(targetEntity="User", inversedBy="subscriptions", cascade={"persist"})
     * @ORM\JoinColumn(name="user_id", referencedColumnName="id", nullable="true")
     */
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
        $base = BaseSubscription::createFromString($input);
        $object = new self($base->getEndpoint());
        $object->withContentEncodings($base->getSupportedContentEncodings());
        foreach ($base->getKeys()->all() as $k => $v) {
            $object->getKeys()->set($k, $v);
        }

        return $object;
    }
}
```
{% endcode %}

{% hint style="info" %}
In this exaple, we assume you already have a valid User entity class.
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
class User //Usual interface here
{
    //Usual user stuff here

    /**
     * @ORM\OneToMany(targetEntity="Subscription", mappedBy="user")
     */
    private Collection $subscriptions;
    
    public function __construct()
    {
        $this->notifications = new ArrayCollection();
    }

    /**
     * @return Notification[]
     */
    public function getSubscriptions(): array
    {
        return $this->notifications->toArray();
    }

    public function addSubscription(Subscription $subscription): self
    {
        $subscription->setUser($this);
        $this->subscriptions->add($subscription);

        return $this;
    }

    public function removeSubscription(Subscription $subscription): self
    {
        $child->setUser(null);
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
    if ($report->subscriptionExpired()) {
        //...Remove this subscription
    }
}
```
{% endcode %}

## Using Doctrine Type

The bundle auto-register the Doctrine type webpush\_subscription. The Subscription object will automatically be converted.

{% code title="src/Entity/Subscription.php" %}
```php
<?php

declare(strict_types=1);

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use WebPush\Subscription as WebPushSubscription;

/**
 * @ORM\Table(name="subscriptions")
 * @ORM\Entity
 */
class Subscription
{
    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private ?int $id = null;
    
    /**
     * @ORM\ManyToOne(targetEntity="User", inversedBy="subscriptions", cascade={"persist"})
     * @ORM\JoinColumn(name="user_id", referencedColumnName="id", nullable="true")
     */
    private ?User $user;
    
    /**
     * @ORM\Column(type="webpush_subscription")
     */
    private WebPushSubscription $subscription;

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

    public function getSubscription(): WebPushSubscription
    {
        return $this->subscription;
    }
}
```
{% endcode %}

## Using Your Own Entity Class

{% hint style="info" %}
New in v1.1!
{% endhint %}

Starting with v1.1, it is possible to use your own Subscription entity class. The only constraint is that it shall implement the interface `WebPush\SubscriptionInterface` or shall have a method that returns an object that implements this interface.

```php
<?php

declare(strict_types=1);

namespace App\Entity;

use function array_key_exists;
use Assert\Assertion;
use DateTimeInterface;
use Safe\DateTimeImmutable;
use function Safe\json_decode;

class Subscription implements SubscriptionInterface
{
    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private ?int $id = null;
    
    /**
     * @ORM\Column(type="integer")
     */
    private string $endpoint;

    /**
     * @ORM\Column(type="array")
     *
     * @var string[]
     */
    private array $keys = [];

    /**
     * @ORM\Column(type="array")
     *
     * @var string[]
     */
    private array $supportedContentEncodings = ['aesgcm'];

    /**
     * @ORM\Column(type="integer", nullable=true)
     */
    private ?int $expirationTime = null;

    public function __construct(string $endpoint)
    {
        $this->endpoint = $endpoint;
    }

    /**
     * @param string[] $contentEncodings
     */
    public function setContentEncodings(array $contentEncodings): self
    {
        $this->supportedContentEncodings = $contentEncodings;

        return $this;
    }

    public function getKeys(): array
    {
        return $this->keys;
    }

    public function hasKey(string $key): bool
    {
        return isset($this->keys[$key]);
    }

    /**
     * @return array<string, string>
     */
    public function getKey(string $key): string
    {
        Assertion::keyExists($this->keys, $key, 'The key does not exist');

        return $this->keys[$key];
    }

    public function setKeys(array $keys): self
    {
        $this->keys = $keys;

        return $this;
    }

    public function getExpirationTime(): ?int
    {
        return $this->expirationTime;
    }

    public function setExpirationTime(?int $expirationTime): self
    {
        $this->expirationTime = $expirationTime;

        return $this;
    }

    public function getEndpoint(): string
    {
        return $this->endpoint;
    }

    /**
     * @return string[]
     */
    public function getSupportedContentEncodings(): array
    {
        return $this->supportedContentEncodings;
    }

    /**
     * @return array<string, string|string[]>
     */
    public function jsonSerialize(): array
    {
        return [
            'endpoint' => $this->endpoint,
            'supportedContentEncodings' => $this->supportedContentEncodings,
            'keys' => $this->keys,
        ];
    }
}
```

