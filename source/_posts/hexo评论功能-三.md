---
title: hexo评论功能(三)
tags: [hexo,博客]
categories: [hexo]
---

### 引言

​        next主题内置多种第三方评论系统，如disqus

```yaml
comments:
  # Available values: tabs | buttons
  style: tabs
  # Choose a comment system to be displayed by default.
  # Available values: changyan | disqus | disqusjs | gitalk | livere | valine
```

但是disqus由于种种原因无法访问，valine需要身份证号码进行实名登记

所以我选择了gitalk，他是利用github的issue来作为评论，真是github全家桶

<!--more-->

### 开启gitalk

```yaml
gitalk:
  enable: true # 开启
  github_id: github账户名 # github账户名
  repo: johnson329.github.io # 博客的仓库名
  client_id:  # GitHub Application Client ID
  client_secret:  # GitHub Application Client Secret
  admin_user: johnson329 # github账户名
  distraction_free_mode: true # Facebook-like distraction free mode
  # Gitalk's display language depends on user's browser or system environment
  # If you want everyone visiting your site to see a uniform language, you can set a force language value
  # Available values: en | es-ES | fr | ru | zh-CN | zh-TW

```



### 创建Github Oauth App

`settings`>`developer settings`>`oauth apps`

- 把`clent_id`和`client_secret`填到上面的配置中

![github oauth 配置](https://i.loli.net/2020/11/27/doYkumgXC93bhGN.png)



- 设置回调地址（页面调用github，github完成用户认证，回过头来重定向到你的主页，叫回调）

![github oauth 回调地址](https://i.loli.net/2020/11/27/Y7HB1iKl2Fjfa8N.png)



