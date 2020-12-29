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
  * A JWT Provider. This library provides implementations for [web-token](https://web-token.spomky-labs.com) and [lcobucci/jwt](https://github.com/lcobucci/jwt)
* Optional:
  * A PSR-3 \(Logger Interface\) implementation for debugging

### Payload extension

* Required:
  * `openssl` extension
  * `mbstring` extension
* Optional:
  * A PSR-6 \(Caching Interface\) implementation \(\*highly recommended\*\)
  * A PSR-3 \(Logger Interface\) implementation for debugging

