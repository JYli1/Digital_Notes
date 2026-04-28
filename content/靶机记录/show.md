作者：大傻子
靶机id：652

## 主机发现
```zsh
┌──(kali㉿kali)-[~/桌面]
└─$ nmap -sn 10.241.108.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-28 16:51 CST
Nmap scan report for 10.241.108.8
Host is up (0.0012s latency).
MAC Address: 08:00:27:35:21:83 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.241.108.43
Host is up (0.011s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.241.108.212
Host is up (0.00060s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.241.108.201
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 8.08 seconds
```
确定靶机ip：`10.241.108.8`(后来发现给了ip)

## 端口扫描
```zsh
┌──(kali㉿kali)-[~/桌面]
└─$ nmap   10.241.108.8 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-28 16:54 CST
Nmap scan report for 10.241.108.8
Host is up (0.0045s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:35:21:83 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.62 seconds
                                                             
```
* 存在80-web端口
## 目录扫描
发现了web就先访问了一下
![](file-20260428170713217.png)
发现是一个`ShowDoc`的网站，应该是一套通用的源码。
