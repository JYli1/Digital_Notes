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
现扫一下目录：
```ps
    root@JYlover   bash   17ms 
  ~/result/a7505d1fd44669cabd1512a909828426/class   master ●    dirsearch -u http://192.168.56.109/

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm | HTTP method: GET | Threads: 25 | Wordlist size: 12289

Target: http://192.168.56.109/

[15:58:08] Scanning:
[15:58:11] 301 -   355B - /.git  ->  http://192.168.56.109/.git/
[15:58:11] 200 -    3KB - /.git/
[15:58:11] 200 -   240B - /.git/info/exclude
[15:58:11] 200 -    31B - /.git/COMMIT_EDITMSG
[15:58:11] 200 -   198B - /.git/config
[15:58:11] 200 -    1KB - /.git/logs/
[15:58:11] 200 -   194B - /.git/logs/HEAD
[15:58:11] 200 -    73B - /.git/description
[15:58:11] 200 -    23B - /.git/HEAD
[15:58:11] 200 -   194B - /.git/logs/refs/heads/master
[15:58:11] 200 -    4KB - /.git/hooks/
[15:58:11] 200 -    54B - /.git/objects/info/packs
[15:58:11] 200 -   105B - /.git/packed-refs
[15:58:11] 200 -   751B - /.git/index
[15:58:11] 200 -    1KB - /.git/info/
[15:58:11] 200 -    59B - /.git/info/refs
[15:58:11] 301 -   365B - /.git/logs/refs  ->  http://192.168.56.109/.git/logs/refs/
[15:58:11] 301 -   371B - /.git/logs/refs/heads  ->  http://192.168.56.109/.git/logs/refs/heads/
[15:58:11] 200 -    1KB - /.git/objects/
[15:58:11] 200 -    1KB - /.git/refs/
[15:58:11] 301 -   366B - /.git/refs/heads  ->  http://192.168.56.109/.git/refs/heads/
[15:58:11] 301 -   365B - /.git/refs/tags  ->  http://192.168.56.109/.git/refs/tags/
[15:58:11] 200 -    20B - /.gitignore
CTRL+C detected: Pausing threads, please wait...
[q]uit / [c]ontinue: qq
[q]uit / [c]ontinue: q
[s]ave / [q]uit without saving: q

Canceled by the user

```
很多git文件，Githacker拉下来
```bash

```