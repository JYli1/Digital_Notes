---
title: Nodejs安全总览
---

> 关联：[[Nodejs安全总览]]、[[../index|安全学习]]

# Nodejs安全总览

Node.js 安全重点关注对象原型、沙箱隔离、模板渲染和依赖包行为。

## 已有笔记

- [[JS原型链污染]]：`__proto__`、`constructor.prototype` 等污染入口。
- [[VM沙箱逃逸]]：从受限 JavaScript 执行环境逃逸到 `process` 或 `child_process`。

## 关联方向

- [[../ssti注入/SSTI总览|SSTI]]
- [[../CVE积累/CVE总览|CVE积累]]

## 复盘模板

1. 污染点或执行点在哪里。
2. 可控键名、可控值、合并函数或模板表达式是什么。
3. 是否能影响鉴权、路径、命令参数或模板配置。
4. 最终 sink 是文件读取、命令执行、鉴权绕过还是拒绝服务。
