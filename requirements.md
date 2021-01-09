# Requirements

## Mandatory

* PHP 7.4 or 8.0+
* A PSR-17 \(HTTP Message Factory\) implementation
* A PSR-18 \(HTTP Client\) implementation
* The `JSON` extension

## Optional

* A PSR-3 \(Logger Interface\) implementation for debugging

## Extension Specific

### VAPID extension

* Required:
  * `openssl` extension
  * `mbstring` extension
  * A JWT Provider
* Optional:
  * A PSR-3 \(Logger Interface\) implementation for debugging

{% hint style="success" %}
This library provides JWT Provider implementations for [web-token](https://web-token.spomky-labs.com) and [lcobucci/jwt](https://github.com/lcobucci/jwt)
{% endhint %}

### Payload extension

* Required:
  * `openssl` extension
  * `mbstring` extension
* Optional:
  * A PSR-6 \(Caching Interface\) implementation
  * A PSR-3 \(Logger Interface\) implementation for debugging

