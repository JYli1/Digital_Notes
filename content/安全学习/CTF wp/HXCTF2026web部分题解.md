# EzShop
## 出题想法
这题考的是floor报错，主要难点就是注入点的发现和floor报错语句的构造。
其实这道题如果人工打没有ai的情况下对于新生还是有点难度的，因为floor报错平时几乎也没怎么遇到过。
（刚开始题目有点问题耽误大家时间了，第一次出题请谅解）


## 找到sql注入点
首先拿到题并且看到提示说有sql注入大概就知道搜索框这里是有注入点的。
但是我们输入`1'`却没有啥反应，这时候应该要反应过来不是常规的注入点了。
那我们就抓包看看：
![](file-20260516133458321.png)
其实细心观察就会发现有一个自定义的响应头了，回显了我们的ip，那我们就应该想到：服务器是怎么获取我们的ip的呢？
我们试试添加
```http
X-Forwarded-For: 127.0.0.1
```
![](file-20260516133718926.png)
看到ip变了，确定是通过xff头获取ip，那这里突然出现一个获取ip并回显的点，我们就应该要注意了，这也是一个sql注入的可能点的。在真实业务场景中我们也可以多试试类似的注入点：
这里模拟的场景是：电商网站通过反向代理（Nginx/CDN/负载均衡）部署，后端信任 X-Forwarded-For 头来获取真实客户端IP，然后将其写入访问日志表做流量分析或安全审计。
写入日志的话那么应该是一个INSERT的sql查询。
```sql
# 后端逻辑是
 sql = f"INSERT INTO access_log (ip, path, time) VALUES ('{ip}', '{path}', NOW())"
```
我们插入`'`试试能不能造成报错。
![](file-20260516134220699.png)
注入点确认。
## 开始注入
因为这里的sql查询时insert语句，插入，所以肯定不是union注入之类的，加上有报错回显。我们考虑报错注入。
我们试试常规的
```sql
1' AND extractvalue(1,concat('~',(select database()))) and '1'='1
```
![](file-20260516140347574.png)
并没有产生报错，奇怪了。难道是被waf了？
再试试，updatexml。
![](file-20260516140449109.png)
估计是被ban了，只要出现关键字就直接return掉。
这时候我们就要试试floor了。
![](file-20260516140546633.png)
成功报错，那么方向很明确了，就是打floor报错。
## floor报错
只要知道了floor就很简单了，问ai也能秒。

payload：
```http
X-Forwarded-For: 1' AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT((SELECT flag FROM secret_flag LIMIT 0,1),FLOOR(RAND(0)*2))x FROM information_schema.tables GROUP BY x)a) AND '1'='1
```

![](file-20260516142236321.png)
floor报错其实本质应该算是groupby报错，利用的是groupby的性质
>**mysql官方注明，在执行group by语句的时候，group by语句后面的字段会被运算两次。**
>**第一次是group by后面的字段和虚拟表进行对比，第二次是插入时会进行运算。**
**由于rand()函数的随机性，导致第二次运算可能和第一运算结果不一致，运算的结果存在，这时插入就会出错。**

`FLOOR(RAND(0)*2)`这段语句只是为了产生一段可以造成报错的01序列。（详细的floor报错原理可以搜索一下橙子科技的sql注入视频，讲的很详细）
大概就是groupby分组的时候要有一个计算并且把结果插入临时表的过程，中间产生了key的冲突导致报错。

# HardShop

