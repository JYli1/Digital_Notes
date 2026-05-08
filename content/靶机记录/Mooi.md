# 主机发现
```bash
┌──(kali㉿kali)-[~]
└─$ nmap 10.216.75.0/24 -sn
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-08 12:58 CST
Nmap scan report for 10.216.75.183
Host is up (0.0070s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.216.75.212
Host is up (0.0019s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.216.75.251
Host is up (0.00042s latency).
MAC Address: 08:00:27:A7:CC:59 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.216.75.81
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 10.80 seconds
```
靶机IP：`10.216.75.251`

# 信息收集
```md
┌──(kali㉿kali)-[~]
└─$ what

──────────────────────────────────────────────────
$ rustscan -a 10.216.75.251 --ulimit 5000 -- -A -sC -sV  (exit: 0)
──────────────────────────────────────────────────

结论

目标主机 10.216.75.251 是一台运行着 SSH、两个 HTTP 服务的 Linux 系统，其中 8080 端口标题暗示可能有备份系统或内部应用，值得深入探查。

关键发现


 端口      服务  版本/详情
 ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 22/tcp    SSH   OpenSSH 8.4p1 Debian 5+deb11u3，支持 RSA/ECDSA/ED25519 密钥
 80/tcp    HTTP  Apache 2.4.62 (Debian)，标题“CYBER MINESWEEPER - 赛博扫雷”，支持 GET/HEAD/POST/OPTIONS
 8080/tcp  HTTP  Apache 2.4.62 (Debian)，标题“Internal Backup System”，同样支持 GET/HEAD/POST/OPTIONS，且被检测到 http-open-proxy
                 风险（可能被重定向）


 • MAC 地址：08:00:27:A7:CC:59（Oracle VirtualBox），确认是虚拟靶机。
 • OS 猜测：Linux 4.15–5.19 / OpenWrt / RouterOS，但未完全确定。
 • HTTP 信息：80 端口标题为中文“赛博扫雷”，可能是一个游戏或挑战页面；8080 端口标题“Internal Backup
   System”暗示可能存在备份文件泄露或管理后台。

下一步建议

 1 Web 应用枚举：分别对 80 和 8080 端口进行目录扫描、文件发现、技术栈识别。标题中的“赛博扫雷”可能包含 LFI/RFI、SQLi 或命令注入点；8080
   的“备份系统”可能存在默认凭证、备份文件下载或未授权访问。
 2 检查 http-open-proxy：8080 端口被标记为可能的重定向代理，需确认其行为（例如能否用作开放代理访问内网）。
 3 SSH 测试：尝试弱口令爆破（仅授权下）或利用已知 CVE（OpenSSH 8.4p1 存在某些漏洞如 CVE-2021-41617，但需确认补丁情况）。
 4 操作系统指纹确认：通过更多主动探测（如 TCP/IP 栈指纹、ICMP 响应）细化 OS 版本，帮助选择攻击向量。
 5 内网渗透：若当前环境为内网，可进一步探测该主机是否与其他机器通信（例如通过代理扫描或 ARP 广播）。

下一步枚举/验证命令（仅限授权环境）

1. 对 80 端口进行目录和文件扫描


 # 使用 dirsearch 或 gobuster 扫描常用路径
 gobuster dir -u http://10.216.75.251:80 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,zip,backup
 -t 50


2. 对 8080 端口进行目录扫描（注意可能是备份系统）


 gobuster dir -u http://10.216.75.251:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x
 php,txt,html,zip,sql,bak -t 50


3. 检查 8080 端口是否可作为开放代理


 # 测试代理重定向（若返回 200 或 302 则可能可用）
 curl -x http://10.216.75.251:8080 http://httpbin.org/ip -v --connect-timeout 5


4. 指纹识别和页面分析


 # 使用 whatweb 识别技术栈
 whatweb http://10.216.75.251:80
 whatweb http://10.216.75.251:8080

 # 或者使用 curl 查看响应头
 curl -I http://10.216.75.251:80
 curl -I http://10.216.75.251:8080


5. 针对“赛博扫雷”可能存在的注入点测试（手动）


 # 尝试常见的 LFI 参数
 http://10.216.75.251:80/?page=/etc/passwd
 http://10.216.75.251:80/?file=../../../../../etc/passwd


6. SSH 弱口令尝试（谨慎：仅限授权环境）


 # 使用 hydra 对 SSH 进行常见用户名/密码爆破
 hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://10.216.75.251 -t 4 -V -f


7. 操作系统进一步探测


 # 使用 nmap 的 -O 选项加上更全面的扫描（避免被防火墙禁止）
 nmap -O --osscan-guess 10.216.75.251


▌ ⚠️ 所有命令必须在拥有合法授权的渗透测试环境中执行，禁止在未授权系统上使用。

```
