---
靶机id: 大傻子
作者: "638"
---
## 主机发现
```bash
┌──(root㉿kali)-[/home/kali]
└─# nmap -sn 10.241.108.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-29 14:16 CST
Nmap scan report for 10.241.108.43
Host is up (0.062s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.241.108.212
Host is up (0.00080s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.241.108.244
Host is up (0.0020s latency).
MAC Address: 08:00:27:FC:21:A8 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.241.108.201
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 37.26 seconds
```
发现靶机ip：`10.241.108.201`
## 端口扫描
```zsh
┌──(root㉿kali)-[/home/kali]
└─# nmap -p- 10 10.241.108.244
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-29 14:19 CST
Nmap scan report for 10.241.108.244
Host is up (0.0016s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5901/tcp open  vnc-1
6001/tcp open  X11:1
MAC Address: 08:00:27:FC:21:A8 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 2 IP addresses (1 host up) scanned in 19.29 seconds
```
除了22端口还发现了`80`、`5901`、`6001`

## vnc了解
* **VNC** 是一种让你用一台电脑（客户端）远程操作另一台电脑（服务器）桌面的技术

`5901/tcp`：VNC 服务的主端口 **大门**，你从这里连接进去|
`6001/tcp`：X11 服务端口（Linux图形底层）**内部管道**，服务器自己用的，你不需要管它|
`:1`：（显示编号）这是第1号图形会话 如果同时开多个桌面，就是 `:1`、`:2`...|

连接命令：
```zsh
vncviewer 10.241.108.244:5901
```

## 渗透测试
因为vnc连接需要密码，我们暂时没有，所以先去web端看一看。
![](file-20260429143428767.png)
一个静态页面，什么也没有，所以就扫一下目录看看。
```bash
PS D:\webtool\Dirsearch> python dirsearch.py -u 10.241.108.244
D:\webtool\Dirsearch\lib\core\installation.py:24: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm | HTTP method: GET | Threads: 25 | Wordlist size: 12289

Target: http://10.241.108.244/

[14:36:09] Scanning:
[14:36:10] 301 -   315B - /.git  ->  http://10.241.108.244/.git/
[14:36:10] 200 -   762B - /.git/branches/
[14:36:10] 200 -     2B - /.git/COMMIT_EDITMSG
[14:36:10] 200 -    4KB - /.git/hooks/
[14:36:10] 200 -    73B - /.git/description
[14:36:10] 200 -    3KB - /.git/
[14:36:10] 200 -    92B - /.git/config
[14:36:10] 200 -   145B - /.git/index
```
泄露了很多git，就直接停了，用githacker拉下来看看：
```bash
PS D:\webtool\GitHacker> githacker --url http://10.241.108.244/.git/ --output-folder result
2026-04-29 14:43:26 INFO 1 urls to be exploited
2026-04-29 14:43:26 INFO Exploiting http://10.241.108.244/.git/ into result\6ccb2befc73ddd0b28240fec20a21fcf
2026-04-29 14:43:26 INFO Directory listing enable under: apache
2026-04-29 14:43:26 ERROR [2880 bytes] 200 .git/?C=N;O=D
2026-04-29 14:43:26 ERROR [2880 bytes] 200 .git/?C=D;O=A
2026-04-29 14:43:26 ERROR [2880 bytes] 200 .git/?C=M;O=A
2026-04-29 14:43:26 ERROR [2880 bytes] 200 .git/?C=S;O=A
2026-04-29 14:43:26 INFO [2 bytes] 200 .git/COMMIT_EDITMSG
2026-04-29 14:43:26 INFO [23 bytes] 200 .git/HEAD
2026-04-29 14:43:26 ERROR [762 bytes] 200 .git/branches/?C=N;O=D
2026-04-29 14:43:26 ERROR [762 bytes] 200 .git/branches/?C=M;O=A
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\config is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/config
2026-04-29 14:43:26 ERROR [762 bytes] 200 .git/branches/?C=D;O=A
2026-04-29 14:43:26 ERROR [762 bytes] 200 .git/branches/?C=S;O=A
2026-04-29 14:43:26 INFO [73 bytes] 200 .git/description
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\?C=N;O=D is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\?C=S;O=A is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/?C=N;O=D
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/?C=S;O=A
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\?C=M;O=A is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\?C=D;O=A is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/?C=M;O=A
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/?C=D;O=A
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\applypatch-msg.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/applypatch-msg.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\commit-msg.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/commit-msg.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\fsmonitor-watchman.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/fsmonitor-watchman.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\post-update.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/post-update.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\pre-applypatch.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/pre-applypatch.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\pre-commit.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/pre-commit.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\pre-rebase.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/pre-rebase.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\pre-push.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\pre-receive.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/pre-push.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\pre-merge-commit.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/pre-merge-commit.sample
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/pre-receive.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\prepare-commit-msg.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/prepare-commit-msg.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\push-to-checkout.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/push-to-checkout.sample
2026-04-29 14:43:26 ERROR C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7\.git\hooks\update.sample is potential dangerous, skip downloading this file
2026-04-29 14:43:26 ERROR [-1 bytes] -1 .git/hooks/update.sample
2026-04-29 14:43:26 ERROR [949 bytes] 200 .git/info/?C=N;O=D
2026-04-29 14:43:26 INFO [145 bytes] 200 .git/index
2026-04-29 14:43:26 ERROR [949 bytes] 200 .git/info/?C=M;O=A
2026-04-29 14:43:26 ERROR [949 bytes] 200 .git/info/?C=S;O=A
2026-04-29 14:43:26 ERROR [949 bytes] 200 .git/info/?C=D;O=A
2026-04-29 14:43:26 INFO [240 bytes] 200 .git/info/exclude
2026-04-29 14:43:26 ERROR [1133 bytes] 200 .git/logs/?C=N;O=D
2026-04-29 14:43:26 ERROR [1133 bytes] 200 .git/logs/?C=M;O=A
2026-04-29 14:43:26 ERROR [1133 bytes] 200 .git/logs/?C=S;O=A
2026-04-29 14:43:26 ERROR [1133 bytes] 200 .git/logs/?C=D;O=A
2026-04-29 14:43:26 INFO [578 bytes] 200 .git/logs/HEAD
2026-04-29 14:43:26 ERROR [961 bytes] 200 .git/logs/refs/?C=N;O=D
2026-04-29 14:43:26 ERROR [961 bytes] 200 .git/logs/refs/?C=M;O=A
2026-04-29 14:43:26 ERROR [961 bytes] 200 .git/logs/refs/?C=S;O=A
2026-04-29 14:43:26 ERROR [961 bytes] 200 .git/logs/refs/?C=D;O=A
2026-04-29 14:43:26 ERROR [979 bytes] 200 .git/logs/refs/heads/?C=N;O=D
2026-04-29 14:43:26 ERROR [979 bytes] 200 .git/logs/refs/heads/?C=M;O=A
2026-04-29 14:43:26 ERROR [979 bytes] 200 .git/logs/refs/heads/?C=S;O=A
2026-04-29 14:43:26 ERROR [979 bytes] 200 .git/logs/refs/heads/?C=D;O=A
2026-04-29 14:43:26 INFO [578 bytes] 200 .git/logs/refs/heads/master
2026-04-29 14:43:26 ERROR [3000 bytes] 200 .git/objects/?C=N;O=D
2026-04-29 14:43:26 ERROR [3000 bytes] 200 .git/objects/?C=S;O=A
2026-04-29 14:43:26 ERROR [3000 bytes] 200 .git/objects/?C=M;O=A
2026-04-29 14:43:26 ERROR [3000 bytes] 200 .git/objects/?C=D;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/2d/?C=N;O=D
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/2d/?C=M;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/2d/?C=S;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/2d/?C=D;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/19/?C=N;O=D
2026-04-29 14:43:26 INFO [55 bytes] 200 .git/objects/2d/ce93ea08ed9059be0a838c6bcf62b7b5c28907
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/19/?C=M;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/19/?C=S;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/19/?C=D;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/67/?C=N;O=D
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/67/?C=M;O=A
2026-04-29 14:43:26 INFO [147 bytes] 200 .git/objects/19/36b7f0b8bc34642423c19738fab503a9d967de
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/67/?C=S;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/67/?C=D;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/90/?C=N;O=D
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/90/?C=M;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/90/?C=S;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/90/?C=D;O=A
2026-04-29 14:43:26 INFO [34 bytes] 200 .git/objects/67/3b2216187128dc73088ac5df036e854798c68f
2026-04-29 14:43:26 INFO [21 bytes] 200 .git/objects/90/15a7a32ca0681be64471d3ac2f8c1f24c1040d
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/ab/?C=N;O=D
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/ab/?C=M;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/ab/?C=S;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/ab/?C=D;O=A
2026-04-29 14:43:26 INFO [55 bytes] 200 .git/objects/ab/d028b26786a20ba6f9dfe4de5305b7341c2395
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/b8/?C=M;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/b8/?C=N;O=D
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/b8/?C=S;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/b8/?C=D;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/bd/?C=N;O=D
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/bd/?C=M;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/bd/?C=S;O=A
2026-04-29 14:43:26 INFO [147 bytes] 200 .git/objects/b8/295d6c67f5f2df8a3649af13bf6867b221cd17
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/bd/?C=D;O=A
2026-04-29 14:43:26 INFO [1000 bytes] 200 .git/objects/bd/9990a1d46f17332711ccdf1d5ca32d584ae5c3
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f6/?C=N;O=D
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f6/?C=M;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f6/?C=S;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f6/?C=D;O=A
2026-04-29 14:43:26 INFO [86 bytes] 200 .git/objects/f6/86416a62f2dc219fc3d6168fd0ae38516f6422
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f7/?C=M;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f7/?C=N;O=D
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f7/?C=S;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f7/?C=D;O=A
2026-04-29 14:43:26 INFO [147 bytes] 200 .git/objects/f7/cc50a34b65f1c6cf3c8bd10e2b78271c348e35
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f9/?C=S;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f9/?C=N;O=D
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f9/?C=M;O=A
2026-04-29 14:43:26 ERROR [1031 bytes] 200 .git/objects/f9/?C=D;O=A
2026-04-29 14:43:26 INFO [117 bytes] 200 .git/objects/f9/f7d8ba3292488a6e7f9fa21d0968ca7bbd6637
2026-04-29 14:43:26 ERROR [778 bytes] 200 .git/objects/info/?C=M;O=A
2026-04-29 14:43:26 ERROR [778 bytes] 200 .git/objects/info/?C=N;O=D
2026-04-29 14:43:26 ERROR [778 bytes] 200 .git/objects/info/?C=S;O=A
2026-04-29 14:43:26 ERROR [778 bytes] 200 .git/objects/info/?C=D;O=A
2026-04-29 14:43:26 ERROR [778 bytes] 200 .git/objects/pack/?C=N;O=D
2026-04-29 14:43:26 ERROR [778 bytes] 200 .git/objects/pack/?C=S;O=A
2026-04-29 14:43:26 ERROR [778 bytes] 200 .git/objects/pack/?C=M;O=A
2026-04-29 14:43:26 ERROR [778 bytes] 200 .git/objects/pack/?C=D;O=A
2026-04-29 14:43:26 ERROR [1136 bytes] 200 .git/refs/?C=M;O=A
2026-04-29 14:43:26 ERROR [1136 bytes] 200 .git/refs/?C=D;O=A
2026-04-29 14:43:26 ERROR [1136 bytes] 200 .git/refs/?C=S;O=A
2026-04-29 14:43:26 ERROR [1136 bytes] 200 .git/refs/?C=N;O=D
2026-04-29 14:43:26 ERROR [964 bytes] 200 .git/refs/heads/?C=S;O=A
2026-04-29 14:43:26 ERROR [964 bytes] 200 .git/refs/heads/?C=N;O=D
2026-04-29 14:43:26 ERROR [964 bytes] 200 .git/refs/heads/?C=M;O=A
2026-04-29 14:43:26 ERROR [964 bytes] 200 .git/refs/heads/?C=D;O=A
2026-04-29 14:43:26 INFO [41 bytes] 200 .git/refs/heads/master
2026-04-29 14:43:26 ERROR [769 bytes] 200 .git/refs/tags/?C=N;O=D
2026-04-29 14:43:26 ERROR [769 bytes] 200 .git/refs/tags/?C=M;O=A
2026-04-29 14:43:26 ERROR [769 bytes] 200 .git/refs/tags/?C=S;O=A
2026-04-29 14:43:26 ERROR [769 bytes] 200 .git/refs/tags/?C=D;O=A
2026-04-29 14:43:26 INFO Cloning downloaded repo from C:\Users\15819\AppData\Local\Temp\tmplb5ca5f7 to result\6ccb2befc73ddd0b28240fec20a21fcf
2026-04-29 14:43:27 ERROR Cloning into 'result\6ccb2befc73ddd0b28240fec20a21fcf'...
done.
2026-04-29 14:43:27 INFO Check it out: result\6ccb2befc73ddd0b28240fec20a21fcf
2026-04-29 14:43:27 INFO 1 / 1 were exploited successfully
2026-04-29 14:43:27 INFO http://10.241.108.244/.git/ -> result\6ccb2befc73ddd0b28240fec20a21fcf
```
z注意这里虽然控制台输出很多`error`，但核心的 Git 对象和引用已经被正确抓取并重组到了 `result\6ccb2befc73ddd0b28240fec20a21fcf` 目录中。只是一些安全策略。
## git文件夹分析
```bash
PS D:\webtool\GitHacker\result\6ccb2befc73ddd0b28240fec20a21fcf> git log
commit 1936b7f0b8bc34642423c19738fab503a9d967de (HEAD -> master, origin/master, origin/HEAD)
Author: mingmingjiu <mingmingjiu@mm.dsz>
Date:   Sun Apr 19 00:10:25 2026 -0400

    4

commit b8295d6c67f5f2df8a3649af13bf6867b221cd17
Author: mingmingjiu <mingmingjiu@mm.dsz>
Date:   Sun Apr 19 00:08:12 2026 -0400

    3

commit f7cc50a34b65f1c6cf3c8bd10e2b78271c348e35
Author: mingmingjiu <mingmingjiu@mm.dsz>
Date:   Sun Apr 19 00:07:59 2026 -0400

    2

commit f9f7d8ba3292488a6e7f9fa21d0968ca7bbd6637
Author: mingmingjiu <mingmingjiu@mm.dsz>
Date:   Sun Apr 19 00:07:17 2026 -0400

    1
```
得到了用户名和域名
`mingmingjiu@mm.dsz`
既然有了域名那我们去添加一个host记录后访问
```bash
#windows


C:\Windows\System32\drivers\etc\host

# Copyright (c) 1993-2009 Microsoft Corp.
#
# This is a sample HOSTS file used by Microsoft TCP/IP for Windows.
#
# This file contains the mappings of IP addresses to host names. Each
# entry should be kept on an individual line. The IP address should
# be placed in the first column followed by the corresponding host name.
# The IP address and the host name should be separated by at least one
# space.
#
# Additionally, comments (such as these) may be inserted on individual
# lines or following the machine name denoted by a '#' symbol.
#
# For example:
#
#      102.54.94.97     rhino.acme.com          # source server
#       38.25.63.10     x.acme.com              # x client host

# localhost name resolution is handled within DNS itself.
#	127.0.0.1       localhost
#	::1             localhost
127.0.0.1 localhost
10.241.108.244 mm.dsz
```

```bash
#linux

/etc/hosts

┌──(kali㉿kali)-[~]
└─$ cat  /etc/hosts   
127.0.0.1       localhost
127.0.1.1       kali

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

10.241.108.226    acfun.dsz
10.241.108.244    mm.dsz
                         
```
![](file-20260429151526686.png)
访问后是一个登录页面，不能输入和点击，尝试修改前端代码，但是最后发现完全没有用，就是一个壳子。那这条路就断了
我们重新回到git文件夹。
我们都看一下
```bash
PS D:\webtool\GitHacker\result\6ccb2befc73ddd0b28240fec20a21fcf> git show f7cc50a34b65f1c6cf3c8bd10e2b78271c348e35
commit f7cc50a34b65f1c6cf3c8bd10e2b78271c348e35
Author: mingmingjiu <mingmingjiu@mm.dsz>
Date:   Sun Apr 19 00:07:59 2026 -0400

    2

diff --git a/pass.txt b/pass.txt
new file mode 100644
index 0000000..673b221
--- /dev/null
+++ b/pass.txt
@@ -0,0 +1 @@
+password:sublarge
PS D:\webtool\GitHacker\result\6ccb2befc73ddd0b28240fec20a21fcf> git show b8295d6c67f5f2df8a3649af13bf6867b221cd17
commit b8295d6c67f5f2df8a3649af13bf6867b221cd17
Author: mingmingjiu <mingmingjiu@mm.dsz>
Date:   Sun Apr 19 00:08:12 2026 -0400

    3

diff --git a/pass.txt b/pass.txt
deleted file mode 100644
index 673b221..0000000
--- a/pass.txt
+++ /dev/null
@@ -1 +0,0 @@
-password:sublarge
```
发现有删除了密码`sublarge`，但是后面试了一下不是这个密码。
这里有一个技巧如果有密码的话可以更快：
```bash
PS D:\webtool\GitHacker\result\6ccb2befc73ddd0b28240fec20a21fcf> git log --diff-filter=D --summary
commit b8295d6c67f5f2df8a3649af13bf6867b221cd17
Author: mingmingjiu <mingmingjiu@mm.dsz>
Date:   Sun Apr 19 00:08:12 2026 -0400

    3

 delete mode 100644 pass.txt
```
这个命令可以直接查看删除的文件
## 密码爆破
既然密码不对，那就没什么别的想法了，只能去爆破一下密码。
爆破ssh：
```bash
┌──(kali㉿kali)-[~]
└─$ hydra  -l mingmingjiu -P /usr/share/wordlists/rockyou.txt ssh://10.241.108.244  -t 4 -e nsr

Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-29 15:33:49
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344402 login tries (l:1/p:14344402), ~3586101 tries per task
[DATA] attacking ssh://10.241.108.244:22/
[22][ssh] host: 10.241.108.244   login: mingmingjiu   password: mingmingjiu
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-04-29 15:34:06
```
这里爆破vnc也是可以的，但是因为没有用户名所以不能用`-e nsr`去尝试用户名作为密码，如果自己添加一个密码字典那也是可以的。下面是直接爆破vnc：
```bash
┌──(kali㉿kali)-[~]
└─$ cat pass.txt
mingmingjiu

┌──(kali㉿kali)-[~]
└─$ hydra  -P ./pass.txt vnc://10.241.108.244 -s 5901 -t 4 -e nsr

Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-04-29 15:35:10
[DATA] max 4 tasks per 1 server, overall 4 tasks, 4 login tries (l:1/p:4), ~1 try per task
[DATA] attacking vnc://10.241.108.244:5901/
[5901][vnc] host: 10.241.108.244   password: mingmingjiu
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-04-29 15:35:11
```
得到密码：`mingmingjiu`

先尝试了ssh连接：
```bash
┌──(kali㉿kali)-[~]
└─$ ssh mingmingjiu@10.241.108.244
mingmingjiu@10.241.108.244's password:
Linux MM 4.19.0-27-amd64 #1 SMP Debian 4.19.316-1 (2024-06-25) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Apr 27 10:24:11 2026 from 10.241.108.201
This account is currently not available.
Connection to 10.241.108.244 closed.
```
但是连上马上就断开了，原因是这个用户没有shell权限。

### ssh登录失败复盘
获取shell之后看了一下`/etc/passwd`的内容
![](file-20260429225141347.png)
这里了解了一下，我们登录时：验证成功后，系统会尝试启动你在 /etc/passwd 中定义的 Shell（例如 /bin/bash）
如果定义的是 `/usr/sbin/nologin`，系统就会运行这个程序，它会打印一段文字（通常是 "This account is currently not available."），然后直接断开连接。

---

那我们试一下vnc连接
```bash
┌──(kali㉿kali)-[~]
└─$ vncviewer 10.241.108.244:5901
Connected to RFB server, using protocol version 3.8
Enabling TightVNC protocol extensions
Performing standard VNC authentication
Password: 
Authentication successful
Desktop name "mingmingjiu's X desktop (MM:1)"
VNC server default format:
  32 bits per pixel.
  Least significant byte first in each pixel.
  True colour: max red 255 green 255 blue 255, shift red 16 green 8 blue 0
Using default colormap which is TrueColor.  Pixel format:
  32 bits per pixel.
  Least significant byte first in each pixel.
  True colour: max red 255 green 255 blue 255, shift red 16 green 8 blue 0


```
连上了。但是打开终端救闪退。这里在Application中找一个其他终端可以打开`Xtrem`
![](file-20260429154141136.png)
## root提权
