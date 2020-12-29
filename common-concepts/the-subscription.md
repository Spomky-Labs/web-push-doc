# The Subscription

The subscription is created on client side when the end-user allows your application to send push messages.

On client side \(Javascript\), you can simply send to your server the object you receive using `JSON.stringify`.

A subscription object will look like:

```javascript
{
 "endpoint":"https://updates.push.services.mozilla.com/wpush/v2/AAAAAAAA[…]AAAAAAAAA",
 "keys":{
 "auth":"XXXXXXXXXXXXXX",
 "p256dh":"YYYYYYYY[…]YYYYYYYYYYYYY"
 }
}
```

On server side, you can get a `WebPush\Subscription` object from the JSON string using the dedicated method `WebPush\Subscription::createFromString`.

```php
<?php

use WebPush\Subscription;

$subscription = Subscription::createFromString('{"endpoint":"https://updates.push.services.mozilla.com/wpush/v2/AAAAAAAA[…]AAAAAAAAA","keys":{"auth":"XXXXXXXXXXXXXX","p256dh":"YYYYYYYY[…]YYYYYYYYYYYYY"}}');
```

## Supported Content Encodings

By default, the content encoding `aesgcm` will be used. This encoding indicates how the payload of the notification should be encoded. The PushManager object from the Push API may list all acceptable encodings. In this case, it could be interesting to set these encodings to the Notification object.

```javascript
// Retreive the supported content encodings
const supportedContentEncodings = PushManager.supportedContentEncodings || ['aesgcm'];

// Assign the encodings to the subscription object
const jsonSubscription = Object.assign(
    subscription.toJSON(),
    { supportedContentEncodings }
);

// Send the subscription object to the application server
fetch('/notification/add', {
    method: 'POST',
    body: JSON.stringify(jsonSubscription ),
});
```

