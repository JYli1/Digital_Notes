# 主机发现

```bash
┌──(root㉿kali)-[/home/kali]
└─# nmap -sn 10.241.108.0/24 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-01 18:06 CST
Nmap scan report for 10.241.108.43
Host is up (0.0070s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.241.108.62
Host is up (0.0013s latency).
MAC Address: 08:00:27:E8:5C:CB (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.241.108.72
Host is up (0.15s latency).
MAC Address: 98:2C:BC:40:09:7F (Intel Corporate)
Nmap scan report for 10.241.108.212
Host is up (0.00052s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.241.108.201
Host is up.
Nmap done: 256 IP addresses (5 hosts up) scanned in 11.63 seconds

```
靶机ip：`10.241.108.62`

# 端口扫描

