# 主机发现
```bash
┌──(kali㉿kali)-[~]
└─$ nmap 10.216.75.0/24 -sn
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-07 17:47 CST
Nmap scan report for 10.216.75.26
Host is up (0.00068s latency).
MAC Address: 08:00:27:56:B7:DE (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.216.75.183
Host is up (0.0043s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.216.75.212
Host is up (0.00027s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.216.75.80
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 5.89 seconds
```
靶机IP：`10.216.75.26`

# 信息收集
`rustscan`扫一下
```bash
┌──(kali㉿kali)-[~]
└─$ what --md --raw -q "从渗透测试方面命令执行结果总结，注意分点"

──────────────────────────────────────────────────
$ rustscan -a 10.216.75.26 --ulimit 5000 -- -A -sC -sV  (exit: 0)
──────────────────────────────────────────────────

从渗透测试角度总结命令执行结果：

 1 开放端口与服务
    • 22/tcp：OpenSSH 10.0p2 Debian（SSH服务）
    • 80/tcp：Apache httpd 2.4.66 Debian（HTTP服务），网页标题为“Maze”
 2 服务版本与操作系统
    • SSH和HTTP版本均较新（OpenSSH 10.0p2、Apache 2.4.66），可能存在已知漏洞需进一步验证。
    • OS推测为Linux（Debian系），但未精确匹配到具体发行版（条件不理想）。
    • 目标主机运行在VirtualBox虚拟机内（MAC地址为Oracle VirtualBox）。
 3 HTTP服务信息
    • 支持方法：POST、OPTIONS、HEAD、GET（无PUT/DELETE等危险方法）。
    • HTTP服务器头明确标识为Apache/2.4.66 (Debian)。
    • 页面标题“Maze”可能暗示Web应用类型或玩法（如CTF靶机名）。
 4 渗透攻击面
    • SSH：可尝试弱口令爆破、版本漏洞（需检查CVE），注意OpenSSH 10.0p2是否为最新安全版本。
    • HTTP：可进一步扫描目录、文件、常见Web漏洞（如文件包含、SQL注入、命令执行），因页面标题“Maze”可能指向迷宫类应用。
 5 其他发现
    • 目标回应快速（延迟0.87ms），本地网络环境。
    • 无关闭端口被发现，OS探测不完整，但仍显示Linux内核版本范围（4.15~5.19）。
    • Uptime极短（刚启动），可能是临时测试机或重启过。

建议后续步骤：

 • 对80端口进行目录枚举（dirb/gobuster）和Web漏洞扫描。
 • 尝试SSH弱口令（如root/toor、admin/admin等）或指定用户爆破。
 • 检查Apache/OpenSSH对应版本是否存在公开利用脚本。
```
开放` 22` \ `80` 端口，直接去web看看

