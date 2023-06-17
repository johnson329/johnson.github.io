---
title: ubuntu18.04安装docker
date: 2020-12-28 21:10:12
tags: [docker]
categories: [docker]
---

## 系统依赖
对于ubuntu操作系统，Docker Engine目前支持以下几个版本

 - Ubuntu Focal 20.04 (LTS)
 - Ubuntu Bionic 18.04 (LTS)
 - Ubuntu Xenial 16.04 (LTS)

在系统架构上，Docker Engine目前只支持`x86_64 `or`amd64`,`armhf`,`arm64 `等系统架构

<!--more-->

 - 查看自己的系统架构

```yaml
arch
```


## 卸载旧的版本

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```
如果提示

```bash
Reading package lists... Done
E: Could not get lock /var/lib/apt/lists/lock - open (11: Resource temporarily unavailable)
E: Unable to lock directory /var/lib/apt/lists/
```
很可能是你的ubuntu操作系统开启了每天自动更新，apt-get被占用了，关掉自动更新，或者等他自动更新完
[How to Fix ‘E: Could not get lock /var/lib/dpkg/lock’ Error in Ubuntu Linux](https://itsfoss.com/could-not-get-lock-error/)

## 添加docker 官方仓库

 1. 更新apt的package index并允许apt使用https

 如果因为网络问题更新速度慢的话，请用你的网络代理工具开启全局代理
 如果你连网络代理工具也没有的话，自行解决
```bash
$ sudo apt-get update

$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```


2. 添加docker官方的GPGkey

```bash
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

```
验证一下你的key的签名是否`9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88`.搜索后八位即可

```bash
$ sudo apt-key fingerprint 0EBFCD88

pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
```

3. 添加docker仓库

根据你的ubuntu的系统架构安装docker engine下面是amd64的安装命令
```bash
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
其他系统架构的命令请参考[官方文档](https://docs.docker.com/engine/install/ubuntu/#set-up-the-repository)

## 安装docker engine



1. 继续更新 apt的package index 然后安装docker engine

```bash
 $ sudo apt-get update
 $ sudo apt-get install docker-ce docker-ce-cli containerd.io
```
要安装指定版本的docker engine 请参考[官方文档](https://docs.docker.com/engine/install/ubuntu/#install-docker-engine)
2. 验证docker engine 是否安装成功

```bash
$ sudo docker run hello-world

```
可以看到这个命令先去本地寻找hello-world:latest镜像，找不到就去dockerhub仓库拉，然后根据拉取镜像生成容器运行，运行结束退出容器
![docker hellow-world](https://i.loli.net/2020/12/28/RcqWIN5ymYXfZHS.png)
Docker 需要用户具有 sudo 权限，为了避免每次命令都输入sudo，可以把用户加入 Docker 用户组，然后重新登陆linux用户就好了，注意是你要注销当前用户再登陆

```bash
sudo usermod -aG docker $USER
```
镜像下载加速
使用阿里云镜像源下载

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxx.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

[docker镜像下载加速](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)



## 安装docker-compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```

## 脚本一键安装

装不好就算了，我自己写的

```
git clone git@github.com:johnson329/MyUtils.git
cd MyUtils/bin
./install_docker_ubuntu18.04.sh
```

## Image和Container概念

- image

**Docker 把应用程序及其依赖，打包在 image 文件里面**

- container

**根据image生成的容器**

image 文件可以看作是容器的模板。Docker 根据 image 文件生成容器的实例。同一个 image 文件，可以生成多个同时运行的容器实例。

更多概念请参考[阮一峰的博客](http://www.ruanyifeng.com/blog/2018/02/docker-tutorial.html)

## docker安装mysql服务

再看一个真实一点的例子，用docker安装mysql

```bash
# 拉取mysql
docker pull mysql
# 启动docker 容器，--name表示容器名叫pwc-mysql -e设置环境变量
# 这个容器会根据环境变量MYSQL_ROOT_PASSWORD设置root用户的密码 
# -p 表示把容器内的3306端口映射到容器外的宿主机 宿主机:容器
# -d表示后台运行，而不是敲完命令我就没地方敲其他命令了
# 最后的mysql是镜像的名字
docker run --name pwc-mysql -e MYSQL_ROOT_PASSWORD=123456 -p 3306:3306 -d mysql
```

这样，一个mysql容器就起来了，我们可以用navicat测试一下连接

## docker-compose安装mysql服务

每次都要敲命令实在太麻烦了，这次敲完下次又要重新来一遍，简直噩梦

更噩梦的是如果有多个服务，比如还有redis，rabbitmq，都要去敲命令，那简直累死

所以，相比于命令式的启动方式

我们还可以用声明的方式启动vim docker-compose.yml

```yml
version: '3'
services:
  mysql-db:
    container_name: mysql-docker
    image: mysql:8.0
    ports:
    - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: test
  rabbitmq:
      image: rabbitmq:3.8.3-management
      container_name: rabbitmq-docker
      restart: always
      hostname: myRabbitmq
      ports:
        # 管理界面
        - 15672:15672
        - 5672:5672
      volumes:
        - ./rabbitmq/data:/var/lib/rabbitmq
      environment:
        - RABBITMQ_DEFAULT_USER=root
        - RABBITMQ_DEFAULT_PASS=123456
```

这里声明了两个service，名字分别为mysql-db和rabbitmq

剩下的都是对应docker run后面的参数了，注意volumes表示的是把容器内外的文件夹作映射，比如mysql产生的数据我们不能放在容器里面，万一容器被删除了，数据就没了，所以我们一般把容器内的数据用`volumes`挂在出来。

而且，`mysql`需要的配置文件，我们一般会实现写好，然后映射到`mysql`容器内部，让容器使用这个启动配置文件，这些都是通过`volumes`实现的

```bash
docker-compose -f docker-compose.yml up -d
```

这样我们就安装了mysql和rabbitmq服务

当然像这样的`docker-compose.yml`你可以去网上找，只要你理解了文件的意思即可。这可比以前那种安装包的方式轻松多了

## 制作自己的image镜像

例如我们写了一个 `flask`项目，像这样 `app.py`

```python
from flask import Flask

app = Flask(__name__)


@app.route('/')
def hello_world():
    return 'Hello World!'


if __name__ == '__main__':
    app.run()

```

`requirements.txt`

```
click==7.1.2
Flask==1.1.2
gunicorn==20.0.4
itsdangerous==1.1.0
Jinja2==2.11.2
MarkupSafe==1.1.1
Werkzeug==1.0.1
```

我们可以把代码提交到仓库，然后在部署服务器上拉下代码，然后用gunicorn部署，像这样 `gunicorn -w 4 app:app`

如果用docker部署呢？

1. 制作Dockerfile文件

```dockerfile
FROM python:3.6
# 指定容器内的工作目录
WORKDIR /flask_project
# 把宿主机的当前目录下的内容 拷贝到 flask_project目录下面
COPY . .
RUN pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# 启动命令
CMD ["gunicorn", "-w", "4", "app:app"]
```

2. 打包镜像

```bash
docker build -t flask_project_image:0.1 .
```

`-t`表示给镜像打上flask_project_image:0.1的标签 

.表示使用当前目录下的Dcokerfile文件

3. 启动

```bash
docker run --name flask_project_container -d flask_project_image:0.1
```

表示以**后台启动**的方式启动一个名为`flask_project_container`的容器

镜像是 `flask_project_image:0.1`

- 查看日志

```bash
docker logs -f flask_project_container
```

- 进入到容器内部

```bash
docker exec -it flask_project_container bash
```

- 停止容器，但是容器没有被删除，只是不运行了

```bash
docker stop flask_project_container
```

- 删除容器

```bash
docker rm flask_project_container
```

## 容器管理

使用容器我们可以很方便的部署我们的项目，一般项目是从单体开始，等流量上来以后，我们可以采用单体集群的方式，也就是再启动一个`flask`项目容器，前面放一个网关，让网关做负载均衡，这样就可以很容易横向扩展，当然前提瓶颈在代码运行环节。

但流量有高峰有低估，总不能一直跑着几十个容器占用服务器资源

我们希望我们的容器能根据负载，例如根据cpu占用率或者内存占用率自动扩容（增加容器）缩容（减少容器）

这就是`kubernets`又叫`k8s`做的事情，它是一个容器集群管理系统，可以做到自动化部署、自动扩容和自动缩容

不过谷歌今年服务也出现了短时间大面积不可用，也不咋地嘛

`k8s`的教程请看[董明伟的博客](https://dongwm.com/post/use-kubernetes-2/)