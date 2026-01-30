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
