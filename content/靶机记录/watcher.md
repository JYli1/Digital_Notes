```bash
┌──(kali㉿JYlover)-[~]
└─$ nmap -p- 192.168.56.104
Starting Nmap 7.98 ( https://nmap.org ) at 2026-06-02 01:20 +0800
Nmap scan report for 192.168.56.104
Host is up (0.0032s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp

Nmap done: 1 IP address (1 host up) scanned in 24.02 seconds
```
发现22，5000
## web渗透
注册了一个账号进来，点了一下系统管理，发现要admin权限，我们看看是不是有办法登录admin账号的
![](file-20260602081353039.png)
抓包看看：
![](file-20260602081517737.png)
发现cookie存在jwt验证。解密看看：
![](file-20260602081914574.png)
payload中存在`username`
那也就是说可能可以把`username`换成`admin`。
并且是HS256加密，那我们尝试爆破一手密钥。
![](file-20260602082449229.png)
密钥是`maze`
![](file-20260602082617195.png)
直接到官网来，把username字段修改为admin。然后抓包改包试试。
