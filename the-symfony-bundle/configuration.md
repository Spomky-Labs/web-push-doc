# Configuration

## VAPID Support

To enable the VAPID header feature, you must install a JWS Provider (see [installation](installation.md)) and configure it with your public and private key (see [this page](../common-concepts/vapid.md) to create these keys)

{% code title="config/packages/webpush.yaml" %}
```yaml
webpush:
  vapid:
    enabled: true # Enable the feature
    subject: 'https://my-service.com:8000' # An URL or an email address
    web_token:
      enabled: true # We use web-token in this example
      public_key: 'BB4W1qfBi7MF_Lnrc6i2oL-glAuKF4kevy9T0k2vyKV4qvuBrN3T6o9-7-NR3mKHwzDXzD3fe7XvIqIU1iADpGQ'
      private_key: 'C40jLFSa5UWxstkFvdwzT3eHONE2FIJSEsVIncSCAqU'
```
{% endcode %}

When using `lcobucci/jwt`, the configuration is very similar.

{% code title="config/packages/webpush.yaml" %}
```yaml
webpush:
  vapid:
    enabled: true
    subject: 'https://my-service.com:8000'
    lcobucci:
      enabled: true # We use web-token in this example
      public_key: 'BB4W1qfBi7MF_Lnrc6i2oL-glAuKF4kevy9T0k2vyKV4qvuBrN3T6o9-7-NR3mKHwzDXzD3fe7XvIqIU1iADpGQ'
      private_key: 'C40jLFSa5UWxstkFvdwzT3eHONE2FIJSEsVIncSCAqU'
```
{% endcode %}

{% hint style="danger" %}
You cannot enable both `web-token` and `lcobucci/jwt` at the same time
{% endhint %}

### Token Lifetime

By default, the library generates VAPID headers that are valid for 1 hour. You can change this value if needed. The parameter requires a relative string as showed [in the PHP documentation](https://www.php.net/manual/en/datetime.formats.relative.php).

{% code title="config/packages/webpush.yaml" %}
```yaml
webpush:
  vapid:
    enabled: true
    token_lifetime: 'now +2 hours'
```
{% endcode %}

{% hint style="warning" %}
The token lifetime should not be greater than 24 hours. Most of the Web Push Services will reject such long-life tokens
{% endhint %}

## Payload Support

### Padding

To obfuscate the real length of the notifications, messages can be padded before encryption. This operation consists in the concatenation of your message and arbitrary data in front of it. When encrypted, the messages will have the same size which reduces attacks.

By default, the padding is set to `recommended` i.e. \~3k bytes.

Acceptable values for this parameter are:

* `none`: no padding
* `recommended`: default value
* `max`: see warning below
* an integer: should be between `0` and `4078` or `3993` for `AESGCM` and `AES128GCM` respectively

{% code title="config/packages/webpush.yaml" %}
```yaml
webpush:
  payload:
    aesgcm:
      padding: 'none' # "none", "recommended", "max" or an integer (0 to 4078)
    aes128gcm:
      padding: 'none' # "none", "recommended", "max" or an integer (0 to 3993)
```
{% endcode %}

{% hint style="danger" %}
Please don't use "`none`" unless your are sending notifications in a development environment.
{% endhint %}

{% hint style="warning" %}
The value "`max`" increases the integrity protection of the messages, but there are known issues on Android and notification are not correctly delivered.
{% endhint %}

### Caching

The notifications [may have a payload](../common-concepts/the-notification.md#json-messages). This payload is encrypted on server side and, during this process, a random key is generated.

The creation of this random key takes approximately 150ms and can impact your server performance when sending thousand of notifications at once.

To reduce the impact on your server, you can enable the caching feature and reuse the encryption key for a defined period of time.

{% hint style="danger" %}
As encryption keys will be stored in the cache, you should make sure the cache is not shared otherwise you may have a security issue.
{% endhint %}

This parameter requires a PSR-6 Cache compatible service. If you set `Psr\Log\CacheItemPoolInterface`, the default Symfony cache will be used.

{% code title="config/packages/webpush.yaml" %}
```yaml
webpush:
  payload:
    aesgcm:
      cache: Psr\Cache\CacheItemPoolInterface
      cache_lifetime: '+1 hour' #Default: now +30min
    aes128gcm:
      cache: Psr\Cache\CacheItemPoolInterface
      cache_lifetime: '+1 hour' #Default: now +30min
```
{% endcode %}

{% hint style="success" %}
You can see the impact of this feature on the CI/CD Pipelines of this library. Go the [https://github.com/Spomky-Labs/web-push/actions?query=workflow%3ABenchmark](https://github.com/Spomky-Labs/web-push/actions?query=workflow%3ABenchmark) and find a summary table displayed at the end of each test.
{% endhint %}

## Debugging

If you have troubles sending notifications, you can log some messages from the libray. To do so, you just have to set the parameter logger in the configuration.

This parameter requires a PSR-3 logger. If you set `Psr\Log\LoggerInterface`, the Symfony logger will be used (PSR-3 copmpatible).

{% code title="config/packages/webpush.yaml" %}
```yaml
webpush:
  logger: Psr\Log\LoggerInterface
```
{% endcode %}
