# The Extension Manager

The Web Push service requires an Extension Manager. This object manages extensions that will manipulate the request before sending it to the Push Service.

In the example below, we add all basic extensions.

```php
use WebPush\ExtensionManager;
use WebPush\PreferAsyncExtension;
use WebPush\TopicExtension;
use WebPush\TTLExtension;
use WebPush\UrgencyExtension;

$extensionManager = ExtensionManager::create()
    ->add(TTLExtension::create())
    ->add(UrgencyExtension::create())
    ->add(TopicExtension::create())
    ->add(PreferAsyncExtension::create())
;
```

{% hint style="info" %}
Please note that the TTL Extension is usually required by Push Services
{% endhint %}

## Payload Extension

The payload extension allows Notifications to have a payload. This extension requires Content Encoding objects that will be responsible of the payload encryption.

The library provides the `AESGCM` and `AES128GCM` content encoding. These encodings are normally supported by all Push Services. The library is able to support any future encoding is deemed necessary.

```php
$payloadExtension = PayloadExtension::create()
    ->addContentEncoding(AESGCM::create())
    ->addContentEncoding(AES128GCM::create())
;

$extensionManager = ExtensionManager::create()
    ->add($payloadExtension )
;
```

## VAPID Extension

The [VAPID header](../common-concepts/vapid.md) authenticates your server and prevent malicious application to send notifications to your users. The header contains a signed JSON Web Token \(JWS\).

The library provides bridges for the following libraries `web-token` and `lcobucci/jwt`.

Please install `web-token/jwt-signature-algorithm-ecdsa` or `lcobucci/jwt` depending on the library you want to use.

```php
// Web-Token
$jwsProvider = WebTokenProvider::create(
    'BB4W1qfBi7MF_Lnrc6i2oL-glAuKF4kevy9T0k2vyKV4qvuBrN3T6o9-7-NR3mKHwzDXzD3fe7XvIqIU1iADpGQ',
    'C40jLFSa5UWxstkFvdwzT3eHONE2FIJSEsVIncSCAqU'
);

// lcobucci/jwt
$jwsProvider = WebTokenProvider::create(
    'BB4W1qfBi7MF_Lnrc6i2oL-glAuKF4kevy9T0k2vyKV4qvuBrN3T6o9-7-NR3mKHwzDXzD3fe7XvIqIU1iADpGQ',
    'C40jLFSa5UWxstkFvdwzT3eHONE2FIJSEsVIncSCAqU'
);

$extensionManager = ExtensionManager::create()
    ->add(VAPIDExtension::create('http://my-service.com', $jwsProvider)
);
```

