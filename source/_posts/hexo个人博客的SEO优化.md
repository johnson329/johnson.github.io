---
title: hexo个人博客的SEO优化
date: 2020-12-29 23:43:41
tags: [hexo]
categories: [hexo]
---


## 一、添加sitemap


也就是网站地图

1. 安装插件

需要安装两个插件来生成 sitemap 文件，前一个是传统的 sitemap，后一个是百度的 sitemap。

```sh
npm install hexo-generator-sitemap --save
npm install hexo-generator-baidu-sitemap --save
```

2. 修改站点配置文件

<!--more-->

```sh
vim _config.yml
```

```bash
sitemap: 
  path: sitemap.xml
baidusitemap:
  path: baidusitemap.xml
#你要修改成你自己的url
url: https://johnson329.github.io
```

3. 生成站点地图

```sh
hexo g
#会在public目录下面生成sitemap.xml和baidusitemap.xml两个文件
```

![image-20201229235532134](https://i.loli.net/2020/12/29/4jbxmlC5a3W2uAn.png)

## 二、生成robots.txt

在`source`目录下面新建`robots.txt`文件

![image-20201230000024312](https://i.loli.net/2020/12/30/dkmP3yARu8NeYL7.png)

```
#hexo robots.txt
User-agent: *

Allow: /
Allow: /archives/
Allow: /categories/
Allow: /tags/

Disallow: /vendors/
Disallow: /js/
Disallow: /css/
Disallow: /fonts/
Disallow: /vendors/
Disallow: /fancybox/


Sitemap: https://johnson329.github.io/sitemap.xml
Sitemap: https://johnson329.github.io/baidusitemap.xml
```

## 三、提交站点到Google

打开[Google Search Console](https://www.google.com/webmasters/)，添加博客地址。

- 输入网址

![image-20201230001355099](https://i.loli.net/2020/12/30/IfLaYNJsiUoRKvO.png)

- 验证所有权

把给定的元标记复制到 `themes\next\layout\_partials\head\head.njk`下面

![image-20201230000843195](https://i.loli.net/2020/12/30/FxgivOn3aYrmpAc.png)



![image-20201230165916727](https://i.loli.net/2020/12/30/2hV1Nr4gK7o3wEd.png)

- 查看控制台

  那就等1天吧
  
  ![image-20201230001647568](https://i.loli.net/2020/12/30/dM5otXDI9gQ2CA8.png)	

- 提交

![image-20201231002548948](https://i.loli.net/2020/12/31/QJWBtjzAhlwn4ux.png)

## 四、提交站点到百度

百度的感觉藏得很深，这块业务是不做了吗？

有空再搞

## 五、出站链接添加nofollow标签

网络爬虫会在当前页面搜索所有的链接，然后一个个查看，所以就很有可能跳到别的网站就不回来了。这个时候就需要`nofollow`起作用了。

带有nofollow属性的任何出站链接都不会被搜索引擎搜索

先安装插件 `npm install hexo-abbrlink --save`

修改站点的配置文件 `_config.yml`

```yaml
# 外部链接优化
nofollow:
  enable: true
  field: site
# 友链
#  exclude:
#    - 'exclude1.com'
#    - 'exclude2.com'
```

```sh
hexo clean && hexo g && hexo s
```

查看底部的外链，我们发现rel属性已经多了一个 `nofollow`

![image-20201230003637658](https://i.loli.net/2020/12/30/PO4bsLTFEnM6oDV.png)

 

## 六、首页优化

把站点下的`index.njk`文件修改

`themes\next\layout\index.njk`

把 `{{title}}`换成  

```sh
{{ theme.keywords }} - {{ config.title }}{{ theme.description }}
```




## 七、参考链接

[Hexo博客Next主题SEO优化方法](https://hoxis.github.io/Hexo+Next%20SEO%E4%BC%98%E5%8C%96.html#comments)

[Hexo-NexT (v7.7.2) 主题配置](https://blog.csdn.net/iwyang/article/details/106989237)

[Hexo 个人博客 SEO 优化（3）：改造你的博客，提升搜索引擎排名](https://juejin.cn/post/6844903600485826567#heading-8)