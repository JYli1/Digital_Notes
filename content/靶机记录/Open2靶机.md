---
作者: 明明就
靶机id: "648"
系统: Linux
难度: Easy
---
# 主机发现
```bash
┌──(kali㉿kali)-[~]
└─$ nmap 10.216.75.0/24 -sn
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-05 23:24 CST
Nmap scan report for 10.216.75.72
Host is up (0.0018s latency).
MAC Address: 08:00:27:98:C5:BC (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.216.75.183
Host is up (0.035s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.216.75.212
Host is up (0.0012s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.216.75.80
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.60 seconds

```
确定靶机IP：
`10.216.75.72`
# 端口扫描
```bash
┌──(kali㉿kali)-[~]
└─$ rustscan -a 10.216.75.72
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
Please contribute more quotes to our GitHub https://github.com/rustscan/rustscan

[~] The config file is expected to be at "/home/kali/.rustscan.toml"
[!] File limit is lower than default batch size. Consider upping with --ulimit. May cause harm to sensitive servers
[!] Your file limit is very small, which negatively impacts RustScan's speed. Use the Docker image, or up the Ulimit with '--ulimit 5000'.
Open 10.216.75.72:80
[~] Starting Script(s)
[~] Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-05 23:26 CST
Initiating ARP Ping Scan at 23:26
Scanning 10.216.75.72 [1 port]
Completed ARP Ping Scan at 23:26, 0.11s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 23:26
Completed Parallel DNS resolution of 1 host. at 23:26, 0.02s elapsed
DNS resolution of 1 IPs took 0.02s. Mode: Async [#: 1, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 23:26
Scanning 10.216.75.72 [1 port]
Discovered open port 80/tcp on 10.216.75.72
Completed SYN Stealth Scan at 23:26, 0.03s elapsed (1 total ports)
Nmap scan report for 10.216.75.72
Host is up, received arp-response (0.0021s latency).
Scanned at 2026-05-05 23:26:24 CST for 0s

PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
MAC Address: 08:00:27:98:C5:BC (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.39 seconds
           Raw packets sent: 2 (72B) | Rcvd: 2 (72B)
```
工具不太熟悉，这么多结果。用工具总结一手：
```bash
┌──(kali㉿kali)-[~]
└─$ what 总结一下结果并且解释一下两个感叹号是什么 --md

──────────────────────────────────────────────────
$ rustscan -a 10.216.75.72  (exit: 0)
──────────────────────────────────────────────────

命令执行结果：

 • 工具：rustscan 扫描目标 IP 10.216.75.72，发现开放端口 80/tcp（HTTP 服务）。
 • 自动调用 Nmap 对该端口详细扫描，确认主机在线且端口开放，MAC 地址为 VirtualBox 虚拟机网卡。

两个感叹号警告含义：

 1 文件限制低于默认批处理大小：当前系统允许打开的文件数太少，可能导致 RustScan 批处理端口时对敏感服务器造成意外影响（如发送大量请求）。建议用
   --ulimit 提高限制。
 2 文件限制过小严重影响扫描速度：提示当前 ulimit 值太小，推荐使用 Docker 镜像或运行 --ulimit 5000 提升速度。
```
发现只开了80端口。我们先去web看看。
就一个登录页面，扫一下目录
```bash
D:\webtool\Dirsearch\lib\core\installation.py:24: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm | HTTP method: GET | Threads: 25 | Wordlist size: 11946

Target: http://10.216.75.72/

[00:01:44] Scanning:
[00:01:47] 403 -   277B - /.php
[00:02:06] 200 -    2KB - /index.php
[00:02:06] 200 -    2KB - /index.php/login/
[00:02:17] 403 -   277B - /server-status
[00:02:17] 403 -   277B - /server-status/
[00:02:25] 400 -   304B - /.%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd
```
好像也没扫到啥有用的，莫非要去爆破了吗？暂时没思路。。。
