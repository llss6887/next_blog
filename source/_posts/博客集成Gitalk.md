---
title: 博客集成Gitalk
date: 2018-09-20 20:36:47
urlname: blog-gitalk
tags:
  - blog
  - gitalk
categories: Gitalk
---
![](0b58d904becb46edb75a3a72c0d70eca.jpg)

## 前言
由于之前的评论是自己写的一个，之后发现gitalk就体验了一把，果断换成gitalk

## 关于Gitalk

Gitalk 是一个基于 Github Issue 和 Preact 开发的评论插件。使用 Github 帐号登录，界面干净整洁，最喜欢的一点是支持 MarkDown语法
<!--more-->
主要特性：

使用 Github 登录
支持多语言 [en, zh-CN, zh-TW, es-ES, fr]
支持个人或组织
无干扰模式（设置 distractionFreeMode 为 true 开启）
快捷键提交评论 （cmd|ctrl + enter）
支持MarkDown语法

![](f78aa36cdf604c428c8aa2d62c76c7eb.png)

![](aab601d2245c4160846d2ddb25137e81.png)
## 原理
Gitalk 是一个利用 Github API,基于 Github issue 和 Preact 开发的评论插件，在 Gitalk 之前还有一个 gitment 插件也是基于这个原理开发的,不过 gitment 已经很久没人维护了。

## 申请client id和 client secret
需要 GitHub Application，如果没有 [点击这里申请](https://github.com/settings/applications/new "点击这里申请")，Authorization callback URL 填写当前使用插件页面的域名。

![](28e19d97ea02428588f01b29cb562a0a.png)
之后回去到clientID和Client Secret
![](46e4a5caf2fa4a29bf2b734f9619b2d0.png)

## 安装

两种方式

直接引入
```html
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/gitalk@1/dist/gitalk.css">
  <script src="https://cdn.jsdelivr.net/npm/gitalk@1/dist/gitalk.min.js"></script>
```

  <!-- or -->

```html
<link rel="stylesheet"href="https://unpkg.com/gitalk/dist/gitalk.css">
<script src="https://unpkg.com/gitalk/dist/gitalk.min.js"></script>
```
npm 安装
```go
npm i --save gitalk
import 'gitalk/dist/gitalk.css'
import Gitalk from 'gitalk'
```
使用
添加一个容器在你的博客合适的位置：

```html
<div id="gitalk-container"></div>
```
用下面的 Javascript 代码来生成 gitalk 插件：

```javascript
var gitalk = new Gitalk({
  clientID: '你的clientid',
  clientSecret: '你的clientSecret',
  repo: '你的博客的仓库名',
  owner: '你的git用户名',
  admin: ['这个是管理账户  也填git用户名'],
  id: location.pathname,      // 这里会生成一个id，根据当前页面的path  也可以自己定义
  distractionFreeMode: false  // 是否开启全屏遮罩 
})
gitalk.render('gitalk-container')
```

将上述的信息填好之后，访问你的博客，就可以出现评论框了。

![](3e6c36c38e884c3091a5d7088d595c93.png)

但是gitalk有一点很烦，就是每次发新的博客，都需要手动初始化一下，对于之前是其他评论系统想迁移到gitalk的就比较烦了，没关系，我们有自动初始化的脚本。[**点击这里 **](https://llss6887.github.io/2018/09/21/gitalkauto/ "点击这里 ")去看如何编写自己的初始化脚本。





博客地址：[ **https://llss6887.github.io**]( https://llss6887.github.io " https://llss6887.github.io")






















































