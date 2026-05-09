---
名称: break
作者: Sublarge
靶机id: "646"
难度: Medium
---
# 主机发现
```bash
┌──(kali㉿kali)-[~]
└─$ nmap 10.216.75.0/24 -sn
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-09 13:52 CST
Nmap scan report for 10.216.75.104
Host is up (0.00089s latency).
MAC Address: 08:00:27:3B:5D:A1 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.216.75.183
Host is up (0.0028s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.216.75.212
Host is up (0.00051s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.216.75.80
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 6.07 seconds
```
靶机ip：`10.216.75.104`

# 信息收集
```md
┌──(kali㉿kali)-[~]
└─$ what

──────────────────────────────────────────────────
$ rustscan -a 10.216.75.104 --ulimit 5000 -- -A -sC -sV  (exit: 0)
──────────────────────────────────────────────────

一句话结论

目标 10.216.75.104 是一个运行着 Web (Website Baker CMS)、SSH 和 MariaDB 数据库 的 Linux
主机，存在多个潜在的渗透切入点，需优先对 Web 应用和数据库进行漏洞利用测试。

关键发现

开放端口与服务


 端口      服务   版本                         备注
 ──────────────────────────────────────────────────────────────────────────────────────
 22/tcp    SSH    OpenSSH 10.0 (protocol 2.0)  较新版本，暂无公开RCE
 80/tcp    HTTP   Apache httpd 2.4.66 (Unix)   标题: Dairy - Website Baker
 3306/tcp  MySQL  MariaDB 11.4.8               支持SSL，认证插件: mysql_native_password


Web 应用（80端口）

 • CMS 识别：Website Baker（老牌CMS，存在历史漏洞，如文件包含、SQL注入、RCE）。
 • HTTP 方法：支持 GET, HEAD, POST, OPTIONS（无PUT/DELETE）。
 • Favicon：MD5 87D268F322601053318326ACE0A561BC，可尝试搜索对应CMS版本。

数据库（3306端口）

 • 版本较新（MariaDB 11.4.8），但认证协议为传统mysql_native_password，允许密码爆破。
 • SSL/TLS 证书信息：commonName=MariaDB Server，无其他限制。

操作系统与网络

 • OS：Linux 4.x ~ 5.x（可能为OpenWrt / MikroTik，但更可能是普通Linux发行版）。
 • MAC：08:00:27:3B:5D:A1 → Oracle VirtualBox虚拟机（内网靶机）。
 • 距离：1跳，局域网内。

下一步建议

1. Web 应用深度侦察（优先）

 • 目录枚举：

    gobuster dir -u http://10.216.75.104 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x
    php,html,txt

 • 查看 robots.txt：
   curl http://10.216.75.104/robots.txt
 • 寻找已知漏洞：
   使用 searchsploit website baker 或 Metasploit 搜索模块，重点关注：
    • 文件上传绕过（→远程代码执行）
    • SQL注入（→获取管理员密码）
    • 本地文件包含（→读取配置/数据库凭证）
 • 弱密码登录测试：
   hydra -L users.txt -P passwords.txt 10.216.75.104 http-post-form
   "/admin/login.php:username=^USER^&password=^PASS^:F=incorrect"

2. 数据库攻击

 • 尝试弱密码登录（root空密码、root/root等）：

    mysql -h 10.216.75.104 -u root -p   # 尝试无密码

 • 爆破密码：

    hydra -l root -P /usr/share/wordlists/rockyou.txt 10.216.75.104 mysql

 • 如果成功：
    • 查询 WordPress / Website Baker 用户表 → 提取密码哈希 → 破解。
    • 尝试读取 /etc/passwd 或系统文件（需要 FILE 权限）：

       SELECT LOAD_FILE('/etc/passwd');


3. SSH 枚举

 • 适当尝试流行弱密码（root:root, admin:admin, user:user），但避免大规模爆破（浪费资源）。
 • 如果从 Web 或数据库获得有效凭证，立即尝试 SSH 登录。

4. 横向与提权准备

 • 如果获得初始 shell，立即运行 sudo -l、find / -perm -4000 -type f 等提权指令。
 • 检查是否有敏感文件（数据库配置、备份等）。
 • 尝试 uname -a 检查内核版本（可能适用脏牛等本地提权）。

▌ 当前阶段，Website Baker CMS 是最佳突破口，集中精力挖掘 Web 漏洞（尤其是历史RCE）和数据库弱口令。

```
我们到web端看了一下，搜索会跳转
```html
http://baker.dsz/search/index.php?referrer=2&string=1&wb_search=%EE%82%90
```
所以我们去添加一个host记录
```md
10.216.75.104 baker.dsz
```
这才成功加载出完整网页
![](file-20260509140652318.png)
## 目录扫描
```bash
PS D:\webtool\Dirsearch> python dirsearch.py -u http://baker.dsz -e *
D:\webtool\Dirsearch\lib\core\installation.py:24: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, jsp, asp, aspx, do, action, cgi, html, htm, js, tar.gz | HTTP method: GET | Threads: 25 | Wordlist size: 15042

Target: http://baker.dsz/

[14:07:39] Scanning:
[14:07:41] 200 -    25B - /.gitignore
[14:07:47] 301 -   346B - /account  ->  http://baker.dsz/account/
[14:07:47] 301 -     0B - /account/  ->  login.php
[14:07:47] 302 -     0B - /account/login.php  ->  http://baker.dsz/index.php
[14:07:48] 301 -   344B - /admin  ->  http://baker.dsz/admin/
[14:07:49] 403 -   312B - /admin/.htaccess
[14:07:49] 302 -     0B - /admin/  ->  http://baker.dsz/admin/start/index.php
[14:07:49] 301 -   350B - /admin/login  ->  http://baker.dsz/admin/login/
[14:07:49] 302 -     0B - /admin/index.php  ->  http://baker.dsz/admin/start/index.php
[14:08:05] 403 -   312B - /cgi-bin/
[14:08:05] 403 -   312B - /cgi-bin/a1stats/a1disp.cgi
[14:08:05] 403 -   312B - /cgi-bin/awstats/
[14:08:05] 403 -   312B - /cgi-bin/awstats.pl
[14:08:05] 403 -   312B - /cgi-bin/htimage.exe?2,2
[14:08:05] 403 -   312B - /cgi-bin/imagemap.exe?2,2
[14:08:05] 403 -   312B - /cgi-bin/htmlscript
[14:08:05] 403 -   312B - /cgi-bin/index.html
[14:08:05] 403 -   312B - /cgi-bin/login.cgi
[14:08:05] 403 -   312B - /cgi-bin/login.php
[14:08:05] 403 -   312B - /cgi-bin/mt-xmlrpc.cgi
[14:08:05] 403 -   312B - /cgi-bin/login
[14:08:05] 403 -   312B - /cgi-bin/mt/mt.cgi
[14:08:05] 403 -   312B - /cgi-bin/mt/mt-xmlrpc.cgi
[14:08:05] 403 -   312B - /cgi-bin/mt7/mt-xmlrpc.cgi
[14:08:05] 403 -   312B - /cgi-bin/mt7/mt.cgi
[14:08:05] 403 -   312B - /cgi-bin/printenv
[14:08:05] 403 -   312B - /cgi-bin/printenv.pl
[14:08:05] 403 -   312B - /cgi-bin/test.cgi
[14:08:05] 403 -   312B - /cgi-bin/test-cgi
[14:08:05] 403 -   312B - /cgi-bin/mt.cgi
[14:08:05] 403 -   312B - /cgi-bin/ViewLog.asp
[14:08:05] 403 -   312B - /cgi-bin/php.ini
[14:08:06] 200 -   136B - /CHANGELOG.md
[14:08:09] 200 -     0B - /config.php
[14:08:16] 200 -   34KB - /favicon.ico
[14:08:22] 301 -   346B - /include  ->  http://baker.dsz/include/
[14:08:22] 301 -     0B - /include/  ->  ../index.php
[14:08:23] 200 -    6KB - /index.php
[14:08:23] 200 -    6KB - /index.php/login/
[14:08:23] 200 -    1KB - /INSTALL.md
[14:08:27] 301 -   348B - /languages  ->  http://baker.dsz/languages/
[14:08:29] 200 -   15KB - /LICENSE.md
[14:08:35] 301 -   344B - /media  ->  http://baker.dsz/media/
[14:08:35] 200 -   401B - /media/
[14:08:37] 301 -   346B - /modules  ->  http://baker.dsz/modules/
[14:08:37] 301 -     0B - /modules/  ->  ../index.php
[14:08:41] 301 -   344B - /pages  ->  http://baker.dsz/pages/
[14:08:41] 301 -     0B - /pages/  ->  ../index.php
[14:08:47] 200 -    35B - /README.md
[14:08:50] 301 -   345B - /search  ->  http://baker.dsz/search/
[14:08:51] 403 -   312B - /server-status/
[14:08:51] 403 -   312B - /server-status
[14:08:59] 301 -   343B - /temp  ->  http://baker.dsz/temp/
[14:08:59] 301 -     0B - /temp/  ->  ../index.php
[14:08:59] 301 -   348B - /templates  ->  http://baker.dsz/templates/
[14:08:59] 301 -     0B - /templates/  ->  ../index.php
[14:09:03] 301 -   342B - /var  ->  http://baker.dsz/var/
[14:09:03] 301 -     0B - /var/  ->  ../index.php
[14:09:04] 200 -   387B - /var/logs/
```
都看了一下，有用的不多，就知道了用的`WBCE CMS`还有一个`admin后台`
试一下弱密码，发现错误几次居然就封了。
![](file-20260509141640939.png)
# 渗透测试
尝试root root登录数据库，居然成功进来了。
连上navicat看看，虽然是root账号都是我们只有很小的权限
	能看`wbce_test`库，我们看到了`wbce_user`表，里面有admin账户和一段密码哈希
我们john爆破一下
```bash
┌──(kali㉿kali)-[~/tmp/tmp]
└─$ john --show hash.txt
?:33333333

1 password hash cracked, 0 left
```
爆出来密码，但是去后台登录不了，说不正确

找到一个注册的地方，可能存在邮箱的用户名爆破，我们用脚本：

```bash
┌──(kali㉿kali)-[~/tmp/tmp]
└─$ python3 111.py
[*] loaded 4950 names
[*] target: http://baker.dsz/admin/login/forgot/index.php
[*] domains: baker.dsz, baker.local
[*] wait: 0s/request
[1792/9900] checking carol@baker.dsz
[?] carol@baker.dsz: captcha_or_asp
[1878/9900] checking martina@baker.dsz
[?] martina@baker.dsz: captcha_or_asp
[9900/9900] checking speaker@baker.local
[-] no matching email found
```


爆出来用户名：carol。martina
我试了一下，一个管理员，一个地权限用户，我创建了一个用户，用cve的poc打了一下
![](file-20260509234920801.png)
确定有时间盲注，这里好像不需要这个盲注，直接可以打`CVE-2024-10331：Droplets 模块远程代码执行 (RCE)`
我们首先用管理员用户
	`Admin-tools`->`Droplets`
在这个页面我们可以写php代码，相当于写一个php函数一样，然后我们之后就可以在page里面使用`[[函数名]]`调用
![](file-20260510002458176.png)
这样写一个webshell。
然后来到`pages`。新建一个文章
![](file-20260510002610904.png)
这里test是我写的，我们看一下。
![](file-20260510002635112.png)
page的内容随便写什么，只要里面用了`[[shell]]`(这里shell是因为我创建的`Droplets`的name是shell)，就可以了。
![](file-20260510002813925.png)
然后我们点击这个`view`或者直接访问`http://baker.dsz/pages/test.php`(这里test是你创建的page的名字)
然后就拿到webshell了。
![](file-20260510002936514.png)
`flag{user-548b5242171e085fc64be9252a132ad5}`
