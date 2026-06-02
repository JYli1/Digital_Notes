---
title: PHP安全总览
---

# PHP安全总览

PHP 安全笔记可以按“输入进入危险函数”“文件/协议封装器”“对象生命周期”“原生类利用”四条线整理。

## 命令与代码执行

- [[PHP命令执行/命令执行基础|命令执行基础]]：系统命令执行函数、空格绕过、黑名单绕过。
- [[PHP命令执行/LD_PRELOAD劫持|LD_PRELOAD劫持]]：通过动态链接预加载劫持函数执行命令。
- [[filter-chain RCE]]：利用 `php://filter` 链构造代码执行。
- [[pearcmd文件包含getshell]]：借助 pearcmd 参数写文件或触发 getshell。
- [[深入了解preg_replace代码执行]]：旧版本 `/e` 修饰符导致的代码执行。

## 反序列化

- [[PHP反序列化/反序列化基础|反序列化基础]]：魔术方法、引用、session 反序列化、字符串逃逸。
- [[PHP反序列化/绕过wakeup|绕过wakeup]]：绕过 `__wakeup()` 检查。
- [[PHP反序列化/绕过O：|绕过O:]]：绕过对象标识过滤。
- [[PHP反序列化/phar的文件包含与反序列化|phar的文件包含与反序列化]]：phar metadata 触发反序列化。
- [[PHP反序列化/再探phar文件包含,反序列化|再探phar文件包含,反序列化]]：结合题目复盘 phar 文件包含细节。
- [[PHP反序列化/phpGC回收机制的利用|phpGC回收机制的利用]]：利用 GC 或 fast destruct 改变析构触发时机。

## 原生类与语言特性

- [[php原生类利用]]：Error、Exception、SoapClient、迭代器、SimpleXMLElement 等内置类利用。
- [[php特性]]：弱类型、比较、字符串和数组相关特性。

## 关联方向

- [[../SSRF/SSRF总览|SSRF]]
- [[../SQL注入/SQL注入总览|SQL注入]]
- [[../CTF wp/ISCTF 2025|ISCTF 2025]]
