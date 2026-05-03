虽然可以看到靶机ip，但是就当作看不到吧。发现一下ip。
# 主机发现
```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sn 22 10.216.75.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-03 17:39 CST
Nmap scan report for 10.216.75.72
Host is up (0.011s latency).
MAC Address: 98:2C:BC:40:09:7F (Intel Corporate)
Nmap scan report for 10.216.75.108
Host is up (0.0065s latency).
MAC Address: 08:00:27:79:C1:03 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.216.75.183
Host is up (0.027s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.216.75.212
Host is up (0.0011s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.216.75.80
Host is up.
Nmap done: 257 IP addresses (5 hosts up) scanned in 5.85 seconds
```
* ip : `10.216.75.108`
# 端口扫描
```bash

```