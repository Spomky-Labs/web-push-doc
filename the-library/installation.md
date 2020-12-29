# Installation

The library can be installed using the package `spomky-labs/web-push-lib`

In addition to this package, you must install the required dependencies that are namely:

* [A HTTP Client implementation](https://packagist.org/providers/psr/http-client-implementation)
* [A PSR7 Request Factory implementation](https://packagist.org/providers/psr/http-factory-implementation)

In the following example, we will install `nyholm/prs7` and `symfony/http-client`.

```bash
composer require nyholm/psr7 symfony/http-client spomky-labs/web-push-lib
```

## VAPID Header

The [VAPID header](../common-concepts/vapid.md) authenticates your server and prevent malicious application to send notifications to your users. The header contains a signed JSON Web Token \(JWS\).

The library provides bridges for the following libraries `web-token` and `lcobucci/jwt`.

Please install `web-token/jwt-signature-algorithm-ecdsa` or `lcobucci/jwt` depending on the library you want to use.

{% hint style="info" %}
It is possible to use any other JWS provider. This will be detailed in the future.
{% endhint %}

