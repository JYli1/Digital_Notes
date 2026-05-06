---
作者: L1Qin9
靶机id: "634"
难度: Easy
---
# 主机发现
```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sn 10.216.75.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-06 14:39 CST
Nmap scan report for 10.216.75.115
Host is up (0.0018s latency).
MAC Address: 08:00:27:89:39:82 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.216.75.183
Host is up (0.0061s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.216.75.212
Host is up (0.00093s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.216.75.80
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 17.30 seconds
```
靶机ip：`10.216.75.115`

# 信息收集
`rustscan`扫一下
```bash
┌──(kali㉿kali)-[~]
└─$ rustscan -a 10.216.75.115 --ulimit 5000
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
RustScan: Because guessing isn't hacking.

[~] The config file is expected to be at "/home/kali/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.216.75.115:22
Open 10.216.75.115:80
Open 10.216.75.115:8080
[~] Starting Script(s)
[~] Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-06 14:42 CST
Initiating ARP Ping Scan at 14:42
Scanning 10.216.75.115 [1 port]
Completed ARP Ping Scan at 14:42, 0.12s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 14:42
Completed Parallel DNS resolution of 1 host. at 14:42, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 1, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 14:42
Scanning 10.216.75.115 [3 ports]
Discovered open port 22/tcp on 10.216.75.115
Discovered open port 80/tcp on 10.216.75.115
Discovered open port 8080/tcp on 10.216.75.115
Completed SYN Stealth Scan at 14:42, 0.05s elapsed (3 total ports)
Nmap scan report for 10.216.75.115
Host is up, received arp-response (0.0021s latency).
Scanned at 2026-05-06 14:42:49 CST for 0s

PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 64
80/tcp   open  http       syn-ack ttl 64
8080/tcp open  http-proxy syn-ack ttl 63
MAC Address: 08:00:27:89:39:82 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.35 seconds
           Raw packets sent: 4 (160B) | Rcvd: 4 (160B)
```
开放`22`、`80`、`8080`端口
扫一下web端的目录：
```bash
┌──(kali㉿kali)-[~]
└─$ gobuster dir -u http://10.216.75.115 -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.216.75.115
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.htaccess            (Status: 403) [Size: 278]
.hta                 (Status: 403) [Size: 278]
.htpasswd            (Status: 403) [Size: 278]
index.html           (Status: 200) [Size: 6]
server-status        (Status: 403) [Size: 278]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
```
没有什么有用的东西
又更全面扫了一下，就放一下总结：
```md
┌──(kali㉿kali)-[~]
└─$ what --md --q "从渗透测试的角度总结分析"

──────────────────────────────────────────────────
$ rustscan -a 10.216.75.115 --ulimit 5000 -- -A -sC -sV  (exit: 0)
(原始 6385 字符，提炼后 prompt ~6567 字符)
──────────────────────────────────────────────────

命令作用

使用 RustScan 快速扫描目标 IP 10.216.75.115，限制文件描述符上限 5000，并将结果传给 Nmap 执行详细扫描（-A 全面扫描，-sC 默认脚本，-sV
版本探测）。

扫描结果

目标存活，共开放 3 个端口：


 端口      服务  版本                            备注
 ───────────────────────────────────────────────────────────────────────────────────────────────────────
 22/tcp    SSH   OpenSSH 8.4p1 Debian 5+deb11u3  支持三种密钥交换算法（RSA/ECDSA/ED25519）
 80/tcp    HTTP  Apache httpd 2.4.62 (Debian)    无标题，纯文本页面（text/html）
 8080/tcp  HTTP  Apache httpd 2.4.65 (Debian)    WordPress 6.9，robots.txt 禁止 /wp-admin/，可能开放代理


操作系统：Linux（内核 4.15~5.19 或 MikroTik RouterOS 7.x），运行于 VirtualBox 虚拟机（MAC 08:00:27:89:39:82）。

渗透测试角度分析

 1 攻击面
    • SSH (22)：版本较新（8.4p1），公开漏洞少，但可尝试弱密码爆破（hydra、medusa）或针对密钥的枚举。
    • HTTP (80)：Apache
      2.4.62，可能部署了未知应用（标题为空），需进一步查看页面源码、目录枚举（gobuster）或检查是否有文件包含、LFI、SQL 注入等。
    • HTTP (8080)：WordPress 6.9 是重点目标。可尝试：
       • 扫描插件/主题漏洞（wpscan）。
       • 枚举用户（wp-json/wp/v2/users）。
       • 利用 robots.txt 指向 /wp-admin/，可尝试弱密码登录。
       • 检查 http-open-proxy 提示：可能被配置为开放代理，可用于 SSRF 或内网探测。
 2 操作系统指纹
   推测为 Debian 或衍生产品（基于 Apache 版本号），但 OS 检测显示包含 RouterOS，需确认实际系统（可能被伪装或跑在虚拟机中）。
 3 网络位置
   ARP 响应（一跳，延迟 3.8ms）说明目标在同一广播域，可直接进行 ARP 欺骗或中间人攻击（若已获攻击机权限）。
 4 后续步骤建议
    • 全端口扫描（RustScan 默认只扫常见端口，可能漏掉其他高端口服务）。
    • 对 80 和 8080 执行详细目录/文件爆破，寻找隐藏路径。
    • 使用 wpscan 枚举 WordPress 漏洞。
    • 尝试 SSH 无密码登录（若存在密钥分发漏洞）。
    • 检查开放代理是否可被利用进行外网请求转发。

总结：目标暴露了三个服务，Web 应用（尤其是 WordPress）是主要突破口，SSH
可作为辅助攻击项。需谨慎对待开放代理可能性，可能用于内网横向移动。
```
8080端口是`WordPress`博客网站（一套开源的博客系统）
用wpscan扫描一下。
```md
┌──(kali㉿kali)-[~]
└─$ what --md --q "从渗透的角度总结并分析" --raw

──────────────────────────────────────────────────
$ wpscan --url http://10.216.75.115:8080/  (exit: 0)
──────────────────────────────────────────────────

wpscan 对 http://10.216.75.115:8080/ 进行了 WordPress 安全扫描。结果如下：

 • 基础信息：WordPress 6.9.4（最新版），Apache/2.4.65 + PHP/8.1.34。
 • 暴露信息：
    • robots.txt 暴露了 /wp-admin/ 和 /wp-admin/admin-ajax.php。
    • XML-RPC 已启用（/xmlrpc.php）。
    • readme.html 和调试日志 debug.log 可公开访问。
    • WP-Cron 已启用（/wp-cron.php）。
 • 主题：b2 4.2.6（第三方主题，版本稍低）。
 • 插件：仅发现一个名为 * 的插件（可能为通配符或路径泄露），版本未知。
 • 配置备份：未找到。

渗透角度分析：

 • 调试日志 debug.log 可能泄漏 PHP 错误、数据库凭证、路径等敏感信息，是高风险发现。
 • XML-RPC 存在可被用于暴力破解、DDoS（pingback 反射）、用户枚举的已知利用点。
 • robots.txt 提供了管理入口路径。
 • 第三方主题 b2 (4.2.6) 可能存在已知安全漏洞（需搜索 CVE 或公开 EXP）。
 • 未发现其他明显漏洞，但信息收集已足够启动进一步攻击：查看 debug.log、尝试 XML-RPC 登录爆破、针对主题版本进行漏洞利用。

```

```md
┌──(kali㉿kali)-[~]
└─$ what --md --raw -q "这是在做本地靶机的渗透测试，给我总结并分析"

──────────────────────────────────────────────────
$ wpscan --url http://10.216.75.115:8080/ --enumerate u  (exit: 0)
──────────────────────────────────────────────────

总结

命令 wpscan --url http://10.216.75.115:8080/ --enumerate u 是对目标 WordPress 站点进行安全扫描，并枚举用户账号。

结果：

 • 成功识别出 WordPress 6.9.4（最新版）。
 • 发现多个信息泄露点：robots.txt（暴露后台路径）、xmlrpc.php（已启用）、readme.html、debug.log、wp-cron.php。
 • 枚举到唯一用户：admin（通过 WP JSON API、RSS 生成器、暴力猜解等多种方式验证）。

------------------------------------------------------------------------------------------------------------------------

渗透测试分析

这是典型的信息收集阶段，已获得对后续攻击极有价值的线索：


 发现项         渗透测试价值
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 用户 admin     已知管理员用户名，可直接用于暴力破解（如 wpscan --passwords ...）或针对性弱口令猜测。
 debug.log      可能包含 SQL 错误、路径泄露、认证信息等敏感数据，建议直接访问查看。
 XML-RPC 启用   可发起密码爆破（利用 system.multicall 绕过登录尝试限制）、DDoS 或Pingback 反射攻击。
 robots.txt     明确给出了 /wp-admin/ 和 admin-ajax.php 路径，减少目标盲猜成本。
 主题 b2 4.2.6  可查询该版本已知漏洞（例如文件包含、XSS、SQL注入），同时注意是商业主题，可能包含未公开漏洞。
 WP-Cron 启用   可能被用于 CVE-2024-xxxx 等针对 WP-Cron 的拒绝服务或计划任务劫持攻击（需进一步探查版本漏洞）。


下一步建议：

 1 爆破 admin 密码：wpscan --url ... --passwords /usr/share/wordlists/rockyou.txt --usernames admin
 2 检查 debug.log：直接下载 http://10.216.75.115:8080/wp-content/debug.log 查看详细日志。
 3 利用 XML-RPC 爆破：使用 xmlrpc_multicall 模块加速爆破。
 4 搜索 b2 主题漏洞：针对 b2 4.2.6 进行公开 CVE 和 PoC 查询。
 5 扫描插件/漏洞：执行 wpscan --url ... -e vp 枚举已安装插件及已知漏洞。

该靶机配置符合典型低版本/默认设置 WordPress，存在管理员凭证泄露和配置不当的风险，成功爆破后可获得 Web Shell
或进一步提权。
```
看了一下日志
```md
┌──(kali㉿kali)-[~]
└─$ what --md --raw -q "这是在做本地靶机的渗透测试，给我总结并分析"

──────────────────────────────────────────────────
$ curl http://10.216.75.115:8080/wp-content/debug.log  (exit: 0)
──────────────────────────────────────────────────

命令做了什么

通过 curl 向 http://10.216.75.115:8080/wp-content/debug.log 发送 HTTP GET 请求，获取了该路径下的 debug.log 文件内容。

结果

成功获取了文件内容，输出是 WordPress 的调试日志，包含大量 PHP 警告、弃用通知、数据库错误。日志内容显示：

 • 重复的 PHP Warning: Trying to access array offset on value of type null 和 Array to string conversion（出自主题
   private8.1.php）。
 • 大量 Undefined variable $data（在 Orders.php）。
 • 数据库表 wp_b2_post_relationships 不存在导致 SQL 查询失败。
 • 数据库列 comment_id 不存在。
 • 表 wp_b2_gold 不存在。
 • PHP 弃用：隐式浮点转整型精度丢失。

渗透测试分析

这是一个典型的信息泄露 + 未授权访问场景：

 1 敏感文件暴露（高危）：/wp-content/debug.log 未加访问限制，任何人均可下载。该文件泄露了：
    • 站点物理路径：/var/www/html/wp-content/themes/b2/...
    • 使用的主题和插件：B2 主题具体模块路径。
    • 数据库表名结构：wordpress.wp_b2_post_relationships、wp_b2_gold 等。
    • 潜在的认证绕过点：如 rest_api_loaded、checkFollowing 等 REST API 端点，可能触发未授权操作。
 2 PHP 代码缺陷：大量未定义变量、null 数组访问，表明代码质量低，可作为进一步代码审计入口（如文件包含、SQL
   注入的辅助信息）。
 3 数据库缺失表/列：表 wp_b2_post_relationships 和列 comment_id
   不存在，可能是插件未正确安装或升级失败，导致功能异常，可作为服务可用性攻击或条件竞争利用点。
 4 弃用警告：Implicit conversion from float to int 提示 PHP 版本兼容问题（可能使用 PHP 8+ 运行旧代码）。

攻击者可以利用这些信息：识别 CMS 版本与漏洞、构造针对性 SQL 注入、利用未定义的变量污染、直接访问后台 REST API（如
/wp-json/b2/v1/ 等）。建议立即删除或禁止访问 debug.log，修复 PHP 代码错误并补齐缺失数据库表。

```