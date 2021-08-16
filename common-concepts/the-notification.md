# The Notification

To reach the client \(web browser\), you need to send a Notification to the Subscription.

```php
<?php
use WebPush\Notification;

$notification = Notification::create();
```

The Notification should have a payload. In this case, the payload will be encrypted on server side and decrypted by the client.

That payload may be a string or a JSON object. The structure of the latter is described in the next section.

```php
<?php
use WebPush\Notification;

$notification = Notification::create()
    ->withPayload('Hello world')
;
```

## TTL \(Time-To-Live\)

With this feature, a value in seconds is added to the notification. It suggests how long a push message is retained by the push service. A value of 0 \(zero\) indicates the notification is delivered immediately.

```php
<?php
use WebPush\Notification;

$notification = Notification::create()
    ->withTTL(3600)
;
```

## Topic

A push message that has been stored by the push service can be replaced with new content. If the user agent is offline during the time the push messages are sent, updating a push message avoids the situation where outdated or redundant messages are sent to the user agent.

Only push messages that have been assigned a topic can be replaced. A push message with a topic replaces any outstanding push message with an identical topic.

```php
<?php
use WebPush\Notification;

$notification = Notification::create()
    ->withTopic('user-account-updated')
;
```

## Urgency

For a device that is battery-powered, it is often critical it remains dormant for extended periods.

Radio communication in particular consumes significant power and limits the length of time the device can operate.

To avoid consuming resources to receive trivial messages, it is helpful if an application server can communicate the urgency of a message and if the user agent can request that the push server only forwards messages of a specific urgency.

| Urgency | Device State | Examples |
| :--- | :--- | :--- |
| very-low | On power and Wi-Fi | Advertisements |
| low | On either power or Wi-Fi | Topic updates |
| normal | On neither power nor Wi-Fi | Chat or Calendar Message |
| high | Low battery | Incoming phone call or time-sensitive alert |

{% hint style="warning" %}
Be carful with the `very-low` urgency: it is not recognized by all Web-Push services
{% endhint %}

```php
<?php
use WebPush\Notification;

$notification = Notification::create()
    ->veryLowUrgency()
    ->lowUrgency()
    ->normalUrgency()
    ->highUrgency()
;
```

## Asynchronous Response

Your application may prefer asynchronous responses to request confirmation from the push service when a push message is delivered and then acknowledged by the user agent. The push service MUST support delivery confirmations to use this feature.

```php
<?php
use WebPush\Notification;

$notification = Notification::create()
    ->async() // Prefer async response
    ->sync() // Prefer sync response (default)
;
```

{% hint style="warning" %}
The `async` mode is not recognised by all Web Push services. In case of failure, you should try sending `sync`notifications.
{% endhint %}

## JSON Messages

As mentioned in the overview section, the specification [defines a structure for the payload](https://developer.mozilla.org/en-US/docs/Web/API/Notification/Notification#parameters). This structure contains properties that the client should be understood and render an appropriate way.

The library provides a `WebPush\Message` class with convenient methods to ease the creation of a message.

```php
<?php
use WebPush\Action;
use WebPush\Message;
use WebPush\Notification;

$message = Message::create('This is the title', null, true)
    ->mute() // Silent
    ->unmute() // Not silent (default)

    ->auto() //Direction = auto (default)
    ->ltr() //Direction = left to right
    ->rtl() //Direction = right to left

    ->addAction(Action::create('alert', 'Click me!'))

    ->interactionRequired()
    ->noInteraction()

    ->renotify()
    ->doNotRenotify() // Default
    
    ->withBody('Hello World!')

    ->withIcon('https://…')
    ->withImage('https://…')
    ->withData(['foo' => 'BAR']) // Arbitrary data
    ->withBadge('badge1')
    ->withLang('fr-FR')
    ->withTimestamp(time())
    ->withTag('foo')

    ->vibrate(300, 100, 400)

    ->toString() // Converts the Message object into a string
;

$notification = Notification::create()
    ->withPayload($message)
;
```

{% hint style="info" %}
Please note that the second and the third parameters are needed for `v1.1+` branch but bill be removed in `v2.0`.
{% endhint %}

The resulting notification payload will look like as follow:

```javascript
{
    "title":"This is the title",
    "options":{
        "actions":[
            {
                "action":"alert",
                "title":"Click me!"
            }
        ],
        "badge":"badge1",
        "body":"Hello World!",
        "data":{
            "foo":"BAR"
        },
        "dir":"rtl",
        "icon":"https://…",
        "image":"https://…",
        "l ang":"fr-FR",
        "renotify":false,
        "requireInteraction":false,
        "silent":false,
        "tag":"foo",
        "timestamp":1629145424,
        "vibrate":[
            300,
            100,
            400
        ]
    }
}
```

On client side, you can easily load that payload and display the notification:

```javascript
  const {title, options}  = payload;
  const notification = new Notification(title, options);
```

