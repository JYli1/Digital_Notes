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
