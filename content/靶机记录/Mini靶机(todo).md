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
┌──(kali㉿kali)-[~]
└─$ rustscan -a 10.216.75.108 --greppable
10.216.75.108 -> [22,80,9000,9001]

┌──(kali㉿kali)-[~]
└─$ nmap 10.216.75.108 -p 22,80,9000,9001
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-03 17:50 CST
Nmap scan report for 10.216.75.108
Host is up (0.0023s latency).

PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
9000/tcp open  cslistener
9001/tcp open  tor-orport
MAC Address: 08:00:27:79:C1:03 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.32 seconds

```
发现四个端口，我们先看看web端

# 目录扫描
