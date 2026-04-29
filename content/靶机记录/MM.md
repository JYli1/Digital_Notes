---
靶机id: 大傻子
作者: "638"
---
## 主机发现
```bash
┌──(root㉿kali)-[/home/kali]
└─# nmap -sn 10.241.108.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-29 14:16 CST
Nmap scan report for 10.241.108.43
Host is up (0.062s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.241.108.212
Host is up (0.00080s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.241.108.244
Host is up (0.0020s latency).
MAC Address: 08:00:27:FC:21:A8 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.241.108.201
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 37.26 seconds
```
发现靶机ip：`10.241.108.201`
## 端口扫描
```zsh
┌──(root㉿kali)-[/home/kali]
└─# nmap -p- 10 10.241.108.244
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-29 14:19 CST
Nmap scan report for 10.241.108.244
Host is up (0.0016s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5901/tcp open  vnc-1
6001/tcp open  X11:1
MAC Address: 08:00:27:FC:21:A8 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 2 IP addresses (1 host up) scanned in 19.29 seconds
```
除了22端口还发现了`80`、`5901`、`6001`
