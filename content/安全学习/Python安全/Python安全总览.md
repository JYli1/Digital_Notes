---
title: Python安全总览
---

# Python安全总览

Python 安全方向主要围绕反序列化、Web 框架签名、沙箱逃逸和运行时对象模型。

## 已有笔记

- [[Pickle反序列化]]：`pickle` 反序列化触发任意对象构造和命令执行。
- [[flask session伪造]]：Flask Session 签名、密钥爆破和伪造。
- [[python原型链污染]]：Python 对象属性污染和链式影响。
- [[python内存马]]：运行时注入和持久化思路。
- [[Pyjail 沙箱逃逸/沙箱逃逸基础|Pyjail 沙箱逃逸基础]]
- [[Pyjail 沙箱逃逸/Bypass|Pyjail Bypass]]
- [[Pyjail 沙箱逃逸/逃逸目标|Pyjail 逃逸目标]]

## 关联方向

- [[../ssti注入/SSTI总览|SSTI]]
- [[../CTF wp/ISCTF 2025|ISCTF 2025]]
- [[../一些小技巧/python斜体字命令执行|python斜体字命令执行]]

## 复盘模板

1. 可控输入进入哪个对象、函数或表达式。
2. 能否访问 `__class__`、`__mro__`、`__subclasses__`、`globals`。
3. 是否有黑名单、长度限制、无字母数字限制。
4. 最终目标是读文件、执行命令还是伪造身份。
