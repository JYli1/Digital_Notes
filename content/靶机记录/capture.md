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
## /var/captures
发现/var/captures下有两个流量包
![](file-20261103155907831.png)
## /var/log/messages
/var/log/messages是一份系统日志
```http
HTTP/1.1 200 OK
Date: Tue, 03 Nov 2026 08:00:52 GMT
Server: uhttpd/2018-03-01, LuCI/0.10.0
Content-Type: application/json
Connection: keep-alive
Content-Length: 2418

{"result":{"1~
May 24 08:00:01 OpenWrt syslog.info syslogd started: BusyBox v1.28.4
May 24 08:00:01 OpenWrt user.notice kernel: klogd started: BusyBox v1.28.4 (2019-05-13 22:04:11 UTC)
May 24 08:00:02 OpenWrt user.info kernel: [    0.000000] Linux version 4.14.120 (openwrt@buildhost) (gcc version 7.4.0 (OpenWrt GCC 7.4.0 r7828-ef3b437d7c)) #0 SMP Fri May 17 11:22:33 2019
May 24 08:00:03 OpenWrt daemon.info procd: - init complete -
May 24 08:00:05 OpenWrt daemon.info dnsmasq[814]: started, version 2.80 cachesize 150
May 24 08:00:05 OpenWrt daemon.info dnsmasq[814]: reading /tmp/resolv.conf.auto
May 24 08:00:08 OpenWrt cron.err crond[712]: crond (busybox 1.28.4) started, log level 5
May 24 08:00:12 OpenWrt authpriv.info dropbear[905]: Running in background
May 24 08:00:14 OpenWrt user.notice netflow-cron: collector ready (interval 30m, retention 7d, outdir /var/captures)
May 24 09:02:44 OpenWrt authpriv.info dropbear[1671]: Child connection from 192.168.99.100:51234
May 24 09:02:47 OpenWrt authpriv.info dropbear[1671]: Password auth succeeded for 'root' from 192.168.99.100:51234
May 24 09:11:52 OpenWrt user.notice netflow-cron: manual capture requested by root (tty pts/0)
May 24 09:12:03 OpenWrt user.notice netflow-cron: capture started on br-lan (snaplen 128, filter 'ip')
May 24 09:12:41 OpenWrt user.notice netflow-cron: flush 1096 packets, 11145411 bytes -> /var/captures/mgmt-20190524-0912.pcap
May 24 09:30:52 OpenWrt user.notice netflow-cron: manual capture requested by root (tty pts/0)
May 24 09:31:07 OpenWrt user.notice netflow-cron: capture started on br-lan (snaplen 128, filter 'ip')
May 24 09:31:58 OpenWrt user.notice netflow-cron: flush 2144 packets, 21386309 bytes -> /var/captures/mgmt-20190524-0931.pcap
May 24 09:32:01 OpenWrt user.notice netflow-cron: retention sweep: 0 files removed (oldest kept: 7d)
May 24 09:32:04 OpenWrt daemon.info dnsmasq[814]: DHCPDISCOVER(br-lan) 192.168.99.53 aa:bb:cc:00:11:53
May 24 09:32:06 OpenWrt user.info netflow-analyzer: diagnostics pack rebuilt after config change, staged in /www/files
May 24 09:58:11 OpenWrt user.notice netflow-cron: manual capture requested by root (tty pts/0)
May 24 09:58:19 OpenWrt user.notice netflow-cron: flush 0 packets, 0 bytes -> /var/captures/mgmt-20190524-0958.pcap
May 24 09:58:19 OpenWrt user.err netflow-cron: capture aborted: no traffic on br-lan, file removed
~":{"rx":"301056 B/s","tx":"281600 B/s"}}}

```
