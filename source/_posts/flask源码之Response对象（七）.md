---
title: flask源码之Response对象(七)
date: 2021-01-15 22:46:09
tags: [flask源码]
categories: [flask]
---



### 原理

![image-20210115225826137](https://i.loli.net/2021/01/15/xJOnfBLewgpsUbY.png)

### 实现

<!--more-->

`full_dispatch_request`方法

![image-20210115230023316](https://i.loli.net/2021/01/15/Imc5eBVTUFQ7iKZ.png)



回到`wsgi_app`方法

![image-20210115230235510](https://i.loli.net/2021/01/15/RxJWtNSTufI3sdU.png)



回顾一下`wsgi`协议



你要被`wsgi server`调用，那么你的返回值，应该是可以被迭代的，例如下面的`list`

```python
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return [b'<h1>Hello, web!</h1>']
```

也就是在`flask`里面，这个`Response`对象，执行 `__call__`方法之后返回的是一个类似 `list`的可迭代对象

我们知道既然`Response`对象是被`finalize_request`方法返回的，那他的实例化过程应该就在这个方法中

我们找到 `finalize_request`方法



```python
    def finalize_request(self, rv, from_error_handler=False):
        response = self.make_response(rv)
        try:
            response = self.process_response(response)
            request_finished.send(self, response=response)
        except Exception:
            if not from_error_handler:
                raise
            self.logger.exception(
                "Request finalizing failed with an error while handling an error"
            )
        return response
```



再看 `make_response`，代码有点多

这个函数的大概逻辑就是，你的rv是视图函数的返回值

如果返回值本身就是 Response 实例，就直接使用它；如果返回值是字符串类型，就把它作为响应的 body，并自动设置状态码和头部信息；
如果返回值是 tuple，会尝试用 (response, status, headers) 或者 (response, headers) 去解析。





```python
    def make_response(self, rv):

        # make sure the body is an instance of the response class
        # 看这里就好了，
        if not isinstance(rv, self.response_class):
            if isinstance(rv, (text_type, bytes, bytearray)):
                # let the response class set the status and headers instead of
                # waiting to do it manually, so that the class can handle any
                # special logic
                rv = self.response_class(rv, status=status, headers=headers)
                status = headers = None
            elif isinstance(rv, dict):
                rv = jsonify(rv)
            elif isinstance(rv, BaseResponse) or callable(rv):
                # evaluate a WSGI callable, or coerce a different response
                # class to the correct type
                try:
                    rv = self.response_class.force_type(rv, request.environ)
                except TypeError as e:
                    new_error = TypeError(
                        "{e}\nThe view function did not return a valid"
                        " response. The return type must be a string, dict, tuple,"
                        " Response instance, or WSGI callable, but it was a"
                        " {rv.__class__.__name__}.".format(e=e, rv=rv)
                    )
                    reraise(TypeError, new_error, sys.exc_info()[2])
            else:
                raise TypeError(
                    "The view function did not return a valid"
                    " response. The return type must be a string, dict, tuple,"
                    " Response instance, or WSGI callable, but it was a"
                    " {rv.__class__.__name__}.".format(rv=rv)
                )

        # prefer the status if it was provided
        if status is not None:
            if isinstance(status, (text_type, bytes, bytearray)):
                rv.status = status
            else:
                rv.status_code = status

        # extend existing headers with provided headers
        if headers:
            rv.headers.extend(headers)

        return rv
```

我们还可以看出来，`Response`对象就是 `response_class`的实例，点进去就可以看到定义了



### Response定义

```python
class Response(ResponseBase, JSONMixin):
    default_mimetype = "text/html"

    def _get_data_for_json(self, cache):
        return self.get_data()

    @property
    def max_cookie_size(self):
        """Read-only view of the :data:`MAX_COOKIE_SIZE` config key.

        See :attr:`~werkzeug.wrappers.BaseResponse.max_cookie_size` in
        Werkzeug's docs.
        """
        if current_app:
            return current_app.config["MAX_COOKIE_SIZE"]

        # return Werkzeug's default when not in an app context
        return super(Response, self).max_cookie_size
```

又是`mixin`实现多继承

别忘了，我们的目的是看他的 `__call__`方法返回的是否是迭代对象



不用大海捞针去父类找，`pycharm`已经帮我们找好了，在左侧`structure`中找



![image-20210115231514061](https://i.loli.net/2021/01/15/rBpN1JfXC5WI7Eh.png)



```python
    def __call__(self, environ, start_response):
        """Process this response as WSGI application.

        :param environ: the WSGI environment.
        :param start_response: the response callable provided by the WSGI
                               server.
        :return: an application iterator
        """
        app_iter, status, headers = self.get_wsgi_response(environ)
        start_response(status, headers)
        return app_iter
```

**看名字也知道，`app_iter`是可迭代对象，可以被`wsgi server`迭代调用**

`start_response`是 `wsgi server`提供的回调函数，我们调用就好了，`wsgi server`会帮我们返回`http response`

再看一眼 `wsgi`协议的`demo`

```python
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return [b'<h1>Hello, web!</h1>']
```

可以发现 `header`的格式要是`list`套 `tuple`这种

`werkzeug`实现了`Headers`数据结构

定义在

```
fro werkzeug.datastructures import Headers
```

他是一个类似字典的数据结构，只不过支持多个相同的`key`以及`key`的有序

参考[Headers对象](https://cizixs.com/2017/01/22/flask-insight-response/)



### 自定义`Response`对象

我们知道`Reseponse`对象是在 `make_response` 种这样被实例化的

```python
rv = self.response_class(rv, status=status, headers=headers)
```



因此我们只需要

```python
from flask import Flask, Response

class MyResponse(Response):
    pass

app = Flask(__name__)
app.response_class = MyResponse

```

便可以定义我们自己的Response对象