> 关联：[[SSTI总览]]、[[../Python安全/Python安全总览|Python安全]]

# Tornado模板注入

Tornado 模板注入属于 [[SSTI总览|SSTI]] 的一个分支。排查时先确认用户输入是否进入模板语法，再判断能否访问 Python 对象、模块或危险函数。

## 参考资料

- Tornado 官方文档：https://tornado-zh.readthedocs.io/zh/latest/
- SSTI 快速入门：https://xz.aliyun.com/news/11706
- https://forum.butian.net/share/2195

## 复盘要点

1. 输入是否被当作模板表达式渲染。
2. 能否使用 Tornado 模板语法访问变量或函数。
3. 是否能进一步接到 [[../Python安全/Pyjail 沙箱逃逸/沙箱逃逸基础|Pyjail 沙箱逃逸]] 或文件读取。
4. 过滤点是模板字符、关键字还是输出回显。
