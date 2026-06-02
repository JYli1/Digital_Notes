---
title: Java安全总览
---

> 关联：[[Java安全总览]]、[[../index|安全学习]]

# Java安全总览

Java 安全笔记可以分成 Java 基础、Web 目录结构、反序列化链和常见框架漏洞几类。

## 基础与 Web

- [[Java开发/Java基础|Java基础]]：类、反射、代理、线程等基础语法。
- [[Java开发/Java Web|Java Web]]：Maven 和 Web 项目基础。
- [[WEB-INF文件夹利用]]：WEB-INF 目录泄露后读取 class、配置和依赖。

## 反序列化

- [[Java反序列化/CommonsCollections 反序列化链/CC1|CommonsCollections CC1]]
- [[Java反序列化/CommonsCollections 反序列化链/CC1 国外|CC1 国外分析]]
- [[Java反序列化/CommonsCollections 反序列化链/CC6|CommonsCollections CC6]]

## 关联方向

- [[../知识分享/2026-1-3-java|Java 知识分享]]
- [[../CVE积累/CVE总览|CVE积累]]

## 复盘模板

1. 入口点：原生反序列化、XML、JSON、RMI、JNDI 或框架参数绑定。
2. 依赖版本：是否存在 CommonsCollections、TemplatesImpl、Groovy 等链条条件。
3. 触发方法：`readObject`、动态代理、反射调用或 getter/setter。
4. 最终 sink：命令执行、文件写入、JNDI 请求或类加载。
