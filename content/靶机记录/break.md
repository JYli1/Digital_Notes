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
