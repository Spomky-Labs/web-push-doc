# What’s New in v1.1?

## The Library

### New Interfaces

The following interfaces have been introduced:

* `WebPush\NotificationInterface`
* `WebPush\SubscriptionInterface`
* `WebPush\StatusReportInterface`

It is no longer required to use the concrete classes. You can use your own objects to be transported and sent.

Methods provided by the WebPush service, the extensions or the extension manager have been updated to always use the interfaces instead of the classes.

### Changes and Deprecations

#### Status Report

The methods `getResponse` and `getRequest` of the class `WebPush\StatusReport` are now deprecated. They should not used anymore and will be removed in `v2.0`.

The method `notificationExpired` of the class `WebPush\StatusReport` is deprecated. Please use `isSubscriptionExpired` instead.

#### Message

The constructor and method `create` of the class `Message` gets a new parameter for the message title. In addition to this change, the body can be `null`.

```php
$message = Message::create('This is the body');
$message = Message::create('This is the title', 'and this is the body');
$message = Message::create('This is the title', null); // and the body is null
```

This class also gets a new parameter to render the JSON data with a structure similar to [the one shown on this page](https://developer.mozilla.org/en-US/docs/Web/API/Notification/Notification#parameters). The title shall be set. By default, the previous data structure is used and this parameter is optional for the branch `1.x`, but will be mandatory for `2.x`.

```php
// Before
$message = Message::create('This is the title', 'Hello World!')
    ->mute()
    ->unmute()

    ->auto()
    ->ltr()
    ->rtl()

    ->addAction(Action::create('alert', 'Click me!'))

    ->interactionRequired()
    ->noInteraction()

    ->renotify()
    ->doNotRenotify()

    ->withBody('Hello World!')

    ->withIcon('https://…')
    ->withImage('https://…')
    ->withData(['foo' => 'BAR'])
    ->withBadge('badge1')
    ->withLang('fr-FR')
    ->withTimestamp(time())
    ->withTag('foo')

    ->vibrate(300, 100, 400)

    ->toString()
;
//will render 
{

    "title":"This is the title",
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

// AFTER
$message = Message::create('This is the title', 'Hello World!', true)
    ... // same as above
    ->toString()
;
//will render 
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

### The Symfony Bundle

### New Service

The service `web_push.service` has been added. This new service uses the same interface as `WebPush\WebPush`. The main difference is that this service uses the Symfony Http Client and allow you to take advantages of the asynchronous request calls.

With `v2.0`, the service `WebPush::class` will point to the new one if the Symfony Http Client is available.

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

 

