# 信息收集
```bash
┌──(kali㉿JYlover)-[~]
└─$ nmap -p- 192.168.56.109
Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-05 14:35 +0800
Nmap scan report for 192.168.56.109
Host is up (0.020s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 32.75 seconds
```
去web端看看
# web渗透
一个登录页面，试了试弱口令试出来`admin`/`admin`
进来一个输入框也不知道是什么
