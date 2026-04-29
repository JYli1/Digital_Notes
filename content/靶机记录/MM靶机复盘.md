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
泄露了很多git，就直接停了，用githacker拉下来看看