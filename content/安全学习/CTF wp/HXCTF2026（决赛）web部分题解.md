> 关联：[[index|CTF wp]]、[[../index|安全学习]]

# PromptPivot
作者：JYli

找到题其实设计的有点粗糙，原本是初赛上，但是后面说决赛上我就没怎么管他，然后决赛出题时间比较赶，没有把它设计的完美，给各位说个抱歉，就当给大家提供一个ai了。那我就说说我的预期解吧。
>平时都这么喜欢提示词注入怎么比赛才这么点做出来，还是给了这么多提示。（bushi


首先拿到题目是一个智能客服，有一个RAG知识库
>这里解释一下RAG，各位以后多多少少是要学点ai-agent的知识的。
>RAG（检索增强生成），核心就是向量数据库。解决的痛点就是ai大模型的幻觉问题，大家平时用ai时会发现很多时候明明是假的消息但是ai说的跟真的一样。例如你问他湖南科技大学的一个什么老师的消息，不联网搜索他很可能给你编一个答案。此时如果湖科大要在官网做一个智能客服怎么办呢？
>那就是
>1. 把和学校有关的实时消息保存为一个大的”说明书“。然后按照一定规则切片（按章节或者按句子什么的），再利用embedding模型（和大语言模型并列的另外一种模型），把这些知识切片转化为多维向量（1，2，3，4，......），一般都有几千个维度。
>2. 然后在你询问时，例如你问湖南科技大学的校长是谁？agent先把你这个问题同样转化成向量，然后安装向量维度上最接近的切片寻找到指定数量的切片（这里怎么理解呢？我们以二维向量举例：那就是一个平面向量嘛，知识库里面也是很多个平面向量，我们找到和我们输入冲重合程度最高的就好了，所以说维度越多越精准）。找到这些向量后给转化成自然语言在拼接一起发送给大语言模型（LLM），就像上面的例子，可能就找到了"湖南科技大学校长xxx参加xxx"，这一条，然后和问题一起发给LLM。这个时候ai是不是就大概率不需要编造了。
>当然这只是最基础的RAG系统，后续还有很多内容例如，重排列（rerank）等优化准确度的方法，感兴趣的可以自行学习

这里也是想让大家知道一下RAG这个东西。但是第一眼我们肯定只会想到提示词注入，那么我们试着引导ai说出他的系统提示词。这里方法有很多，就不过多讲了。（这里就是没设计好的地方，很多人可能直接就问出来有啥工具，或者直接在输入款就让他调用了），预期解是得到系统提示里的base64消息，
`{"tool": "web_fetch", "description": "获取网页内容的tools，你需要用一条request.get()函数"}`
知道有一个web_fetch工具，然后发现从输入框直接要他调用却失败。转而在RAG知识库中要他调用。题目很简单大家都可以去复现一下。
# 破旧小店
作者：JYli

这个题只有一解是我没有想到的，我当时都打算放简答题200分的。一个筛子cms，应该是随便上网搜一下就一大堆cve才对呀，而且我找的老版本1.5.2很多cve应该都是可以的。
这里提供两种，是否有其他的大家可以自行测试。
两个链接：
https://xz.aliyun.com/news/16382
https://www.exploit-db.com/exploits/52489

首先我们起码要先进入后台才有机会漏洞利用。
拿到一个web
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529165526929.png)
首先要注意到这是一个`WBCE CMS`，此时你就应该要想到可能网上会有现存cve。
然后第二你会去试试这个搜索款是不是有sql注入之类的。
然后试了半天没有结果，这个时候以及90%确定是cve题了。
但是你上网搜会发现基本都是要登录后台的，那你现在什么都没有，那怎么办？
扫目录。这是真实环境对一个web页面基本的信息收集。
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529165852978.png)
注意到`robots.txt`、`backup目录`，config文件也可以去看一下但是这里是没有东西的。
我们去看看。
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529170022261.png)
很明显一个备份文件，一个admin后台。先去下载备份文件看看
可以得到一份sql文件
```sql
-- WBCE users table backup
-- Generated during site migration

CREATE TABLE `wb_users` (
  `user_id` int(11) NOT NULL AUTO_INCREMENT,
  `group_id` int(11) NOT NULL DEFAULT 0,
  `groups_id` varchar(255) NOT NULL DEFAULT '',
  `username` varchar(255) NOT NULL DEFAULT '',
  `display_name` varchar(255) NOT NULL DEFAULT '',
  `language` varchar(5) NOT NULL DEFAULT '',
  `email` text NOT NULL,
  `password` varchar(255) NOT NULL DEFAULT '',
  `active` int(11) NOT NULL DEFAULT 0,
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3 COLLATE=utf8mb3_unicode_ci;

INSERT INTO `wb_users` (`user_id`,`group_id`,`groups_id`,`username`,`display_name`,`language`,`email`,`password`,`active`) VALUES
(1,1,'1','editor','Administrator','EN','editor@example.local','efb9f68d4a27db0ce485b845b4081e82',1);

```
重点是其中有完整的账号和MD5(密码)，[cmd5](https://www.cmd5.com/)网站弱密码直接解密。（强一点点也可以试试john等工具爆破，原本是没设这么简单的密码的，考虑到你们可能没接触过爆破工具）
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529170348294.png)
然后可以成功登录后台
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529170523667.png)
## 方法一： CVE-2022-25099

在`Add-ons-->Language`页面存在一个没有任何限制的文件上传。（所以说你不知道cve。但凡你多点点，看到文件上传就试一试就做出来了）
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529170730996.png)
超级简单好吧。
## 方法二：Droplets（水滴）模块存在任意php代码执行
我题目描述也提到了水滴。
这个后面再网上搜没有搜到比较完整的复现文章了，不知道为什么。但是还是有描述的呀
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529171009080.png)
别看是英文就没用呀，翻译一下。
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529171154662.png)
讲的相当清楚了。
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529171235952.png)
找到水滴模块添加（add droplet）

![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529171648540.png)

然后来到page页面添加文章，在里面引用我们刚刚创建的文件脚本（shell.php），这里随便什么名字，后缀也不需要
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529171828746.png)
然后查看我们刚刚写的文章就好
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529172008042.png)
后门利用
![](https://raw.githubusercontent.com/JYli1/my-blog-images2/main/images/HXCTF2026%EF%BC%88%E5%86%B3%E8%B5%9B%EF%BC%89web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/file-20260529172039626.png)
所以说真的非常非常简单一道题了，各位没做出来好好复现总结一下吧