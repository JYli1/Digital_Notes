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
## 