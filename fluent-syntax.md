# Fluent Syntax

In the documentation, you will see that methods are called “fluently”.

```php
<?php

use WebPush\Payload\AES128GCM;
use WebPush\Payload\AESGCM;
use WebPush\Payload\PayloadExtension;

$payloadExtension = PayloadExtension::create()
    ->addContentEncoding(AESGCM::create()->maxPadding())
    ->addContentEncoding(AES128GCM::create()->maxPadding())
;
```

If you don’t adhere to this coding style, you are free to use the “standard” way of coding. The following example has the same behavior ase above.

```php
<?php

use WebPush\Payload\AES128GCM;
use WebPush\Payload\AESGCM;
use WebPush\Payload\PayloadExtension;

$aesgcm = new AESGCM();
$aesgcm->maxPadding();

$aes128gcm = new AES128GCM();
$aes128gcm->maxPadding();

$payloadExtension = new PayloadExtension();
$payloadExtension->addContentEncoding($aesgcm);
$payloadExtension->addContentEncoding($aes128gcm);
```
