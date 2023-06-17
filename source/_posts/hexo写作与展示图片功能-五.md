---
title: hexo写作与展示图片功能(五)
date: 2020-11-27 04:57:05
tags: [hexo]
---

### 安装picgo

**PicGo: 一个用于快速上传图片并获取图片 URL 链接的工具**

### 图床

- 注册[sm.ms](https://sm.ms/)账号


获取token，在picgo中设置sm.ms的token，选中markdown

- 配置picgo server并重启

<!--more-->

![image-20201127052314378](https://i.loli.net/2020/11/27/1Gaxqr9HTnXuwmQ.png)

### 配置typora

`文件`->`偏好设置`->`图像`

如下图所示设置

![image-20201127051821349](https://i.loli.net/2020/11/27/uFKRYzGodq5IDXT.png)

### 上传图片

使用微信截图或者用操作系统自带的截图工具截图，直接粘贴到typora中即可生成markdown代码

### 原理

原理就是typora会把图片发送给 picgo server ,picgo server返回包含图片url的markdown代码

### 图床迁移

picgo-migrater，不过好像不太好用的样子。

自己搞也可以，读取md文件，利用正则匹配图片，下载，然后上传到新的图床，并把返回的url转化为原来的url，逻辑很清晰