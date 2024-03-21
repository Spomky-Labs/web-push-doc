# Requirements

## Mandatory

* PHP 8.2+
* The `JSON` extension

## Optional

* A PSR-3 (Logger Interface) implementation for debugging

## Extension Specific

### VAPID extension

* Required:
  * `openssl` extension
  * `mbstring` extension
  * A JWT Provider
  * A PSR-20 (Clock) implementation
* Optional:
  * A PSR-3 (Logger Interface) implementation for debugging

{% hint style="success" %}
This library provides JWT Provider implementations for [web-token](https://web-token.spomky-labs.com) and [lcobucci/jwt](https://github.com/lcobucci/jwt)
{% endhint %}

### Payload extension

* Required:
  * `openssl` extension
  * `mbstring` extension
* Optional:
  * A PSR-6 (Caching Interface) implementation
  * A PSR-3 (Logger Interface) implementation for debugging
