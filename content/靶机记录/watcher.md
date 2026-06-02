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
偶看错了，上面说的是
```
jyli normal 退出登录

⚠️ 权限不足！只有 admin 等级的用户才能访问系统管理面板。
```
那么也就是说真正限制的应该是`level`字段，我们改一下：
![](file-20260602083012780.png)
成功登录进来管理员页面了。
（这里最好是直接改浏览器的cookie，这样后面就不要一直改包了）
进来之后我们就随便点点点。
![](file-20260602083401742.png)
发现后台的全局设置都是一个`/api/settings/update`接口实现。这里测一下sql注入。
![](file-20260602083647418.png)
发现存在数据库语法报错，应该存在注入漏洞。
我们构造子查询。
![](file-20260602084701563.png)
发现不存在database()函数。那就可能不是mysql了。可能是sqlite。我们试试sqlite的函数。

![](file-20260602084754174.png)
成功回显sqlite版本。
下一步获取表明
```sql
SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table'
```
![](file-20260602084947700.png)
应该是在secret表中我们看看表结构
```sql
SELECT sql FROM sqlite_master WHERE type='table' AND name='secret')

```
![](file-20260602085408245.png)
存在secret字段的呢，直接查询
```sql
select secret from secret
```
![](file-20260602085538225.png)
得到账号密码。
`watcher:mazesec123q1231w!@#!@@#$`

![](file-20260602085749773.png)
`flag{user-c3949567202847f1ad8664095f0a94e4}`
# 提权
话不多说先传一个pspy监控
![](file-20260602092916310.png)
注意到/opt目录下有个脚本在自动执行。
但是我们没有权限看
![](file-20260602093026969.png)

pspy里还有一句这个
```bash
inotifywait -m -e create /home/watcher/uploads
```
问了一下ai：
实时监控 `/home/watcher/uploads` 目录，当有新文件或子目录创建时，立即输出事件信息。
说明如果uploads文件夹发生变化会触发一些东西。那这个uploads文件夹我们去看看。
开始里面什么也没有，我们随便创建一个文件
![](file-20260602093418722.png)
然后我们在注意一下pspy监控
![](file-20260602093550364.png)
检测到变化时会调用另外一个脚本，参数是我们写的文件，但是这个脚本我们看不到呀
