# 信息收集
```bash

┌──(kali㉿JYlover)-[/mnt/d/github_project/bajitmp]
└─$ nmap -p- 10.20.3.72
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-03 11:06 +0800
Nmap scan report for bogon (10.20.3.72)
Host is up (0.0017s latency).
Not shown: 65530 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
2222/tcp open  EtherNetIP-1
8080/tcp open  http-proxy
8443/tcp open  https-alt

Nmap done: 1 IP address (1 host up) scanned in 8.33 seconds
```

扫描一下端口，发现8080开着，80端口进去是一个欢迎页面，暂时没有什么信息
```bash
┌──(kali㉿JYlover)-[/mnt/d/github_project/bajitmp]
└─$ curl http://10.20.3.72
Hello, MazeSec!
```

# web渗透
进来8080.

![](file-20261103154542366.png)
提示我们有一个``/var/log/messages``文件，肯定是得登录的
目录扫描一下：
```bash
┌  root@JYlover  bash  38.507s 
└  ~/webtool/Dirsearch  #  dirsearch -u http://10.20.3.72:8080/

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm | HTTP method: GET | Threads: 25 | Wordlist size: 12289

Target: http://10.20.3.72:8080/

[15:17:00] Scanning: 
[15:17:08] 403 -   181B - /adminjsp
[15:17:08] 403 -   181B - /admin-ajax.php
[15:17:08] 403 -   181B - /admin-ANTIGO
......
[15:17:12] 403 -   181B - /adminxxx
[15:17:18] 403 -   122B - /export/
[15:17:18] 403 -   122B - /export
[15:17:18] 403 -   122B - /files
[15:17:18] 403 -   122B - /files/
[15:17:20] 200 -    3KB - /index.html

Task Completed

```
中间省略了很多admin开头的文件，没有什么规律。
主要是在后面的`/export`和`/files`目录，他们的响应大小也不同，且403说明文件夹存在只是我们没有权限

这里我也不知道ai怎么猜出来的一个**export/index.txt**和**files/readme.txt**文件，想不通
```bash
┌  root@JYlover  bash  32.9s 
└  ~/webtool/Dirsearch  #  curl http://10.20.3.72:8080/export/index.txt
netflow-analyzer export index
=============================
generated : 2019-05-24 09:32:10 UTC
collector : netflow-cron
outdir    : /var/captures
retention : 7 days

Scheduled rollups are registered here by "netflow-cron --register"
so the reporting host can pull them through the download CGI.
Manual runs write straight into outdir and stay unregistered.

(none)


生成时间：2019-05-24 09:32:10 UTC
收集器：netflow-cron
输出目录：/var/captures
保留期：7 天

定期汇总任务由“netflow-cron --register”注册，报告主机可以通过下载 CGI 拉取这些汇总任务。手动运行的结果直接写入输出目录，并保持未注册状态。

（无注册任务）
```

```bash
┌  root@JYlover  bash  67ms 
└  ~/webtool/Dirsearch  #  curl http://10.20.3.72:8080/files/readme.txt
netflow-analyzer diagnostics dump
=================================
build    : 0.9.4 (2019-05-24)
collector: netflow-cron, 30 min rotation, 7 day retention
archive  : /var/captures
export   : /www/export/index.txt
images   : this directory (sysupgrade image + diagnostic blob)


netflow-analyzer 诊断转储
=================================
构建版本：0.9.4 (2019-05-24)
收集器：netflow-cron，30 分钟轮换，7 天保留
归档：/var/captures
导出：/www/export/index.txt
图片：本目录（系统升级镜像 + 诊断数据块）

```
这里就知道有一个/var/captures文件，

然后我们尝试登录（密码也是ai猜出来的）
由于两个文件都提到了netflow，且构建时间为2019年
```
root/netflow2019       
```
登录后会获得一个cookie，并且跳转到一个js文件重复执行查看网卡流量。
![](file-20261103154905413.png)
![](file-20261103154923731.png)

这里存在一个**CVE-2019-12272**
![](file-20261103155513575.png)
拿到webshell，但是找了一下都没有找到flag，所以先去看看上面获取到的一些文件

