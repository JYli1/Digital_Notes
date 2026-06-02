---
title: SSTI总览
---

# SSTI总览

SSTI 是用户输入进入模板引擎后被当作模板表达式求值。不同框架的对象访问方式和沙箱限制不同，整理时要把模板语法、可达对象和最终执行链分开写。

## 已有笔记

- [[bottle模板注入]]：Bottle 场景下的 SSTI 和 cookie 反序列化问题。
- [[Smarty模板注入]]：PHP Smarty 模板注入。
- [[tornado模板注入]]：Tornado 模板语法和常见入口。

## 关联方向

- [[../Python安全/Pyjail 沙箱逃逸/沙箱逃逸基础|Pyjail 沙箱逃逸]]
- [[../Python安全/Pickle反序列化|Pickle反序列化]]
- [[../Nodejs安全/VM沙箱逃逸|VM沙箱逃逸]]

## 复盘模板

1. 模板引擎：Jinja2、Bottle、Tornado、Smarty 或其他。
2. 注入位置：变量输出、模板文件名、配置项还是表达式。
3. 探测 payload：数学表达式、变量读取、对象遍历。
4. 利用路径：读配置、读文件、命令执行或反序列化。
5. 限制条件：过滤字符、沙箱、黑名单、无回显。
