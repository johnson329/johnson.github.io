---
title: 搭建 hexo 博客(一)
tags: [hexo,博客]
categories: [hexo]
---
### 环境

ubuntu 20.04

node 10.21.0

### 下载和安装nvm

nvm是node.js的版本管理器。使用nvm可以避免出现`EACCES` 权限错误和方便切换node.js和npm版本。

- 执行以下命令下载和安装nvm

注意可能由于网络问题这个命令执行没成功，请在代理环境下执行以下命令

```sh
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash
```

- 验证安装

```sh
command -v nvm
#或者
nvm
```

- 出现`nvm: command not found`或者没有输出nvm

重启terminal或者参考[#1404](https://github.com/nvm-sh/nvm/issues/1404)解决

仍有问题请google或者参考项目的[README](https://github.com/nvm-sh/nvm)或者[github issue](https://github.com/nvm-sh/nvm/issues)解决

<!--more-->

### 下载和安装Node.js和npm

- 查看所有可供安装的node版本：

```sh
nvm ls-remote
```

- 下载node.js

```sh
#下载指定版本的node.js
nvm install 10.21.0
#如果想下载最新版本的node.js
nvm install node 
```

- 查看本地已安装的node.js

```sh
nvm ls
```



### 安装hexo

```sh
npm install -g hexo-cli
```



### 搭建博客

- 初始化hexo项目

```sh
#新建一个项目，如果没写projectname，当前目录名就是项目名
hexo init projectname
```

- 选择主题

  我是参考这个[知乎问答](https://www.zhihu.com/question/24422335)选择的[next主题](https://github.com/iissnan/hexo-theme-next/blob/master/README.cn.md)，不过next官方版本已经停止维护了，官方推荐使用[next社区维护版](https://github.com/theme-next/hexo-theme-next)，但这个仓库更新也比较慢，而且代码无法高亮的问题难以解决。所以我用的是这个项目的维护者开源的另外一个[版本](https://github.com/next-theme/hexo-theme-next)


- 下载next主题

```sh
cd projectname
#下载next主题到themes/next目录
git clone https://github.com/next-theme/hexo-theme-next themes/next
```

- 切换到hexo主题。修改hexo的_config.yml配置文件，`vim _config.yml`

```yaml
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: next
```

- 启动服务器

```sh
hexo server
```

- 写一篇博客

```sh
hexo new "博客标题"
#找到文章
cd /source/_posts
#编辑文章
vim 博客标题.md
```

- 开启代码高亮

编辑hexo的_config.yml文件

```yaml
highlight:
  enable: true
  line_number: true
  auto_detect: true
  tab_replace: ''
  wrap: true
  hljs: true
```

`cd themes/next`

编辑next的_config.yml文件，自行选择代码高亮方式

如果高亮不生效，尝试hexo generate生成静态文件

```yaml
codeblock:
  # Code Highlight theme
  # See: https://github.com/highlightjs/highlight.js/tree/master/src/styles
  theme:
    light: darcula
    dark: tomorrow-night
```

你可以选择多种代码高亮样式

参考[highlight demo页面](https://highlightjs.org/static/demo/)选择你喜欢的代码高亮样式



### 部署

- 新建github仓库或者gitee仓库

仓库名必须为 <github的账户名>.github.io

- 配置github地址

修改配置文件，告诉hexo远程仓库的地址

```yaml
deploy:
  type: git
  repo: https://github.com/johnson329/johnson.github.io
  branch: master
```

- 安装hexo-deployer-git

它可以帮助我们把项目推送到远程仓库

```bash
npm install hexo-deployer-git –save
```

- 配置git

```bash
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

- 部署

```bash
hexo clean 
hexo deploy #或者hexo d
#输入github的用户名和密码，大功告成，如果想不输入用户名和密码，把配置中的repo换成ssh地址就好了
#至此，静态文件（html,css,js以及markdown）都被同步到远程仓库的master分支了
```







