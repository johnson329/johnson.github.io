---
title: flask源码之钩子函数（十）
date: 2021-01-23 02:27:46
tags: [flask源码]
categories: [flask]
---

## 引言

先看这样一个需求，`flask`在处理一个请求的时候，可能会在多处打日志，反映在日志文件中就是不同行，那不同行的日志怎么判断是一个请求所为呢？

我们可以给 `request`对象绑定一个 `request_id`属性，在打日志的时候带上这个属性

```python
from flask import Flask, request, current_app
import uuid

flask_app = Flask(__name__)


@flask_app.before_request
def gen_request_id():
    request_id = uuid.uuid1()
    request.request_id = request_id


@flask_app.route('/')
def hello_world():
    current_app.logger.warn("{}".format(request.request_id))
    return "hello flask"


if __name__ == '__main__':
    flask_app.run()

```

<!--more-->

这里我们用到了`flask`的钩子函数，`before_request`

它类似与`django`的中间件或者`java`的`AOP`,主要作用是把每个请求公共的一些操作抽取出来统一处理

![这里写图片描述](https://i.loli.net/2021/01/23/dVkIrvGXFJ9P57Y.png)

重点看一下 `teardown_request`，它是注册一个函数在每个请求的末尾运行，不管是否有异常, 每次请求的最后都会执行

而 `after_request`是在视图函数没有异常情况下才会被调用的钩子函数

## 原理

### 注册

```python
@flask_app.before_request
def gen_request_id():
    request_id = uuid.uuid1()
    request.request_id = request_id
```

就是把 `gen_request_id`存入这个字典中

```python
{None:[gen_request_id]}
```

这个字典是`Flask`类的属性值，属性是 `before_request_funcs`

接着在请求运行中，遍历这个字典

### 实现

`before_request`源码

```python
 class Flask(object):
    @setupmethod
    def before_request(self, f):
        """Registers a function to run before each request.

        For example, this can be used to open a database connection, or to load
        the logged in user from the session.

        The function will be called without any arguments. If it returns a
        non-None value, the value is handled as if it was the return value from
        the view, and further request handling is stopped.
        """
        self.before_request_funcs.setdefault(None, []).append(f)
```

再看 `full_dispatch_request`

![image-20210123022046189](https://i.loli.net/2021/01/23/rhsNg368AuJ9BVD.png)

这个 `preprocess_request`就是处理 `@flask_app.before_request`注册的函数的

`preprocess_request`

```python
    def preprocess_request(self):
        bp = _request_ctx_stack.top.request.blueprint
        # 从字典中获取所有被@flask_app.before_request注册的函数
        # 键是None的表示应用注册的，此外还有蓝图的
        funcs = self.before_request_funcs.get(None, ())
        if bp is not None and bp in self.before_request_funcs:
            funcs = chain(funcs, self.before_request_funcs[bp])
        # 获取所有函数遍历执行，谁return了，就直接返回到客户端，所以这里要注意
        for func in funcs:
            rv = func()
            if rv is not None:
                return rv
```

其他钩子函数都是类似思路

值得注意的是`teardown_request`注册的钩子函数是在 `RequestContext`被`pop`的时候执行的，从而实现无论视图函数是否有异常，这里的钩子函数都会被执行