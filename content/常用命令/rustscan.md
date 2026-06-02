> 关联：[[index|常用命令]]、[[nmap]]、[[../靶机记录/index|靶机记录]]

```markdown
# RustScan 命令速查表

## 基础语法

```bash
rustscan -a <目标> -- <nmap参数>
```

## 常用参数

| 参数 | 说明 | 示例 |
| :--- | :--- | :--- |
| `-a` | 目标IP/域名/CIDR | `-a 192.168.1.1` |
| `-p` | 指定端口 | `-p 22,80,443` |
| `-r` | 端口范围 | `-r 1-1000` |
| `--top` | 热门端口 | `--top` |
| `-b` | 并发数(默认4500) | `-b 1000` |
| `-t` | 超时毫秒 | `-t 3000` |
| `--tries` | 重试次数 | `--tries 2` |
| `--greppable` | 仅显示端口 | `--greppable` |
| `--json` | JSON输出 | `--json` |

## 常用命令

### 快速发现端口
```bash
rustscan -a 192.168.1.100 --greppable
```

### 端口范围扫描
```bash
rustscan -a 192.168.1.100 -r 1-1000
```

### 自动调用Nmap
```bash
rustscan -a 192.168.1.100 -- -A -sC -sV
```

### 扫描多目标
```bash
rustscan -a 192.168.1.100,10.0.0.0/24,example.com
```

## 配置文件 `~/.rustscan.toml`

```toml
batch_size = 3000
timeout = 1500
tries = 2
scan_order = "Random"
```

## 速度调节

| 场景 | 命令 |
| :--- | :--- |
| 快速扫描 | `-b 5000 -t 1000` |
| 稳定扫描 | `-b 1000 -t 3000 --tries 2` |
| 隐蔽扫描 | `-b 500 -t 5000 --scan-order Random` |
```
