---
title: Flask源码之Request对象（六）
date: 2021-01-06 23:04:34
tags: [flask源码]
categories: [flask]
---



## 引言

我们知道 

```python
from flask import request
```

是一个代理对象，实际上代理的是 `RequestContext`的`request`属性

```python
class RequestContext(object):

    def __init__(self, app, environ, request=None, session=None):
        self.app = app
        if request is None:
            request = app.request_class(environ)
        self.request = request
        self.url_adapter = None
        try:
            self.url_adapter = app.create_url_adapter(self.request)
        except HTTPException as e:
            self.request.routing_exception = e
        self.flashes = None
        self.session = session
```

那么`request`属性又是什么什么呢？

<!--more-->

## 原理与实现

```python
request = app.request_class(environ)
```

`request_class`定义如下

![image-20210106233013649](https://i.loli.net/2021/01/06/rZFvHm12f84nTQc.png)

从上面代码可以看出来，`request`是 `request_class`的实例，`request_class`是一个`Request`类

定义在`flask`的`wrappers.py`中



```python
class Request(RequestBase, JSONMixin):
```

`mixin`是`python`多继承的方式，从面向对象的语义来看，表达的是 `has`关系而不是 `is`关系，用于为子类提供一些额外的方法

相当于`java`的`interface`

我们知道我们用的`request`实际上是这个`Request`的实例就好了


### 继承关系





![image-20210106235017437](https://i.loli.net/2021/01/06/b4ypciGuervH6Tn.png)




我们重点看一下 `JSONMixin`的实现

`flask.wrappers.JSONMixin`

```python
class JSONMixin(_JSONMixin):
    json_module = json

    def on_json_loading_failed(self, e):
        if current_app and current_app.debug:
            raise BadRequest("Failed to decode JSON object: {0}".format(e))

        raise BadRequest()
```

有一个 `on_json_loading_failed`方法，他的作用是什么呢

### on_json_loading_failed

#### 作用

我们知道前后端以`json`方式交互，对于前端传过来的数据我们要`load`成字典，`json.load(environ中前端传过来的json数据)`

那么失败了就会调用这个函数

我们可以 继承`Request`，写一个自定义的对象，重写这个方法，并赋值给`Flask`类



```python
from flask import Flask, jsonify, Request
from werkzeug.exceptions import BadRequest

flask_app = Flask(__name__)

class MyRequest(Request):
    def on_json_loading_failed(self, e):
        raise BadRequest("我是自己写的错误描述")


flask_app.request_class = MyRequest


@flask_app.route('/', endpoint="11", methods=["GET", "POST"])
def hello_world():
    return jsonify(code=0, msg="success")


if __name__ == '__main__':
    flask_app.run()
```

这样，当`load`失败后我们就可以自己决定该怎么办

值得注意的是，默认情况下，`json.load`失败了会`raise BadRequest`,结合`flask`中的异常处理方式，我们也可以把他用 `errorhandler`处理，例如`@flask_app.errorhandler(BadRequest)`

### _JSONMixin

按两下 `shift`搜索这个父类 `_JSONMixin`，注意他是被重命名了,我们要搜索 `JSONMixin`，只不过位置在 `werkzeug`中



定义在 `werkzeug`中

![image-20210106234345164](https://i.loli.net/2021/01/06/WiIjaN98Zrh1KEV.png)



我们看到这个类有个一`json`方法用于获取前端的`json`数据。根据继承关系，我们可以

```python
from flask import request
json_data=request.json()
```




## json_module

此外我们还看到两个 `JSONMixin`都有 `json_module`属性，属性值`json`对象，这个`json`是什么？是标准库的`json`吗？？？

```python
class JSONMixin(_JSONMixin):
    json_module = json

    def on_json_loading_failed(self, e):
        if current_app and current_app.debug:
            raise BadRequest("Failed to decode JSON object: {0}".format(e))

        raise BadRequest()
```

查看 `json` 源码知道，这里`json_module`属性的作用是**指定了序列化反序列化的模块**,这个`json`不是标准库的，而是自定义的

位置在 `flask.json`

![image-20210106235551849](https://i.loli.net/2021/01/06/eM6VU5nutdcaBF2.png)

我们很容易在其中找到`loads`等方法，注意到其中的`_json`就是标准库的`json`库，所以他是对标准库`json`的封装

当然，我们也可以用自己的`json`模块

例如你想不开想使用`pickle`（跑路吧）

```python
class MyRequest(Request):
    json_module=pickle
    def on_json_loading_failed(self, e):
        raise BadRequest()
```

## 参考文章

[flask-insight-request](https://cizixs.com/2017/01/18/flask-insight-request/)