---
title: Gitalk的自动初始化
date: 2018-09-21 20:36:47
urlname: gitalkauto
tags:
  - gitalk
  - java
  - python
categories: Gitalk
---

![](8ac9761909ea434b8671d0c1708bf591.jpg)


## 前言

上篇博客讲了如何集成gitalk，如果你还不会集成gitalk [点击这里](https://llss6887.github.io/2018/09/20/blog-gitalk/ "点击这里")  快去集成你的gitalk吧，上次我们还有一个问题没有解决，那就是每次发文之后，需要手动的去初始化gitalk，就去抓了一下gitalk初始化发送的网络请求，自己写一个脚本来实现自动初始化，你也可以嵌到你的博客系统里，每次发新的文章都自动进行初始化，下面我们来看，如何实现。
<!--more-->
## 正文
在使用脚本之前，你需要获取一个权限，[Personal access tokens](https://github.com/settings/tokens "Personal access tokens")在当前页面为Token添加所有的repo权限
![](a8c6f8022a36430e98de451a20e34a73.png)

![](02d5f2c2dd614a75b1fc9c38667c03c4.png)
这样就获取到了Token，之后我们来看代码

python版本：

```python
# -*- coding:utf-8 -*-

import requests
from urllib import parse

username = 'git名称'
repos = '博客仓库'
title = '需要的初始化的博客标题'
label = '博客地址的相对路径 例如：/blog/article/121' 
domain = '博客首页地址  例如：http://heimamba.com'
token = '刚刚获取的Token'


def autoTalk(title, label):
    doUrl = 'https://api.github.com/repos/%s/%s/issues' % (username, repos)

    param = "{\"title\":\"%s\",\"labels\":[\"%s\", \"Gitalk\"],\"body\":\"%s%s\\n\\n\"}" % (title, label, domain, label)
    #param = parse.quote_plus(param)
    header = {"accept": "*/*", "connection": "Keep-Alive",
              "user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                            "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.108 Safari/537.36",
              "Authorization": "token %s" % token}
    sesson = requests.session()
    post = sesson.post(doUrl, data=param, headers=header)

```
java版本：

```java
import org.apache.http.HttpResponse;
import org.apache.http.client.HttpClient;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.protocol.HTTP;
import org.springframework.beans.factory.annotation.Autowired;

public class GitalkAuto {
	/**
     *
     * @param username  git用戶名
     * @param repo git博客的仓库名
     * @param uri 博客地址的相对路径 例如：/blog/article/121
     * @param blogUrl 博客首页地址  例如：http://heimamba.com
     * @param title 需要的初始化的博客标题
     * @param token 获取到的Token
     * @return
     * @throws Exception
     */
    public static Object autoTalk(String username,String repo, String uri, String blogUrl,String title, String token) throws Exception{
        String url = "https://api.github.com/repos/"+username+"/"+repo+"/issues";
        String param = String.format("{\"title\":\"%s\",\"labels\":[\"%s\", \"Gitalk\"],\"body\":\"%s%s\\n\\n\"}"
                                            , title, uri, blogUrl, uri);
        HttpClient client = HttpClients.createDefault();
        HttpPost post = new HttpPost(url);
        StringEntity entity = new StringEntity(param, HTTP.UTF_8);
        post.setHeader("accept", "*/*");
        post.setHeader("connection", "Keep-Alive");
        post.setHeader("user-agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.108 Safari/537.36");
        post.setHeader("Authorization", "token "+token);
        post.setEntity(entity);
        HttpResponse response = client.execute(post);
        return response.getEntity();
    }
}
```

上面的python版本和java版本，分别能初始化自己的博客评论，我的是将代码放到了发布文章的时候，自动初始化。
上面只是解决的，单个文章的问题，如果之前是其他评论系统，现在改用gitalk的，那岂不是需要一个个的执行。

因为我的博客比较简单，我就自己顺手写个批量初始化的，其实就是爬取文章的uri和title然后调用单个的初始化脚本，我的脚本如下：

```python
import requests
from lxml import etree

from talkAuto.TalkAuto import autoTalk

url = 'www.heimamba.com'
header = {"accept": "*/*", "connection": "Keep-Alive",
          "user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                        "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.108 Safari/537.36"}


def talkAuto(title, lable):
    for index, value in enumerate(title):
        t = value.strip()
        l = lable[index]
        autoTalk(t, l)


def get_url(uri):
    res = requests.get(uri, headers=header)
    html = etree.HTML(res.text)
    xpath = html.xpath('//*[@class="blog-content"]/a/@href')
    title = html.xpath('//*[@id="grid"]/li/section/article/text()')
    split_ = html.xpath("//li[@class='next']/a/@href")
    talkAuto(title,xpath)
    if split_:
        split_ = split_[0].split(';')[0]
        get_url(url+split_)


if __name__ == '__main__':
    get_url(url)
    print("done")
```

这个脚本只是适合我自己的博客，如果要其他博客也用的话，需要根据自己的实际情况更改xpath的值。

##结语

之前刚接触Gitalk的时候，网络上自动化的初始脚本都是ruby，对于我这个学渣来说，只能看个大概，无奈，只能自己写了，好了，希望这篇文章对你们有帮助，下面附上git的自动化gitalk的地址。


源码地址：[**戳这里**](https://github.com/llss6887/talkAuto "戳这里")

博客地址：[ **https://llss6887.github.io**]( https://llss6887.github.io " https://llss6887.github.io")
































































