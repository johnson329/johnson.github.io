---
title: Flask源码之一个请求的旅程（二）
date: 2020-12-24 15:05:41
tags: [flask源码]
categories: [flask]
---

### 入口

由上一篇文章我们知道 `Flask` 要有如下函数或者一个实现了 `__call__`方法的类，才可以被`wsgi server`调用



```python
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return [b'<h1>Hello, web!</h1>']
```

```python
class Applications(object):
	def __call__(self,environ, start_response)
    	start_response('200 OK', [('Content-Type', 'text/html')])
    	return [b'<h1>Hello, web!</h1>']
# 实现了__call__方法的类就可以像函数一样被调用
# 例如 application=Applications()
# application(environt,start_response)
```

那 `Flask`是怎么做的呢？ 我们来看一下简单的 `Flask demo`

<!--more-->

```python
from flask import Flask

flask_app = Flask(__name__)


@flask_app.route('/')
def hello_world():
    return "hello flask"


# python3 my_wsgi_server.py flask_app:flask_app
if __name__ == '__main__':
    flask_app.run()

```



这段代码很简单，先是创建了一个`Flask`实例（对象），然后调用`Flask`对象的`route`方法。

启动项目的时候调用`Flask`对象的`run()`方法

我们先不去看`run`方法内部是如何实现的，因为Flask内置了一个`wsgi server`，从 `run`方法进入会涉及到很多 `wsgi server` 的代码，而生产环境我们一般不用这个 `wsgi server`，而是用`gunicorn`或者`uwsgi`。

在 `pycharm`中按 `ctrl`，鼠标左键点击 `Flask`查看源码

我们可以看到Flask是一个类，那么他一定实现 `__call__`方法



![Flask源码](https://i.loli.net/2020/12/24/vYRJB68Xpdy3g2r.png)



点击 `pycharm`左侧的 `structure`我们可以看到果然有 `__call__`方法，而且这个 `__call__`方法的参数是 `environ,start_response`，`wsgi server`最后调用的也是这个方法

![image-20201224152301791](https://i.loli.net/2020/12/24/XaSczmoUrh9fOZk.png)



我们进入到这个方法，随便打个断点

### 启动项目

接下来我们以 `debug`的方式启动项目

![image-20201224152851475](https://i.loli.net/2020/12/24/p5SqLCVibvXWm3a.png)

浏览器访问 `http://127.0.0.1:5000/`,我们看到 `pycharm`在 我们的断点处停了下来，`__call__`方法又调用了`wsgi_app`方法，参数都一摸一样

![image-20201224153123774](https://i.loli.net/2020/12/24/ZKEONfHDgLvItPV.png)



再看 `wsgi_app`方法



```python
    def wsgi_app(self, environ, start_response):

        ctx = self.request_context(environ)
        error = None
        try:

            try:
                ctx.push()
                response = self.full_dispatch_request()
            except Exception as e:
                error = e
                response = self.handle_exception(e)
            except:  # noqa: B001
                error = sys.exc_info()[1]
                raise

            return response(environ, start_response)
        finally:
            if self.should_ignore_error(error):
                error = None
            ctx.auto_pop(error)
```

这个方法其实就做了以下几件事

1. 路由：就是根据你的请求，找到对应的视图函数。例如我们的请求 `http://127.0.0.1:5000/` 会被 `@flask_app.route('/')`匹配到，那么就会调用 `hello_world`方法
2. 处理请求上下文和应用上下文
3. 错误处理
4. 回调  `wsgi server`提供的 `start_response` 方法把结果返回给  `wsgi server`

接下来分别介绍这几个部分

