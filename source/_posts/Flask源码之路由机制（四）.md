---
title: Flask源码之路由机制（四）
date: 2020-12-30 23:31:47
tags: [flask源码]
categories: [flask]
---

## 原理

一个 web 应用中，不同的路径会有不同的处理函数，**路由就是根据请求的 URL 找到对应处理函数的过程。**

在下面的例子中，就是根据"/"找到`hello_world`的过程

```python
from flask import Flask, request

flask_app = Flask(__name__)


@flask_app.route('/',endpoint="11")
def hello_world():
    return "{}".format(request.remote_addr)


if __name__ == '__main__':
    flask_app.run()
```

我们很容易想到用字典去做路由，`key`是 `url`，`value`是对应的处理函数或者叫视图函数

<!--more-->

```python
{"/":hello_world}
```

但是对于动态路由，这样做就不好实现了

`Flask`是用`Map`和`Rule`这两种数据结构来实现的，`Map`的结构类似字典，但能处理更复杂的情况，我们姑且认为`Map`就是字典

大概像这样

```python
{'/': "11"}
```

`value`不是视图函数名，而是一个字符串，我们叫它`endpoint`，除非手动指定，它一般是函数名的字符串形式

`endpoint`和视图函数是对应的，这个字典存储在`Flask`的`view_functions`属性中，它是一个字典

它的结构类似这样



```python
{'11':hello_world}
```

![image-20201230235156471](https://i.loli.net/2020/12/30/zeNT6rGUDj4Wfxa.png)

以上就是在执行 `@flask_app.route('/')`的时候发生的事情，`Map`和`view_functions`会形成上面的样子

### 匹配

`flask`有一个`url_map`属性，这个属性就是`Map`实例，你可以认为是一个空字典。`route`的作用就是在项目启动的时候往里面添加`Rule`对象，就是`url`规则

当一个请求来的时候，`Flask`会根据`url`在`Map`找到对应的`Rule`,再由`Rule`获取`endpoint`，再根据`endpoint`找到对应的`function`，然后执行 `function()`



### 为什么要`endpoint`？

直接用视图函数名不好吗？

有时候我们需要根据视图函数的名字来获取这个视图函数的`url`

`Flask`内置了`url_for`函数，参数就是`endpoint`的名字，例如  `url_for("11")`返回的就是 `"/"`

如果没有`endpoint`而使用函数名的字符串`hellow_world`，例如`url_for("hello_world")`,万一你的同事把函数名给改了，你的`url_for`就要报错了，而`endpoint`一般不会去改，谁改带他周末去爬山

参考[What is an 'endpoint' in Flask?](https://stackoverflow.com/questions/19261833/what-is-an-endpoint-in-flask)

## 实现

### Rule和Map

`Rule`和`Map`都定义在 `werkzeug/routing.py`中

测试以下代码

```python
from werkzeug.routing import Map, Rule

m = Map([
    Rule('/', endpoint='index'),
    Rule('/downloads/', endpoint='downloads/index'),

])
# 添加一个Rule
m.add(
    Rule('/downloads/<int:id>', endpoint='downloads/show')
)

# 把Map绑定到某个域名，返回MapAdapter对象
urls = m.bind("example.com", "/")

print(urls.match("/downloads/42"))
# 返回('downloads/show', {'id': 42})


print(urls.match("/downloads/42", return_rule=True))
# 返回 (<Rule '/downloads/<id>' -> downloads/show>, {'id': 42})
print(urls.match( return_rule=True))
# 也可以不填原始url字符，直接匹配出当前请求的Rule

```

我们可以知道

1. `Map`中的元素是`Rule`对象，`Rule`对象其实就是`URL`规则和`endpoint`的封装对象
2. `Map`要绑定到某个域名下，实际上也可以绑定到`environ`，毕竟`environ`中有域名信息
3. `Map`的`bind`方法返回 `MapAdapter`对象，`MapAdapter`对象执行实际的匹配工作，它可以根据请求的URL匹配出`（Rule，请求参数）`，也就是`match`方法做的事情

### route方法

`route`方法做的事情就是向`Map`里面`add` `Rule对象`

我们看`route`方法执行这一句 `@flask_app.route('/',endpoint="11")`

```python
    def route(self, rule, **options):
        def decorator(f):
            endpoint = options.pop("endpoint", None)
            # here
            self.add_url_rule(rule, endpoint, f, **options)
            return f

        return decorator
```

再看 `add_url_rule`

```python
    def add_url_rule(
        self,
        rule,
        endpoint=None,
        view_func=None,
        provide_automatic_options=None,
        **options
    ):
        # 1.没有指定endpoint，那么他就是函数名的字符串形式
        if endpoint is None:
            endpoint = _endpoint_from_view_func(view_func)
        options["endpoint"] = endpoint
        # 2.HTTP请求方法，没有指定那就是GET
        methods = options.pop("methods", None)
        if methods is None:
            methods = getattr(view_func, "methods", None) or ("GET",)
        # 3.把字符串形式的URL规则转换成Rule对象，例如"/"转换成Rule("/")
        rule = self.url_rule_class(rule, methods=methods, **options)
        
        # 4.把Rule添加到Map中，注意Flask对象有一个url_map属性，值一开始就是空的Map对象
        self.url_map.add(rule)
        if view_func is not None:
            old_func = self.view_functions.get(endpoint)
            if old_func is not None and old_func != view_func:
                raise AssertionError(
                    "View function mapping is overwriting an "
                    "existing endpoint function: %s" % endpoint
                )
            # 5.把endpoint和视图函数放到view_fuctions这个字典中    
            self.view_functions[endpoint] = view_func
```

就是为了第4、5两步

### 匹配

匹配的过程就是根据请求的`URL`去`match`出`Rule`对象，再根据`Rule`对象找到视图函数

`full_dispatch_request->dispatch_request->dispatch_request`

```python
def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ)
    error = None
    try:
        try:
            ctx.push()
            # 看这里
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



```python
    def full_dispatch_request(self):
        self.try_trigger_before_first_request_functions()
        try:
            request_started.send(self)
            rv = self.preprocess_request()
            if rv is None:
                # 看这里，rv就是视图函数的返回结果
                rv = self.dispatch_request()
        except Exception as e:
            rv = self.handle_user_exception(e)
        return self.finalize_request(rv)
```



重点看这个方法，请求到这里之后

```python
    def dispatch_request(self):
        req = _request_ctx_stack.top.request
        rule = req.url_rule
        return self.view_functions[rule.endpoint](**req.view_args)
```

我们知道`req`是从`LocalStack`中取出的 `RequestContext`的`request`属性，保存着请求的信息

`request`有一个`url_rule`属性，他就是`Rule`对象，我们从`Rule`对象中拿到`endpoint`,再从 `view_functions`中根据`endpoint`拿到视图函数，并传入请求参数，执行之后返回结果就结束了



### 一个疑问

那么`req`是什么时候有的`Rule`属性的呢？？？

是在 `RequestContext`对象的`push`方法中，我们知道请求来了第一步就是`push`

我还是删去了一些无关代码

```python
    def push(self):
        if self.url_adapter is not None:
        	self.match_request()
```

而`match_request`做的事情就是用 `MapAdapter对象match`出当前请求的`Rule`和请求参数 `view_args`，然后绑定到`request`上，这样`request`就有了`url_rule`属性

### 流程图

图随手画的，有什么好的画图工具可以推荐一下

![image-20210107170309038.png](https://i.loli.net/2021/01/07/RC2WYrGvcDAaJhN.png)



`url_adapter`就是`MapAdapter`对象

```python
    def match_request(self):
        try:
            result = self.url_adapter.match(return_rule=True)
            self.request.url_rule, self.request.view_args = result
        except HTTPException as e:
            self.request.routing_exception = e
```

至于`match`方法为什么不需要传入当前请求的URL，那是因为`url_adapter`已经包含了当前请求的信息了



在`RequestContext`的 `__init__`方法中我们可以看到， ` self.url_adapter = app.create_url_adapter(self.request)`

```python
    def __init__(self, app, environ, request=None, session=None):
        self.url_adapter = None
        try:
            self.url_adapter = app.create_url_adapter(self.request)
        except HTTPException as e:
            self.request.routing_exception = e
 

```



再看 `create_url_adapter`方法，会用`Map`对象 `bind_to_environ`



```python
    def create_url_adapter(self, request):

        if request is not None:
            # If subdomain matching is disabled (the default), use the
            # default subdomain in all cases. This should be the default
            # in Werkzeug but it currently does not have that feature.
            subdomain = (
                (self.url_map.default_subdomain or None)
                if not self.subdomain_matching
                else None
            )
            return self.url_map.bind_to_environ(
                request.environ,
                server_name=self.config["SERVER_NAME"],
                subdomain=subdomain,
            )
```

做的事情就是把`Map`表绑定到`environ`上，毕竟你这个`Map`也就是路由表，要属于某个域名

再看 `bind_to_environ`这个方法，没必要都看明白，需要的时候，再断点调试就好了



```python
def bind_to_environ(self, environ, server_name=None, subdomain=None):
  
    def _get_wsgi_string(name):
        val = environ.get(name)
        if val is not None:
            return wsgi_decoding_dance(val, self.charset)

    script_name = _get_wsgi_string("SCRIPT_NAME")
    path_info = _get_wsgi_string("PATH_INFO")
    query_args = _get_wsgi_string("QUERY_STRING")
    return Map.bind(
        self,
        server_name,
        script_name,
        subdomain,
        scheme,
        environ["REQUEST_METHOD"],
        #here
        path_info,
        # here
        query_args=query_args,
    )
```

我们看到，这个方法把从`envirion`中取出来的`path_info`和`query_args`传到了`bind`方法中，然后返回`MapAdapter`对象，接着就可以用`MapAdapter`对象`match`出`Rule`了

也就是说我们从 `Map`构造`MapAdapter`，然后就可以直接用`match`方法匹配出当前请求的`Rule`，再根据`Rule`获取`endpoint`，再由此获取视图函数然后调用就好了

## 参考文章

[flask 源码解析：路由](https://cizixs.com/2017/01/12/flask-insight-routing/)