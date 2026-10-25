# Fluent Syntax

In the documentation, you will see that methods are called "fluently".

```php
<?php

use Psr\Clock\ClockInterface;
use Symfony\Component\Clock\NativeClock;
use WebPush\Payload\AES128GCM;
use WebPush\Payload\AESGCM;
use WebPush\Payload\PayloadExtension;

$clock = new NativeClock(); // PSR-20 Clock implementation

$payloadExtension = PayloadExtension::create()
    ->addContentEncoding(AESGCM::create($clock)->maxPadding())
    ->addContentEncoding(AES128GCM::create($clock)->maxPadding())
;
```

If you don't adhere to this coding style, you are free to use the "standard" way of coding. The following example has the same behavior as above.

```php
<?php

use Psr\Clock\ClockInterface;
use Symfony\Component\Clock\NativeClock;
use WebPush\Payload\AES128GCM;
use WebPush\Payload\AESGCM;
use WebPush\Payload\PayloadExtension;

$clock = new NativeClock(); // PSR-20 Clock implementation

$aesgcm = new AESGCM($clock);
$aesgcm->maxPadding();

$aes128gcm = new AES128GCM($clock);
$aes128gcm->maxPadding();

$payloadExtension = new PayloadExtension();
$payloadExtension->addContentEncoding($aesgcm);
$payloadExtension->addContentEncoding($aes128gcm);
```
