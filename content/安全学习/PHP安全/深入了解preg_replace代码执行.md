# 深入了解 preg_replace 代码执行

旧版本 PHP 中 `preg_replace` 的 `/e` 修饰符会把替换结果当作 PHP 代码执行，因此可能从正则替换变成代码执行。这个点适合归到 [[PHP安全总览|PHP安全]] 和 [[PHP命令执行/命令执行基础|命令执行基础]]。

## 参考资料

- https://xz.aliyun.com/news/2239
- https://cloud.tencent.com/developer/article/1610410

## 复盘要点

1. PHP 版本是否支持 `/e` 修饰符。
2. 正则、替换内容、目标字符串哪些部分可控。
3. 替换结果是否能构造成有效 PHP 代码。
4. 是否存在过滤或转义。

## 关联笔记

- [[PHP安全总览]]
- [[PHP命令执行/命令执行基础]]
- [[php特性]]
