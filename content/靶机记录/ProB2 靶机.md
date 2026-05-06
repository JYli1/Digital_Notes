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