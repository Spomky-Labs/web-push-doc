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
Please note that the TTL Extension is usually required by Push Services. To avoid any trouble, please use all extensions.
{% endhint %}

## Payload Extension

The payload extension allows Notifications to have a payload. This extension requires Content Encoding objects that will be responsible for the payload encryption.

The library provides the `AESGCM` and `AES128GCM` content encoding. These encodings are normally supported by all Push Services. The library is able to support any future encoding if deemed necessary.

Both content encodings require a PSR-20 Clock implementation. You can use `symfony/clock` for example.

```php
use Psr\Clock\ClockInterface;
use Symfony\Component\Clock\NativeClock;
use WebPush\Payload\AES128GCM;
use WebPush\Payload\AESGCM;
use WebPush\Payload\PayloadExtension;

$clock = new NativeClock(); // PSR-20 clock implementation

$payloadExtension = PayloadExtension::create()
    ->addContentEncoding(AESGCM::create($clock))
    ->addContentEncoding(AES128GCM::create($clock))
;

$extensionManager = ExtensionManager::create()
    ->add($payloadExtension)
;
```

## VAPID Extension

The [VAPID header](../common-concepts/vapid.md) authenticates your server and prevents malicious applications from sending notifications to your users. The header contains a signed JSON Web Token (JWS).

The library provides bridges for the following libraries `web-token` and `lcobucci/jwt`.

Please install `web-token/jwt-library` or `lcobucci/jwt` depending on the library you want to use.

The VAPID extension requires a PSR-20 Clock implementation. You can use `symfony/clock` for example.

```php
use Psr\Clock\ClockInterface;
use Symfony\Component\Clock\NativeClock;
use WebPush\VAPID\LcobucciProvider;
use WebPush\VAPID\VAPIDExtension;
use WebPush\VAPID\WebTokenProvider;

$clock = new NativeClock(); // PSR-20 clock implementation

// Web-Token
$jwsProvider = WebTokenProvider::create(
    'BB4W1qfBi7MF_Lnrc6i2oL-glAuKF4kevy9T0k2vyKV4qvuBrN3T6o9-7-NR3mKHwzDXzD3fe7XvIqIU1iADpGQ', // Public key
    'C40jLFSa5UWxstkFvdwzT3eHONE2FIJSEsVIncSCAqU' // Private key
);

// OR lcobucci/jwt
$jwsProvider = LcobucciProvider::create(
    'BB4W1qfBi7MF_Lnrc6i2oL-glAuKF4kevy9T0k2vyKV4qvuBrN3T6o9-7-NR3mKHwzDXzD3fe7XvIqIU1iADpGQ',
    'C40jLFSa5UWxstkFvdwzT3eHONE2FIJSEsVIncSCAqU'
);

$extensionManager = ExtensionManager::create()
    ->add(VAPIDExtension::create('http://my-service.com', $jwsProvider, $clock))
;
```

{% hint style="danger" %}
The public key used with your server shall be the same as the one in your Javascript application.
{% endhint %}

{% hint style="warning" %}
If this public/private key changes, subscriptions will become invalid.
{% endhint %}
