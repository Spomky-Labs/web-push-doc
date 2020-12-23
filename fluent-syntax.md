# Fluent Syntax

In the documentation, you will see that methods are called “fluently”.

```php
<?php

use Minishlink\WebPush\Payload\AES128GCM;
use Minishlink\WebPush\Payload\AESGCM;
use Minishlink\WebPush\Payload\PayloadExtension;

$payloadExtension = PayloadExtension::create()
    ->addContentEncoding(AESGCM::create()->maxPadding())
    ->addContentEncoding(AES128GCM::create()->maxPadding())
;
```

If you don’t adhere to this coding style, you are free to use the “standard” way of coding. The following example has the same behavior ase above.

```php
<?php

use Minishlink\WebPush\Payload\AES128GCM;
use Minishlink\WebPush\Payload\AESGCM;
use Minishlink\WebPush\Payload\PayloadExtension;

$aesgcm = new AESGCM();
$aesgcm->maxPadding();

$aes128gcm = new AES128GCM();
$aes128gcm->maxPadding();

$payloadExtension = new PayloadExtension();
$payloadExtension->addContentEncoding($aesgcm);
$payloadExtension->addContentEncoding($aes128gcm);
```

