---
title: hexo配置菜单、分类、标签、预览、scheme(二)
tags: [hexo,博客]
categories: [hexo]
---

### 语言、时区

`_config.yml`

```yaml
language: zh-CN
timezone: 'Asia/Shanghai'
```



### 菜单

`themes/next/_config.yml`

去掉相应注释即可。

菜单配置的结构是`key:路由`

<!--more-->

```yaml
menu:
  home: / || fa fa-home
  about: /about/ || fa fa-user
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
  archives: /archives/ || fa fa-archive
```



### 分类

去掉菜单配置项的注释后，页面多了菜单，但是点击菜单会显示页面不存在

我们需要创建对应的页面

- 创建分类页面

```bash
hexo new page categories
#source目录下会生成categories目录
vim source/categories/index.md
```

- 编辑分类页面

```yaml
---
title: 分类
date: 2020-06-21 23:41:18
type: "categories"
---
```

不加`type: "categories"` 的话，分类页面无法统计有多少分类

- 使用分类

在文章的开头添加，例如`vim source/_posts/搭建hexo博客.md`

```markdown
---
title: 搭建 hexo 博客
categories: [hexo]
---
```

这篇博客就属于hexo分类



### 标签

- 创建标签页面

```yaml
hexo new page tags
#source目录下会生成categories目录
vim source/tags/index.md
```

- 编辑标签页面

```
---
title: 标签
date: 2020-06-21 23:38:14
type: "tags"
---
```

- 使用标签

在博客文章的开头添加以下内容，这篇博客就有了”hexo“和”博客“两个标签

```markdown
---
title: 搭建 hexo 博客
tags: [hexo,博客]
categories: [hexo]
---
```



### 首页文章预览

在文章中加入注释`<!--more-->` ，注释以上的内容就是首页预览的内容



### 主题的几种scheme

我的理解就是next主题的几种布局，在`themes/next/_config.yml`中搜索scheme，取消注释即可选择你喜欢的scheme

```yaml
# Schemes
#scheme: Muse
#scheme: Mist
#scheme: Pisces
scheme: Gemini
```





### 参考资料

[Github+Hexo使用NexT主题和优化](https://www.jianshu.com/p/67b731c3f840)



