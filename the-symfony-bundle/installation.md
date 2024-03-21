# Installation

The bundle can be installed using the package `spomky-labs/web-push-bundle`

```bash
composer require spomky-labs/web-push-bundle
```

If you use Symfony Flex, the bundle is ready to be used. Otherwise, you must enable it. The bundle class is `WebPush\Bundle\WebPushBundle`.

When done, the bundle is ready and can send the notifications. However, there are extra packages we highly recommend to install and set up.

## VAPID Header

The [VAPID header](../common-concepts/vapid.md) authenticates your server and prevent malicious application to send notifications to your users. The header contains a signed JSON Web Token (JWS).

The library provides bridges for the following libraries `web-token` and `lcobucci/jwt`.

Please install `web-token/jwt-signature-algorithm-ecdsa` or `lcobucci/jwt` depending on the library you want to use.

{% hint style="info" %}
It is possible to use any other JWS provider. This will be detailed in the future.
{% endhint %}
