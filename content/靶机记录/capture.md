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
```py
"""
OpenWrt 靶场 - WRT 容器 Web 服务（uhttpd + LuCI 0.10 复刻）
==========================================================
复刻 OpenWrt 18.06 时代的 uhttpd + LuCI 0.10 管理界面行为：

  公开（无需认证）：
    GET  /                                  LuCI 登录页（干净页面，仅版本指纹）
    GET  /luci/...  /files/...              静态文件（/files 为公开下载目录）
    GET  /cgi-bin/cgi-download?path=        【漏洞A】cgi-io 下载端点
                                            （ACL 用未规范化路径做 fnmatch，
                                             可被 ../ 绕过 —— CVE-2026-62947 同构）

  需要会话（LuCI sysauth）：
    POST /cgi-bin/luci/                     登录（luci_username/luci_password）
    GET  /admin/status/overview             登录后的状态页（带宽组件调用漏洞端点）
    GET  /admin/status/realtime/bandwidth_status?iface=
    GET  /cgi-bin/luci/bandwidth_status?iface=
                                            【漏洞B】CVE-2019-12272 命令注入
    GET  /cgi-bin/luci/backup/download?file=  系统备份下载 -> /var/backups/
    GET  /admin/ssh_keys                    SSH 密钥管理页（TLS 加密后才可见）
    POST /cgi-bin/luci/admin/ssh_keys       上传 SSH 密钥 -> /var/keys/

  TLS 参数刻意复刻 2019 年设备：TLS1.2 + TLS_RSA_WITH_AES_128_CBC_SHA
  （静态 RSA 密钥交换，无前向保密；关闭 EMS / EtM），因此有服务器私钥
  就能事后解密整段会话 —— 这正是本题的密码学考点。

纯标准库实现，容器内外均可运行（本地调试见 README）。
"""
import fnmatch
import json
import os
import re
import secrets
import ssl
import stat as statmod
import subprocess
import sys
import threading
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from pathlib import Path
from urllib.parse import unquote, urlparse, parse_qs

DOCROOT = os.environ.get("LUCI_DOCROOT", "/www")
CGI_DIR = os.environ.get("LUCI_CGI_DIR", str(Path(__file__).parent / "cgi"))
BACKUP_DIR = os.environ.get("LUCI_BACKUP_DIR", "/var/backups")
KEYSTORE_DIR = os.environ.get("LUCI_KEYSTORE_DIR", "/var/keys")
ACL_DIR = os.environ.get("LUCI_ACL_DIR", "/usr/share/rpcd/acl.d")

WEB_USER = os.environ.get("LUCI_USER", "root")
WEB_PASSWORD = os.environ.get("LUCI_PASSWORD", "netflow2019")

SERVER_BANNER = "uhttpd/2018-03-01, LuCI/0.10.0"
SESSION_TTL = 3600
SESSIONS = {}                      # token -> {"user":.., "ts":..}


def find_bash():
    """定位可用的 bash：环境变量 > Linux /bin/bash > Windows Git Bash
    （跳过 System32/WindowsApps 里的 WSL 存根）"""
    env_bin = os.environ.get("BASH_BIN")
    if env_bin and os.path.exists(env_bin):
        return env_bin
    if os.path.exists("/bin/bash"):
        return "/bin/bash"
    skip = ("system32", "windowsapps")
    for d in os.environ.get("PATH", "").split(os.pathsep):
        dl = d.lower()
        if any(s in dl for s in skip):
            continue
        for name in ("bash.exe", "bash"):
            p = os.path.join(d, name)
            if os.path.isfile(p):
                return p
    return "bash"


BASH = os.environ.get("BASH_BIN") or find_bash()
HTTP_PORT = int(os.environ.get("HTTP_PORT", "80"))
HTTPS_PORT = int(os.environ.get("HTTPS_PORT", "443"))
CERT = os.environ.get("LUCI_CERT", str(Path(__file__).parent / "tls" / "server.crt"))
KEY = os.environ.get("LUCI_KEY", str(Path(__file__).parent / "tls" / "server.key"))


def log(msg):
    print("[%s] %s" % (time.strftime("%H:%M:%S"), msg), flush=True)


def new_session(user):
    token = secrets.token_hex(16)
    SESSIONS[token] = {"user": user, "ts": time.time()}
    return token


def session_user(self):
    """从 Cookie 头解析 sysauth 会话，返回用户名或 None"""
    raw = self.headers.get("Cookie", "")
    m = re.search(r"(?:^|;\s*)sysauth=([0-9a-f]{16,})", raw)
    if not m:
        return None
    sess = SESSIONS.get(m.group(1))
    if not sess or time.time() - sess["ts"] > SESSION_TTL:
        return None
    return sess["user"]


LOGIN_PAGE = (Path(__file__).parent / "www" / "index.html")


def file_acls(role="netflow"):
    """读取 rpcd 的 ACL 定义，返回该 role 可读的 file glob 列表。

    真实路径是 /usr/share/rpcd/acl.d/*.json，schema 形如：
        {"netflow": {"read": {"file": {"/www/export/*": ["read"]}}}}
    cgi-io 拿到 path 后调 ubus 的 session.access 做校验，rpcd 侧用 fnmatch()
    匹配。**注意 fnmatch 未传 FNM_PATHNAME**，于是 `*` 会跨 `/` 匹配
    —— 这正是 CVE-2026-62947 能被绕过的原因之一。
    """
    globs = []
    acl_dir = Path(ACL_DIR)
    if not acl_dir.is_dir():
        return globs
    for f in sorted(acl_dir.glob("*.json")):
        try:
            data = json.loads(f.read_text(encoding="utf-8"))
        except (OSError, ValueError):
            continue
        if not isinstance(data, dict):
            continue
        for name, spec in data.items():
            if name not in (role, "*") or not isinstance(spec, dict):
                continue
            files = (spec.get("read") or {}).get("file") or {}
            if not isinstance(files, dict):
                continue
            for pattern, perms in files.items():
                perms = perms if isinstance(perms, (list, tuple)) else [perms]
                if any(p in ("read", "*") for p in perms):
                    globs.append(pattern)
    return globs


def login_html(error=False):
    html = LOGIN_PAGE.read_text(encoding="utf-8") if LOGIN_PAGE.is_file() else \
        "<html><body>Authorization Required</body></html>"
    if error:
        html = html.replace('<div class="error" id="err"></div>',
                            '<div class="error" id="err">Invalid username or password.</div>')
    return html.encode()


class Handler(BaseHTTPRequestHandler):
    server_version = "uhttpd"
    sys_version = ""
    protocol_version = "HTTP/1.1"

    # ------------------------------------------------------------ 基础
    def log_message(self, fmt, *args):
        pass

    def version_string(self):
        return ""

    def log_request(self, code="-", size="-"):
        log('%s - "%s %s" %s' % (self.client_address[0], self.command,
                                 self.path, code))

    def send_response(self, code, message=None):
        self.log_request(code)
        self.send_response_only(code, message)
        self.send_header("Date", self.date_time_string())

    def _send(self, code, body=b"", ctype="text/html; charset=UTF-8",
              extra=None):
        if isinstance(body, str):
            body = body.encode()
        self.send_response(code)
        self.send_header("Server", SERVER_BANNER)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(body)))
        self.send_header("Connection", "keep-alive")
        for k, v in (extra or {}).items():
            self.send_header(k, v)
        self.end_headers()
        if self.command != "HEAD":
            self.wfile.write(body)

    def _static(self, rel):
        rel = unquote(rel.split("?")[0])
        root = Path(DOCROOT).resolve()
        path = (root / rel).resolve()
        if not str(path).startswith(str(root)):
            return False
        if path.is_dir():
            # uhttpd 配 option no_dirlists 1：不列目录内容，直接 403
            # （默认 0 会输出目录列表；这里按"加固过的设备"设定，与
            #  /etc/config/uhttpd 里的 no_dirlists '1' 一致）
            self._send(403,
                       "<html><head><title>403 Forbidden</title></head>"
                       "<body><h1>403 Forbidden</h1>"
                       "<p>Directory index forbidden.</p></body></html>",
                       "text/html; charset=UTF-8")
            return True
        if not path.is_file():
            return False
        ctype = {".html": "text/html; charset=UTF-8", ".css": "text/css",
                 ".js": "application/javascript", ".png": "image/png",
                 ".ico": "image/x-icon", ".bin": "application/octet-stream",
                 ".txt": "text/plain; charset=UTF-8"}.get(path.suffix,
                                                          "application/octet-stream")
        self._send(200, path.read_bytes(), ctype)
        return True

    def _forbidden(self):
        body = ("<html><head><title>403 Forbidden</title>"
                "<meta http-equiv='refresh' content='0;url=/'></head>"
                "<body><h1>403 Forbidden</h1>"
                "<p>Access denied - authentication required.</p></body></html>")
        self._send(403, body)

    # ------------------------------------------------------------ CGI
    def _run_cgi(self, script, params):
        """执行 bash CGI：stdout = CGI 头 + 空行 + body（支持二进制）"""
        script_path = Path(CGI_DIR) / script
        if not script_path.is_file():
            self._send(404, "404 Not Found\n", "text/plain")
            return
        env = dict(os.environ)
        env.update({
            "QUERY_STRING": params,
            "REQUEST_METHOD": self.command,
            "REMOTE_ADDR": self.client_address[0],
            "LUCI_DOCROOT": DOCROOT.replace("\\", "/"),
            "SCRIPT_NAME": self.path.split("?")[0],
            "PATH": CGI_DIR.replace("\\", "/") + os.pathsep + env.get("PATH", ""),
        })
        try:
            proc = subprocess.run([BASH, str(script_path).replace("\\", "/")],
                                  env=env, capture_output=True, timeout=20)
            out = proc.stdout
            if proc.returncode != 0 and not out:
                out = b"CGI failed (rc=%d): %s" % (proc.returncode,
                                                   proc.stderr[:200])
                self._send(500, out, "text/plain")
                return
        except Exception as e:
            self._send(500, "CGI error: %s" % e, "text/plain")
            return
        idx_crlf = out.find(b"\r\n\r\n")
        idx_lf = out.find(b"\n\n")
        cands = [i for i in (idx_crlf, idx_lf) if i >= 0]
        if cands:
            split_at = min(cands)
            sep_len = 4 if split_at == idx_crlf else 2
            head, body = out[:split_at], out[split_at + sep_len:]
        else:
            head, body = b"", out
        headers = {}
        for line in head.decode("latin-1").splitlines():
            if ":" in line:
                k, v = line.split(":", 1)
                headers[k.strip().lower()] = v.strip()
        code = 200
        if "status" in headers:
            try:
                code = int(headers["status"].split()[0])
            except Exception:
                code = 500
        ctype = headers.get("content-type", "text/html; charset=UTF-8")
        extra = {}
        if "content-disposition" in headers:
            extra["Content-Disposition"] = headers["content-disposition"]
        self._send(code, body, ctype, extra)

    # ------------------------------------------------- cgi-io: cgi-download
    def _cgi_error(self, code, message):
        """cgi-io 的错误响应体是 blobmsg 风格的 JSON"""
        body = json.dumps({"error": code, "message": message}).encode()
        self._send(code, body, "application/json")

    def _cgi_download(self, params):
        """cgi-io 的下载端点：GET /cgi-bin/cgi-download?sessionid=&path=&filename=

        按 CVE-2026-62947 的根因实现（OpenWrt cgi-io / rpcd 的真实缺陷）：
          1. ACL 校验用的是**未规范化**的原始 path，且 fnmatch 不带 FNM_PATHNAME
             —— `*` 跨 `/`，`/www/export/*` 能匹配 `/www/export/../../etc/shadow`
          2. 校验通过后，stat()/open() 仍用**同一个未规范化路径**
             —— 内核解析 `..`，实际打开的是 ACL 范围之外的文件

        也就是说：ACL 是"字符串层面"的边界，文件系统是"语义层面"的边界，
        两者不一致 → 穿越。正确修法是先 canonicalize 再校验（25.12.5 的做法）。
        """
        # 字段名与真实 cgi-io 的 main_download() 一致：
        #   fields = { "sessionid", "path", "filename", "mimetype" }
        form = parse_qs(params, keep_blank_values=True)
        path = (form.get("path") or [""])[0]
        filename = (form.get("filename") or [""])[0]
        sid = (form.get("sessionid") or [""])[0]

        # 真实源码：!session_access(sid, "cgi-io", "download", "read")
        #              -> failure(403, "Download permission denied")
        # 本靶场：匿名请求回落到内置的 netflow 只读身份（见 docs/CVE_NOTES.md）
        role = "netflow"
        globs = file_acls(role)
        if not globs:
            return self._cgi_error(403, "Download permission denied")

        # 真实源码：!session_access(sid, "file", path, "read")
        #            -> failure(403, "Access to path denied by ACL")
        # ---- 漏洞点 1：未经 canonicalize 的字符串匹配 ----
        if not path or not any(fnmatch.fnmatchcase(path, g) for g in globs):
            log("cgi-download: DENY  %s (acl=%s)" % (path or "-", ",".join(globs)))
            return self._cgi_error(403, "Access to path denied by ACL")

        # 真实源码：if (stat(path, &s)) -> failure(404, errno, "Failed to stat ...")
        # ---- 漏洞点 2：同一个未规范化路径直接交给 stat()/open() ----
        try:
            st = os.stat(path)
        except OSError:
            log("cgi-download: ESTAT %s" % path)
            return self._cgi_error(404, "Failed to stat requested path")

        # 真实源码：if (!S_ISREG && !S_ISBLK)
        #            -> failure(403, "Requested path is not a regular file or block device")
        if not (statmod.S_ISREG(st.st_mode) or statmod.S_ISBLK(st.st_mode)):
            return self._cgi_error(
                403, "Requested path is not a regular file or block device")

        try:
            data = Path(path).read_bytes()
        except OSError:
            return self._cgi_error(403, "Download failed")
        name = filename or os.path.basename(path.rstrip("/")) or "download"
        log("cgi-download: GRANT %s -> %s (%d bytes, sid=%s)"
            % (self.client_address[0], path, len(data), sid or "-"))
        self._send(200, data, "application/octet-stream",
                   {"Content-Disposition": 'attachment; filename="%s"' % name})

    # ------------------------------------------------------------ 登录
    def _do_login(self, body):
        form = parse_qs(body.decode("latin-1", "replace"))
        user = (form.get("luci_username") or [""])[0]
        pw = (form.get("luci_password") or [""])[0]
        if user == WEB_USER and pw == WEB_PASSWORD:
            token = new_session(user)
            log("sysauth: login succeeded for %s from %s"
                % (user, self.client_address[0]))
            self.send_response(302)
            self.send_header("Server", SERVER_BANNER)
            self.send_header("Location", "/admin/status/overview")
            self.send_header("Set-Cookie", "sysauth=%s; Path=/" % token)
            self.send_header("Content-Length", "0")
            self.end_headers()
        else:
            log("sysauth: login FAILED for %r from %s"
                % (user, self.client_address[0]))
            self._send(200, login_html(error=True))

    # ------------------------------------------------------------ 备份下载
    def _backup_download(self, params):
        form = parse_qs(params)
        name = (form.get("file") or [""])[0]
        if not re.fullmatch(r"[A-Za-z0-9._-]+", name or ""):
            return self._send(400, "bad file name\n", "text/plain")
        path = Path(BACKUP_DIR) / name
        if not path.is_file():
            return self._send(404, "backup not found\n", "text/plain")
        log("backup: %s downloaded by %s" % (name, self.client_address[0]))
        self._send(200, path.read_bytes(), "application/gzip",
                   {"Content-Disposition":
                    'attachment; filename="%s"' % name})

    # ------------------------------------------------------------ SSH 密钥
    def _ssh_keys_page(self):
        keys = []
        kdir = Path(KEYSTORE_DIR)
        if kdir.is_dir():
            for f in sorted(kdir.glob("*")):
                try:
                    txt = f.read_text(errors="replace")
                except OSError:
                    continue
                keys += re.findall(r"-----[^-]+-----", txt)
        rows = "".join("<tr><td>%s</td><td>%s</td></tr>"
                       % (i + 1, k) for i, k in enumerate(keys))
        body = """<!DOCTYPE html><html><head><meta charset="utf-8">
<title>OpenWrt - SSH-Keys</title></head>
<body style="font:13px Helvetica,Arial,sans-serif;background:#f5f5f5;color:#333">
<div style="max-width:760px;margin:40px auto;background:#fff;border:1px solid #ddd;padding:24px">
<h2 style="color:#00537f">SSH-Keys</h2>
<p>Public keys installed for the admin account. Uploaded via
<code>POST /cgi-bin/luci/admin/ssh_keys</code>.</p>
<table border="0" cellpadding="6" style="border-collapse:collapse">
<tr><th align="left">#</th><th align="left">key</th></tr>%s</table>
<p style="margin-top:18px;color:#999;font-size:11px">
key rotation audit 2019-05-24: 2 keys on file, no pending revocations</p>
<p style="color:#999;font-size:11px">Powered by LuCI 0.10.0 (git-19.020.25664)</p>
</div></body></html>""" % (rows or "<tr><td colspan=2><i>(none)</i></td></tr>")
        self._send(200, body.encode())

    def _ssh_keys_upload(self, body):
        form = parse_qs(body.decode("latin-1", "replace"))
        pem = (form.get("keys") or [""])[0]
        if "-----BEGIN" not in pem or "PRIVATE KEY" not in pem:
            return self._send(400, '{"error":"no private key in body"}',
                              "application/json")
        kdir = Path(KEYSTORE_DIR)
        kdir.mkdir(parents=True, exist_ok=True)
        with open(kdir / "received_keys.pem", "a") as fh:
            fh.write(pem if pem.endswith("\n") else pem + "\n")
        log("ssh_keys: stored 1 key from %s" % self.client_address[0])
        self._send(200, '{"result":"stored","keys":1}', "application/json")

    # ------------------------------------------------------------ 状态页
    def _overview_page(self):
        body = """<!DOCTYPE html><html><head><meta charset="utf-8">
<title>OpenWrt - Status</title></head>
<body style="font:13px Helvetica,Arial,sans-serif;background:#f5f5f5;color:#333">
<div style="max-width:760px;margin:40px auto;background:#fff;border:1px solid #ddd;padding:24px">
<h2 style="color:#00537f">Status</h2>
<p>Realtime bandwidth (eth0):</p>
<pre id="bw">loading...</pre>
<script>
function poll(){
  fetch('/admin/status/realtime/bandwidth_status?iface=eth0')
    .then(function(r){return r.json()})
    .then(function(j){document.getElementById('bw').textContent=JSON.stringify(j)})
    .catch(function(e){document.getElementById('bw').textContent='error'});
}
poll(); setInterval(poll, 3000);
</script>
<p style="margin-top:18px;color:#999;font-size:11px">Powered by LuCI 0.10.0 (git-19.020.25664)</p>
</div></body></html>"""
        self._send(200, body.encode())

    # ------------------------------------------------------------ 路由
    def _route(self):
        parsed = urlparse(self.path)
        path = unquote(parsed.path)
        params = unquote(parsed.query)
        user = session_user(self)

        # ---- 公开：登录页 / 静态 ----
        if path in ("/", "/index.html"):
            return self._static("index.html")
        if path.startswith("/luci/") or path == "/favicon.ico" \
                or path in ("/files", "/export") \
                or path.startswith("/files/") or path.startswith("/export/"):
            # uhttpd 以 /www 为 docroot，/www/export 下的索引对上报主机可见
            if not self._static(path.lstrip("/")):
                self._send(404, "404 Not Found\n", "text/plain")
            return

        # ---- 公开：cgi-io 下载端点（OpenWrt 标配；见 docs/CVE_NOTES.md）----
        if path == "/cgi-bin/cgi-download":
            return self._cgi_download(params)

        # ---- 登录 ----
        if self.command == "POST" and path.rstrip("/") == "/cgi-bin/luci":
            return self._do_login(self._read_body())
        if self.command == "POST" and path == "/cgi-bin/luci/admin/login":
            return self._do_login(self._read_body())

        # ---- 以下全部需要会话 ----
        if path == "/admin/status/overview":
            return self._overview_page() if user else self._forbidden()
        if path == "/admin/ssh_keys":
            return self._ssh_keys_page() if user else self._forbidden()
        if path == "/cgi-bin/luci/backup/download":
            return self._backup_download(params) if user else self._forbidden()
        if self.command == "POST" and path == "/cgi-bin/luci/admin/ssh_keys":
            return self._ssh_keys_upload(self._read_body()) if user else self._forbidden()

        # ---- CVE-2019-12272（需会话）----
        if path == "/admin/status/realtime/bandwidth_status":
            return self._run_cgi("bandwidth_status.sh", params) if user \
                else self._forbidden()
        if path == "/admin/status/realtime/wireless_status":
            return self._run_cgi("bandwidth_status.sh", params) if user \
                else self._forbidden()
        if path == "/cgi-bin/luci/bandwidth_status":
            return self._run_cgi("bandwidth_status.sh", params) if user \
                else self._forbidden()

        if path.startswith("/admin"):
            return self._forbidden()
        return self._send(404, "404 Not Found\n", "text/plain")

    def _read_body(self):
        try:
            n = int(self.headers.get("Content-Length", "0"))
        except ValueError:
            n = 0
        return self.rfile.read(n) if n > 0 else b""

    def do_GET(self):
        try:
            self._route()
        except (BrokenPipeError, ConnectionResetError):
            pass
        except Exception as e:
            log("ERROR %s %s: %r" % (self.command, self.path, e))
            try:
                self._send(500, "Internal Server Error\n", "text/plain")
            except Exception:
                pass

    def do_POST(self):
        self.do_GET()

    do_HEAD = do_GET

    def handle_one_request(self):
        try:
            super().handle_one_request()
        except (BrokenPipeError, ConnectionResetError):
            self.close_connection = True


def _ssl_opt(name):
    """兼容不同 Python 版本：ssl.Options.OP_* (3.7+) / 旧版 ssl.OP_*"""
    opt = getattr(ssl.Options, name, None)
    if opt is None:
        opt = getattr(ssl, name, 0)
    return int(opt)


# OpenSSL 3.x 中这两个选项的位值（Python 3.13 起 ssl 模块不再导出这两个常量，
# 故用探针实测；不同构建若不一致，探针会自动发现正确位值）
_FALLBACK_BITS = {"OP_NO_EXTENDED_MASTER_SECRET": 0x00000001,
                  "OP_NO_ENCRYPT_THEN_MAC": 0x00080000}
_EXT_MAGIC = {"OP_NO_EXTENDED_MASTER_SECRET": b"\x00\x17\x00\x00",   # EMS 扩展
              "OP_NO_ENCRYPT_THEN_MAC": b"\x00\x16\x00\x00"}         # EtM 扩展


def _probe_option_bits():
    """实测当前 OpenSSL 构建中"关闭 EMS / EtM"的选项位：
    带候选位构造 ClientHello，观察对应扩展是否从 ClientHello 中消失。"""
    found = {}

    def hello(bit):
        import socket
        srv = socket.socket()
        srv.bind(("127.0.0.1", 0))
        srv.listen(1)
        box = {}

        def acc():
            try:
                conn, _ = srv.accept()
                box["d"] = conn.recv(8192)
                conn.close()
            except Exception:
                pass
        t = threading.Thread(target=acc, daemon=True)
        t.start()
        ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
        ctx.check_hostname = False
        ctx.verify_mode = ssl.CERT_NONE
        ctx.options |= bit
        try:
            ctx.wrap_socket(socket.create_connection(srv.getsockname())).close()
        except Exception:
            pass
        t.join(2)
        srv.close()
        return box.get("d", b"")

    try:
        base = hello(0)
        if not all(m in base for m in _EXT_MAGIC.values()):
            return found                       # 基线异常，放弃探测
        for bit in range(32):
            data = hello(1 << bit)
            for name, magic in _EXT_MAGIC.items():
                if magic not in data and name not in found:
                    found[name] = 1 << bit
            if len(found) == len(_EXT_MAGIC):
                break
    except Exception:
        pass
    return found


def _resolve_opts():
    """汇总最终要设置的选项位：优先 ssl 模块常量，其次探针结果，最后已知位值"""
    probed = _probe_option_bits()
    opts, how = 0, []
    for name in _FALLBACK_BITS:
        v = _ssl_opt(name)
        if v:
            opts |= v
            how.append("%s=常量" % name)
        elif name in probed:
            opts |= probed[name]
            how.append("%s=探针(0x%08x)" % (name, probed[name]))
        else:
            opts |= _FALLBACK_BITS[name]
            how.append("%s=内置(0x%08x)" % (name, _FALLBACK_BITS[name]))
    return opts, ", ".join(how)


_TLS_OPTS, _TLS_OPTS_HOW = _resolve_opts()


def make_tls_context():
    """复刻 2019 年设备 TLS 配置：TLS1.2 + 静态 RSA 密钥交换（无 PFS），
    关闭 extended_master_secret / encrypt_then_mac —— 有私钥即可事后解密
    （Wireshark / verify_decrypt.py 均可）。"""
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    ctx.load_cert_chain(CERT, KEY)
    ctx.maximum_version = ssl.TLSVersion.TLSv1_2
    ctx.minimum_version = ssl.TLSVersion.TLSv1_2
    # 只保留 TLS_RSA_WITH_AES_128_CBC_SHA (0x002f)，排除所有 ECDHE/DHE
    ctx.set_ciphers("AES128-SHA:!kECDHE:!kDHE:!kECDH:!aECDSA:!aDH:!aNULL:!eNULL:@SECLEVEL=0")
    ctx.options |= _TLS_OPTS
    return ctx


def serve(port, use_tls=False):
    httpd = ThreadingHTTPServer(("0.0.0.0", port), Handler)
    if use_tls:
        httpd.socket = make_tls_context().wrap_socket(httpd.socket, server_side=True)
        log("TLS 配置: TLSv1.2 / TLS_RSA_WITH_AES_128_CBC_SHA (无 PFS)，%s"
            % _TLS_OPTS_HOW)
        proto = "https"
    else:
        proto = "http"
    log("uhttpd 风格服务监听 %s://0.0.0.0:%d (docroot=%s)" % (proto, port, DOCROOT))
    httpd.serve_forever()


def main():
    if not Path(CERT).is_file() or not Path(KEY).is_file():
        print("缺少 TLS 证书: %s / %s" % (CERT, KEY), file=sys.stderr)
        sys.exit(1)
    t = threading.Thread(target=serve, args=(HTTPS_PORT, True), daemon=True)
    t.start()
    serve(HTTP_PORT, False)


if __name__ == "__main__":
    main()"
```
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

# ---- 导出目录（rpcd ACL 只授权这里的读权限，cgi-io 从这里对外提供下载）----
# 注意：这里**不列**任何归档句柄，也不教命名规则；导出登记为空。
#       真实句柄由采集器自己写进系统日志 /var/log/messages（flush 行）。
{
    printf 'netflow-analyzer export index\n'
    printf '=============================\n'
    printf 'generated : 2019-05-24 09:32:10 UTC\n'
    printf 'collector : netflow-cron\n'
    printf 'outdir    : /var/captures\n'
    printf 'retention : 7 days\n\n'
    printf 'Scheduled rollups are registered here by "netflow-cron --register"\n'
    printf 'so the reporting host can pull them through the download CGI.\n'
    printf 'Manual runs write straight into outdir and stay unregistered.\n\n'
    printf '(none)\n'
} > /www/export/index.txt

# ---- 采集器运行日志（syslog 格式；放这里让"已知目录、不知文件名"有解）----
cat > /var/log/messages <<'LOGEOF'
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
LOGEOF

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
我们知道有/var/backups备份文件，可以用一样的方式复制到/www/files/下面下载出来