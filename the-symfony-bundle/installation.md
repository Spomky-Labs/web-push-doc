# Installation

The bundle can be installed using the package `spomky-labs/web-push-lib`

In addition to this package, you must install the required dependencies that are namely:

* [A HTTP Client implementation](https://packagist.org/providers/psr/http-client-implementation)
* [A PSR7 Request Factory implementation](https://packagist.org/providers/psr/http-factory-implementation)

In the following example, we will install nyholm/prs7 and symfony/http-client.

```bash
composer install nyholm/psr7 symfony/http-client spomky-labs/web-push-lib
```

If you use Symfony Flex, the bundle is ready to be used. Otherwise, you must enable it. The bundle class is `WebPush\Bundle\WebPushBundle`.

When done, the bundle is ready and can send the notifications. However, there are extra packages we highly recommend to install and set up.

The [VAPID header](../common-concepts/vapid.md) authenticates your server and prevent malicious application to send notifications to your users. The header contains a signed JSON Web Token \(JWS\).

At the time of writing, the library provides bridges for the following libraries `web-token` and `lcobucci/jwt`.

Please install `web-token/jwt-signature-algorithm-ecdsa` or `lcobucci/jwt` depending on the library you want to use.

{% hint style="info" %}
It is possible to use any other JWS provider. This will be detailed in the future.
{% endhint %}

