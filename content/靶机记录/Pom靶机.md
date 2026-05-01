# 主机发现

```bash
┌──(root㉿kali)-[/home/kali]
└─# nmap -sn 10.241.108.0/24 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-01 18:06 CST
Nmap scan report for 10.241.108.43
Host is up (0.0070s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.241.108.62
Host is up (0.0013s latency).
MAC Address: 08:00:27:E8:5C:CB (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.241.108.72
Host is up (0.15s latency).
MAC Address: 98:2C:BC:40:09:7F (Intel Corporate)
Nmap scan report for 10.241.108.212
Host is up (0.00052s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.241.108.201
Host is up.
Nmap done: 256 IP addresses (5 hosts up) scanned in 11.63 seconds

```
靶机ip：`10.241.108.62`

# 端口扫描-信息收集
看佬们的wp发现一个新工具`Rustscan`，这次试了一下

```bash
┌──(root㉿kali)-[/home/kali/tmp]
└─# rustscan -a 10.241.108.62 -- -A      
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
With RustScan, I scan ports so fast, even my firewall gets whiplash 💨

[~] The config file is expected to be at "/root/.rustscan.toml"
[!] File limit is lower than default batch size. Consider upping with --ulimit. May cause harm to sensitive servers
[!] Your file limit is very small, which negatively impacts RustScan's speed. Use the Docker image, or up the Ulimit with '--ulimit 5000'. 
Open 10.241.108.62:22
Open 10.241.108.62:80
Open 10.241.108.62:55555
[~] Starting Script(s)
[>] Running script "nmap -vvv -p {{port}} -{{ipversion}} {{ip}} -A" on ip 10.241.108.62
Depending on the complexity of the script, results may take some time to appear.
[~] Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-01 21:30 CST
NSE: Loaded 157 scripts for scanning.
NSE: Script Pre-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 21:30
Completed NSE at 21:30, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 21:30
Completed NSE at 21:30, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 21:30
Completed NSE at 21:30, 0.00s elapsed
Initiating ARP Ping Scan at 21:30
Scanning 10.241.108.62 [1 port]
Completed ARP Ping Scan at 21:30, 0.09s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 21:30
Completed Parallel DNS resolution of 1 host. at 21:30, 0.08s elapsed
DNS resolution of 1 IPs took 0.08s. Mode: Async [#: 3, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 21:30
Scanning 10.241.108.62 [3 ports]
Discovered open port 80/tcp on 10.241.108.62
Discovered open port 22/tcp on 10.241.108.62
Discovered open port 55555/tcp on 10.241.108.62
Completed SYN Stealth Scan at 21:30, 0.03s elapsed (3 total ports)
Initiating Service scan at 21:30
Scanning 3 services on 10.241.108.62
Stats: 0:01:30 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 66.67% done; ETC: 21:32 (0:00:45 remaining)
Stats: 0:01:35 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 66.67% done; ETC: 21:32 (0:00:48 remaining)
Stats: 0:01:40 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 66.67% done; ETC: 21:32 (0:00:50 remaining)
Completed Service scan at 21:33, 162.73s elapsed (3 services on 1 host)
Initiating OS detection (try #1) against 10.241.108.62
NSE: Script scanning 10.241.108.62.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 21:33
Completed NSE at 21:33, 0.27s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 21:33
Completed NSE at 21:33, 1.16s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 21:33
Completed NSE at 21:33, 0.00s elapsed
Nmap scan report for 10.241.108.62
Host is up, received arp-response (0.0010s latency).
Scanned at 2026-05-01 21:30:24 CST for 165s

PORT      STATE SERVICE     REASON         VERSION
22/tcp    open  ssh         syn-ack ttl 64 OpenSSH 10.0 (protocol 2.0)
80/tcp    open  http        syn-ack ttl 64 Apache httpd 2.4.66 ((Unix))
|_http-title: Whitelabel Error Page
| http-methods: 
|   Supported Methods: HEAD GET POST OPTIONS TRACE
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.66 (Unix)
55555/tcp open  ssl/unknown syn-ack ttl 64
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Maven/stateOrProvinceName=State/countryName=US/localityName=City/organizationalUnitName=Proxy
| Issuer: commonName=localhost/organizationName=Maven/stateOrProvinceName=State/countryName=US/localityName=City/organizationalUnitName=Proxy
| Public Key type: rsa
| Public Key bits: 4096
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2026-02-26T07:20:48
| Not valid after:  2036-02-24T07:20:48
| MD5:   2c76:e6b1:965f:a05f:ed53:8a14:1d60:e6c3
| SHA-1: 4ec1:5a4c:7082:c3bb:2d29:7ac2:66be:2aa9:0cec:2628
| -----BEGIN CERTIFICATE-----
| MIIFoTCCA4mgAwIBAgIUVLyuh/l5EdN2c47Y0L//ePAO7GYwDQYJKoZIhvcNAQEL
| BQAwYDELMAkGA1UEBhMCVVMxDjAMBgNVBAgMBVN0YXRlMQ0wCwYDVQQHDARDaXR5
| MQ4wDAYDVQQKDAVNYXZlbjEOMAwGA1UECwwFUHJveHkxEjAQBgNVBAMMCWxvY2Fs
| aG9zdDAeFw0yNjAyMjYwNzIwNDhaFw0zNjAyMjQwNzIwNDhaMGAxCzAJBgNVBAYT
| AlVTMQ4wDAYDVQQIDAVTdGF0ZTENMAsGA1UEBwwEQ2l0eTEOMAwGA1UECgwFTWF2
| ZW4xDjAMBgNVBAsMBVByb3h5MRIwEAYDVQQDDAlsb2NhbGhvc3QwggIiMA0GCSqG
| SIb3DQEBAQUAA4ICDwAwggIKAoICAQDylEQQUsAPSo6YTElEnKXxIQ6utctm6EVM
| 7AZKMb8FpkRymjIwHXnchsIClpzxVnone92ZP8APFhkXGTU9JQhc4Zk5KPYgRQMw
| pLvZj8vqM5b3Vr8/3K7pAeUW9b2cB6deg28AQBviUEMJM1ZhMfIlnZh7reBx5GI+
| 6HvRVlD5Xg73zFdvj+JLX9QRLEaUWKIEs35wwEyNp99aXQVoTGmPfvUOP8IejpBa
| DA/YD/1JMqiUOyxlVg48pNaNcItvPDJ63PuueXHXReTF47E+ly0uC1GPZJiVfmQR
| Y6M+Rawsl0gc12xEpbybU+ZKwXdXLbOESHf1HEFYaSLt/AsvhuS+U6A1S8OWzcJD
| PMaa+Na4lks40ThhIbL5gpNNvLAVnGqSrN/XdQxXEqMnrGu6nju0260eWvcyl7qZ
| Z/YEbguuNMs643fkxaDG9P323ZE4BmKevLA3b062Vmjd0AmvPTKFw2GY6AY1XdIP
| 9EHSePaTA1KgXaowUBybK+gnOFUWWjI6ngWLUQjO+nCinMjlgG/URYHPIybdYTBv
| PS2aQ18tYPKlCdoQNvWEhIYYKroKpfr3WHANuAySJfp5OmcGz9p5EOTU0oOl0LSq
| C42zck1eWT/qLDppvJrrqxCS/x1k2SplkQx50ohnkwr0MtaNDFqm1HJDfnSGliHM
| GYSliYwgDwIDAQABo1MwUTAdBgNVHQ4EFgQU9/xnvtfBkD4TGxnNgjoDSQXG1tAw
| HwYDVR0jBBgwFoAU9/xnvtfBkD4TGxnNgjoDSQXG1tAwDwYDVR0TAQH/BAUwAwEB
| /zANBgkqhkiG9w0BAQsFAAOCAgEAuOVYZ8E3uOVBdnMrUQuGDFP6ZNGvyAmNMp84
| 2QxL8g8r8K5SNo/JHhBf8CkY2BJG15l01/Ic6g56DqzTsQjfmwQAjfnqVWyHwcp9
| KSj9XdE3V39QO1ZGgICStAW0n3TXAa1jRn+GHcz7SZyTB9jGOqh1aAOrAH6/Rsdf
| XBsBpesbeMAHhD2oJdMl45Bkl3dW8FruNP/CAde8vpPR0AQmsHy/2a5m7dcsCK39
| JgroLyhPJGFdHi698agrn5d1vNwzpfQRmu9Iw6Bk/0W5U8jlFYMZu+fK2rzA06o6
| 96GZumbchDHhsi7Ajoa+RYTEXqbyU9a2KDnm3OZC5R5WQY/d+BVnjRmefPWv8kqa
| 5gJ0FGoYCMiR8PY0dpH9wKm4IwxsQAIOe3S2VuEaI6ceRO7Yq/FS31addb+B3UjV
| pWm7+WDDjsEKXY7jxjqWk7Lq3Rh1bTMDR4fxFVhXLish3vQfDA8xSZrGCRaqurwM
| AzjwR2b2GkKVaUVgMJ/8W1kupoJAjr63JIhzmCz3Ciq9/uwgpLFqHhMn1ooBKToU
| rWXPT5PXKEFHODfP+uj9NP8aTxrevJ1SCX7e0S6UO61J0Zjmk7jY+L6ke5blKGiV
| 9NYsvoozjwwRKzAxR95V8TVCy4gNa5ViLG6FcXQ1hLDdfMfRTfuMrYvTtiqhWVvd
| TgrwK04=
|_-----END CERTIFICATE-----
| fingerprint-strings: 
|   DNSStatusRequestTCP: 
|     ^@^L^@^@^P^@^@^@^@^@^@^@^@^@
|     [?2004hsh-5.2$ 
|     [Jsh-5.2$
|   DNSVersionBindReqTCP: 
|     [?2004hsh-5.2$ ^C
|     [?2004l
|     [?2004h
|     [?2004l
|     [?2004hsh-5.2$
|   GenericLines: 
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$
|   GetRequest: 
|     GET / HTTP/1.0
|     [?2004hsh-5.2$ GET / HTTP/1.0
|     [?2004l
|     GET: command not found
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$
|   HTTPOptions: 
|     OPTIONS / HTTP/1.0
|     [?2004hsh-5.2$ OPTIONS / HTTP/1.0
|     [?2004l
|     OPTIONS: command not found
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$
|   Help: 
|     [?2004hsh-5.2$ HELP
|     [?2004l
|     HELP: command not found
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$
|   Kerberos: 
|     ^C^B^A^B
|     [?2004hsh-5.2$
|   NULL, RPCCheck: 
|     [?2004hsh-5.2$
|   RTSPRequest: 
|     OPTIONS / RTSP/1.0
|     [?2004hsh-5.2$ OPTIONS / RTSP/1.0
|     [?2004l
|     OPTIONS: command not found
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$ 
|     [?2004l
|     [?2004hsh-5.2$
|   SSLSessionReq: 
|     [?2004hsh-5.2$ ^C
|     [?2004l
|     [?2004h
|   TLSSessionReq: 
|     ^C^E^B
|   TerminalServerCookie: 
|     ^C^@^@^@
|_    [?2004hsh-5.2$
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port55555-TCP:V=7.95%T=SSL%I=7%D=5/1%Time=69F4AB01%P=x86_64-pc-linux-gn
SF:u%r(NULL,10,"\x1b\[\?2004hsh-5\.2\$\x20")%r(GenericLines,7C,"\x1b\[\?20
SF:04hsh-5\.2\$\x20\r\n\x1b\[\?2004l\r\x1b\[\?2004hsh-5\.2\$\x20\r\n\x1b\[
SF:\?2004l\r\x1b\[\?2004hsh-5\.2\$\x20\r\n\x1b\[\?2004l\r\x1b\[\?2004hsh-5
SF:\.2\$\x20\r\n\x1b\[\?2004l\r\x1b\[\?2004hsh-5\.2\$\x20")%r(GetRequest,B
SF:C,"GET\x20/\x20HTTP/1\.0\r\n\r\n\r\n\r\n\x1b\[\?2004hsh-5\.2\$\x20GET\x
SF:20/\x20HTTP/1\.0\r\n\x1b\[\?2004l\rsh:\x20GET:\x20command\x20not\x20fou
SF:nd\r\n\x1b\[\?2004hsh-5\.2\$\x20\r\n\x1b\[\?2004l\r\x1b\[\?2004hsh-5\.2
SF:\$\x20\r\n\x1b\[\?2004l\r\x1b\[\?2004hsh-5\.2\$\x20\r\n\x1b\[\?2004l\r\
SF:x1b\[\?2004hsh-5\.2\$\x20")%r(HTTPOptions,C8,"OPTIONS\x20/\x20HTTP/1\.0
SF:\r\n\r\n\r\n\r\n\x1b\[\?2004hsh-5\.2\$\x20OPTIONS\x20/\x20HTTP/1\.0\r\n
SF:\x1b\[\?2004l\rsh:\x20OPTIONS:\x20command\x20not\x20found\r\n\x1b\[\?20
SF:04hsh-5\.2\$\x20\r\n\x1b\[\?2004l\r\x1b\[\?2004hsh-5\.2\$\x20\r\n\x1b\[
SF:\?2004l\r\x1b\[\?2004hsh-5\.2\$\x20\r\n\x1b\[\?2004l\r\x1b\[\?2004hsh-5
SF:\.2\$\x20")%r(RTSPRequest,C8,"OPTIONS\x20/\x20RTSP/1\.0\r\n\r\n\r\n\r\n
SF:\x1b\[\?2004hsh-5\.2\$\x20OPTIONS\x20/\x20RTSP/1\.0\r\n\x1b\[\?2004l\rs
SF:h:\x20OPTIONS:\x20command\x20not\x20found\r\n\x1b\[\?2004hsh-5\.2\$\x20
SF:\r\n\x1b\[\?2004l\r\x1b\[\?2004hsh-5\.2\$\x20\r\n\x1b\[\?2004l\r\x1b\[\
SF:?2004hsh-5\.2\$\x20\r\n\x1b\[\?2004l\r\x1b\[\?2004hsh-5\.2\$\x20")%r(RP
SF:CCheck,10,"\x1b\[\?2004hsh-5\.2\$\x20")%r(DNSVersionBindReqTCP,3E,"\x1b
SF:\[\?2004hsh-5\.2\$\x20\^C\x1b\[\?2004l\r\x1b\[\?2004h\x1b\[\?2004l\r\r\
SF:n\x1b\[\?2004hsh-5\.2\$\x20")%r(DNSStatusRequestTCP,3A,"\^@\^L\^@\^@\^P
SF:\^@\^@\^@\^@\^@\^@\^@\^@\^@\x1b\[\?2004hsh-5\.2\$\x20\x1b\[H\x1b\[Jsh-5
SF:\.2\$\x20")%r(Help,67,"\x1b\[\?2004hsh-5\.2\$\x20HELP\r\n\x1b\[\?2004l\
SF:rsh:\x20HELP:\x20command\x20not\x20found\r\n\x1b\[\?2004hsh-5\.2\$\x20\
SF:r\n\x1b\[\?2004l\r\x1b\[\?2004hsh-5\.2\$\x20")%r(SSLSessionReq,3B,"\x1b
SF:\[\?2004hsh-5\.2\$\x20\^C\x1b\[\?2004l\r\x1b\[\?2004h\x1b\[C\x1b\[C\x1b
SF:\[C\x1b\[C\x1b\[C\x1b\[C\x1b\[C\x1b\[C")%r(TerminalServerCookie,18,"\^C
SF:\^@\^@\^@\x1b\[\?2004hsh-5\.2\$\x20")%r(TLSSessionReq,8,"\^C\^E\^B\r\n"
SF:)%r(Kerberos,1A,"\^C\^B\^A\^B\x1b\[\?2004hsh-5\.2\$\x20\x07\x07");
MAC Address: 08:00:27:E8:5C:CB (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=5/1%OT=22%CT=%CU=36438%PV=Y%DS=1%DC=D%G=N%M=080027%TM=
OS:69F4AB95%P=x86_64-pc-linux-gnu)SEQ(SP=100%GCD=1%ISR=108%TI=Z%CI=Z%II=I%T
OS:S=A)OPS(O1=M5B4ST11NW7%O2=M5B4ST11NW7%O3=M5B4NNT11NW7%O4=M5B4ST11NW7%O5=
OS:M5B4ST11NW7%O6=M5B4ST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=F
OS:E88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M5B4NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A
OS:=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%
OS:Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=
OS:A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=
OS:Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%
OS:T=40%CD=S)

Uptime guess: 31.330 days (since Tue Mar 31 13:37:44 2026)
Network Distance: 1 hop
TCP Sequence Prediction: Difficulty=256 (Good luck!)
IP ID Sequence Generation: All zeros

TRACEROUTE
HOP RTT     ADDRESS
1   1.02 ms 10.241.108.62

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 21:33
Completed NSE at 21:33, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 21:33
Completed NSE at 21:33, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 21:33
Completed NSE at 21:33, 0.00s elapsed
Read data files from: /usr/share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 166.39 seconds
           Raw packets sent: 26 (1.938KB) | Rcvd: 20 (1.538KB)

```
我们直接ssl登录上去
```bash
┌──(root㉿kali)-[/home/kali/tmp]
└─# openssl s_client -connect 10.241.108.62:55555 -quiet
Connecting to 10.241.108.62
Can't use SSL_get_servername
depth=0 C=US, ST=State, L=City, O=Maven, OU=Proxy, CN=localhost
verify error:num=18:self-signed certificate
verify return:1
depth=0 C=US, ST=State, L=City, O=Maven, OU=Proxy, CN=localhost
verify return:1
sh-5.2$ ls /home
ls /home
mav1234   qc2000    terra536
sh-5.2$ ls
ls
journel   user.txt
sh-5.2$ cat journel
cat journel
I’ve encrypted my secret with my key. No one else can read it now. It’s safely hidden away:)


```
这就直接拿到了user-shell，并且有提示说`用密钥加密了秘密`
我们找一找有没有隐藏文件
