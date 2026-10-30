# 信息收集
## 端口扫描
```bash
┌──(kali㉿JYlover)-[/mnt/c/Users/15819]
└─$ nmap -p- 172.16.12.210
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-30 19:25 +0800
Nmap scan report for 172.16.12.210
Host is up (0.067s latency).
Not shown: 65524 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
2049/tcp  open  nfs
3389/tcp  open  ms-wbt-server
5900/tcp  open  vnc
33599/tcp open  unknown
33849/tcp open  unknown
34045/tcp open  unknown
36773/tcp open  unknown
59093/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 41.09 seconds
```
有vnc，应该是要vnc登录的。先去web看看

# web渗透
![](file-20261030192951486.png)
没看出什么来，扫一下目录：
```bash
┌  root@JYlover  bash  0ms 
└  /mnt/c/Users/15819  #  dirsearch -u http://172.16.12.210

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm | HTTP method: GET | Threads: 25 | Wordlist size: 12289

Target: http://172.16.12.210/

[19:30:44] Scanning: 
[19:30:45] 200 -   182B - /.env
[19:31:01] 200 -    1KB - /cgi-bin/test-cgi
[19:31:01] 200 -   820B - /cgi-bin/printenv
[19:31:07] 200 -    1KB - /index.html
[19:31:16] 403 -   316B - /server-status
[19:31:16] 403 -   316B - /server-status/

Task Completed

```
![](file-20261030193103070.png)
有一个env配置文件泄露了vnc凭证
直接登录:
```bash
┌──(kali㉿JYlover)-[/mnt/c/Users/15819]
└─$ vncviewer 172.16.12.210:5900
```
存在一个handover.txt泄露了desk用户密码，直接切换过去获得userflag
![](file-20261030193809572.png)

# 提权
![](file-20261030193842103.png)
也没有别的用户了，那肯定是直接提权了
一些常规的提权点没有发现，例如sudo权限，计划任务都没有可利用点

但是注意到web和handover.txt都同时出现了一个路径：
`/srv/backup`，而且到现在为止他都没被利用到那利用点可能就在这里。
![](file-20261030194747158.png)
所以 desk 能进目录、能执行里面的文件，**不能创建、不能改名、不能 chmod**。

如果一个目录对 desk 只给"进 + 执行"，这个设计意图是什么
答案肯定是由一个**以 root 运行的备份进程**往里面写，`desk` 作为运维账号只负责进去查看和校验，所以给 `r-x` 不给 `w`。

也就是说：**这个目录的合法写入者是 root，而 desk 的角色是执行东西的人。**


![](file-20261030195935357.png)
这里就可以猜到，就是说允许网络中其他的机器挂载这个共享文件，并且有root权限去写，然后我们就是需要能够root写一个程序来给我们执行，所以这里可以利用
![](file-20261030202541971.png)
创建一个目录挂载上去，我们创建一个文件夹，然后vnc上确定一下
![](file-20261030202646475.png)
果然出现了，那就说明我们有写权限了，而且是root权限，剩下的就简单了
