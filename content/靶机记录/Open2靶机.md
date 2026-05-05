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
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-05 23:20 CST
Nmap scan report for 10.216.75.183
Host is up (0.0039s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.216.75.212
Host is up (0.0014s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.216.75.80
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 15.10 seconds

```
确定靶机IP：
``