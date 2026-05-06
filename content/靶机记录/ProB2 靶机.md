---
作者: L1Qin9
靶机id: "634"
难度: Easy
---
# 主机发现
```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sn 10.216.75.0/24
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-06 14:39 CST
Nmap scan report for 10.216.75.115
Host is up (0.0018s latency).
MAC Address: 08:00:27:89:39:82 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Nmap scan report for 10.216.75.183
Host is up (0.0061s latency).
MAC Address: 4E:20:31:25:4A:3C (Unknown)
Nmap scan report for 10.216.75.212
Host is up (0.00093s latency).
MAC Address: 30:E3:A4:48:AC:29 (Unknown)
Nmap scan report for 10.216.75.80
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 17.30 seconds
```
靶机ip：`10.216.75.115`

# 信息收集
`rustscan`扫一下
```bash
┌──(kali㉿kali)-[~]
└─$ rustscan -a 10.216.75.115 --ulimit 5000
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
RustScan: Because guessing isn't hacking.

[~] The config file is expected to be at "/home/kali/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.216.75.115:22
Open 10.216.75.115:80
Open 10.216.75.115:8080
[~] Starting Script(s)
[~] Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-06 14:42 CST
Initiating ARP Ping Scan at 14:42
Scanning 10.216.75.115 [1 port]
Completed ARP Ping Scan at 14:42, 0.12s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 14:42
Completed Parallel DNS resolution of 1 host. at 14:42, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 1, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 14:42
Scanning 10.216.75.115 [3 ports]
Discovered open port 22/tcp on 10.216.75.115
Discovered open port 80/tcp on 10.216.75.115
Discovered open port 8080/tcp on 10.216.75.115
Completed SYN Stealth Scan at 14:42, 0.05s elapsed (3 total ports)
Nmap scan report for 10.216.75.115
Host is up, received arp-response (0.0021s latency).
Scanned at 2026-05-06 14:42:49 CST for 0s

PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 64
80/tcp   open  http       syn-ack ttl 64
8080/tcp open  http-proxy syn-ack ttl 63
MAC Address: 08:00:27:89:39:82 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.35 seconds
           Raw packets sent: 4 (160B) | Rcvd: 4 (160B)
```
开放`22`、`80`、`8080`端口
