# Doctrine

The bundle provides new Doctrine types to simplify the way you store the `Subscription` objects with Doctrine.

You can send notifications to any users, even if they are not registered in your application. But let say you want to be able to notify your users when they receive new messages on their dashborad \(e.g. an answer to a ticket\).

{% hint style="info" %}
In the example below, we consider you already have a working application with a `User` entity.
{% endhint %}

## Configuration

To enable this feature, the following configuration option  shall be set:

```yaml
webpush:
    doctrine_mapping: true
```

This will tell the bundle to register new Doctrine types and mappings. The DoctrineBundle shall be enabled. No additional configuration is required.

## The `Subscription` Entity

First of all, we need to create a Subscription Entity that extends the Subscription object. We also need to associate one or more Subscription entities to a specific user \(Many To One relationship\).

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
    public static function createFromBaseSubscription(string $input): self
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
We allow `$user` to be `null` because anonymous user could also want to receive push notifications
{% endhint %}

## The `User` Entity

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

{% hint style="info" %}
Don’t forget to update your database schema
{% endhint %}

