安装python开发环境包括安装python,pip,pycharm,搜狗输入法

### 安装python,pip

ubuntu18.04内置了python3.6.9

```bash
python3
Python 3.6.9 (default, Nov  7 2019, 10:44:02) 
[GCC 8.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

但是没有安装pip

```bash
sudo apt instal python3-pip -y
```

有了pip3，接下来用pycharm `create new project`就不会出现`ModuleNotFoundError: No module named 'distutils.util'`错误了

- pip和pip3的区别

python 有python2和python3的区别
那么pip也有pip和pip3的区别
大概是这样的
1、pip是python的包管理工具，pip和pip3版本不同，都位于Scripts\目录下：
2、如果系统中只安装了Python2，那么就只能使用pip。
3、如果系统中只安装了Python3，那么既可以使用pip也可以使用pip3，二者是等价的。
4、如果系统中同时安装了Python2和Python3，则pip默认给Python2用，pip3指定给Python3用。
5、重要：虚拟环境中，若只存在一个python版本，可以认为在用系统中pip和pip3命令都是相同的

### 安装pycharm

安装很简单

想要创建桌面快捷方式可以在pycharm菜单栏中选择`tools`->`Create Desktop Entry..``

点击左下角Applications就可以看到了
<!--more-->
### 中文输入法

- 下载安装搜狗输入法

![image-20201201134537385](https://i.loli.net/2020/12/17/t3LVfFsvo5JORie.png)

![image-20201201134643498](https://i.loli.net/2020/12/01/RojFc865LHnB9MN.png)

![image-20201201134702354](https://i.loli.net/2020/12/01/KehZYaRO4DCUN52.png)

安装完之后ctrl+q关闭窗口

- 切换输入法系统

![image-20201201134927129](https://i.loli.net/2020/12/01/LhXDsCiv7YEBN9g.png)

- 切换输入法

![image-20201201135300861](https://i.loli.net/2020/12/01/fEpuNib486jXtcY.png)

![image-20201201135336435](https://i.loli.net/2020/12/01/ktY4OAM7LyniDJQ.png)