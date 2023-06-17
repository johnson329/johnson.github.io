---
title: 'Flask源码之异常处理-兼论前后端分离场景下的接口格式问题（五）'
date: 2021-01-05 23:35:15
tags: [flask源码]
categories: [flask]
---

## 引言

### 问题提出

先看下面一段代码

```python
from flask import Flask, jsonify

flask_app = Flask(__name__)


@flask_app.route('/', endpoint="11")
def hello_world():
    a = 1 / 0
    return jsonify(
        code=0,
        msg="success",
        data=["hello,world!"]
    )

if __name__ == '__main__':
    flask_app.run()
```

这段代码有一个很显然的未捕捉的异常，就是`1/0`

<!--more-->

再看`flask`在生产环境中返回了什么，一个状态是`500`、内容是`html`的返回

![image-20210105234225278](https://i.loli.net/2021/01/05/QaeZiYGX5CDPwO7.png)

然而在用`json`格式交互的前后端分离场景下，前端希望后端仍然返回`json`格式的数据，而不是`html`

再考虑到可能以后还有安卓和`ios`，人家可能都不需要`html`

所以我们需要在后端服务出现未捕获的异常时候，返回自定义的`json`格式数据

修改代码如下

```python
from flask import Flask, jsonify

flask_app = Flask(__name__)


@flask_app.route('/', endpoint="11")
def hello_world():
    a = 1 / 0
    return jsonify(
        code=0,
        msg="success",
        data=["hello,world!"]
    )


@flask_app.errorhandler(500)
def handle_500(e):
    # 可能还要记录一下自定义的日志
    # 也可能还需要回滚一下数据库操作
    return jsonify(
        code=-1,
        msg="unknown error"
    )


if __name__ == '__main__':
    flask_app.run()
```

重启服务

![image-20210105235636806](https://i.loli.net/2021/01/05/v4glcPw9Uh6WkEf.png)

这样，我们的前端、ios和安卓只需要再一个全局的位置，判断一下，如果`code`等于 `-1`，就展示`unknown error`或者自己换个名字 `服务器未知异常`就好了

这里我们用到了  `flask`提供的 `errorhandler`

### 为什么建议用errorhandler

有的人可能会在没有看文档的情况下，写一个装饰器去装饰视图函数，来捕捉未知异常，像这样

```python
from flask import Flask, jsonify

flask_app = Flask(__name__)


def handle_500(func):
    def wrapper(*args, **kw):
        try:
            rv = func(*args, **kw)
        except Exception as e:
            # 可能还要记录一下自定义的日志
            # 可能还需要回滚数据库操作
            rv = {
                "code": -1,
                "msg": "unknown error"
            }
        return rv

    return wrapper


@flask_app.route('/', endpoint="11")
@handle_500
def hello_world():
    a = 1 / 0
    return jsonify(
        code=0,
        msg="success",
        data=["hello,world!"]
    )


# @flask_app.errorhandler(500)
# def handle_500(e):
#     # 可能还要记录一下自定义的日志
#     return jsonify(
#         code=-1,
#         msg="unknown error"
#     )


if __name__ == '__main__':
    flask_app.run()
```



这样写我个人不太建议:

1. 官方文档建议用`errorhandler`的处理方式，最好用这种
2. `handle_500`这个装饰器要在`@app.route`这个装饰器下面才有作用
3. 如果你用的是函数视图而不是类视图，那么你每个函数都要加这样一个装饰器，产生重复代码，还可能会遗忘。如果你用类视图，可以写一个被装饰的基类
4. 这个装饰器只能捕捉函数视图或者类视图中的异常，我们在开发中还有 `@app.before_request`等钩子函数，里面也会出现异常，这个装饰器无法捕捉，但 `errorhandler`可以
5. 这样更加解耦

`2345`的问题`errorhandler`都可以解决

## 原理

下面是程序执行的方法调用栈，基本原理就是异常在源码中已经被`catch`了然后检查`Flask`对象有没有对应的`handler`可以处理，没有就上抛，一直抛

![image-20210107155113928](https://i.loli.net/2021/01/07/Rr15AxCtGHVLEwm.png)

## 源码实现

这里用断点调试就很简单了

先在 `full_dispatch_request` 打断点，`ctrl+鼠标左键`进入方法内部

![image-20210106003550310](https://i.loli.net/2021/01/06/DJYlMceuOkCyKb2.png)



在 `full_dispatch_request`方法内部打断点，注意要打两个断点，然后按`f9`让程序走到这个断点

![image-20210106004015653](https://i.loli.net/2021/01/06/3k1NQSXU4wbi8hn.png)



进入`dispatch_request`内部，在`self.view_functions[rule.endpoint](**req.view_args)`处打断点



![image-20210106003750673](https://i.loli.net/2021/01/06/JDmrTBp8e4AFKkG.png)

 `self.view_functions[rule.endpoint](**req.view_args)`，这一步其实就是在执行 `hello_world()`

我们知道 `hello_world`这个函数会抛出一个 `零不能被除`的异常

那这个异常会被捕捉吗？

继续往前调试，我们会回到上一层（因为我们在之前`full_dispatch_request`内部打了两个断点）



![image-20210106004158333](https://i.loli.net/2021/01/06/peXLSnAg9jy2N8W.png)

`零不能被除`异常在上一层被捕捉到了，这个`e`就是我们的异常

我们在这个地方停留一会儿，我们发现这个`try`内部除了 `dispatch_request` ，还有 `rv = self.preprocess_request()`这一句，看名字也知道是预处理，什么预处理呢？没错就是之前提到的 `@app.before_request`装饰的钩子函数，由此可见，钩子函数的异常也会被捕捉到

继续看`except`之后的代码

那么 `rv = self.handle_user_exception(e)`会帮我们处理这个 `division by zero`异常吗？

其实也不会，他会把异常继续往上抛，我们稍后再讲他的作用

点击调用栈的这个地方，我们要回到上一层，继续打上一个断点，注意看图中 `Frames`的位置，选择箭头指向的上一层

![image-20210106004705665](https://i.loli.net/2021/01/06/VcnvN93Y7JzIkgB.png)

在`error=e`处打断点

![image-20210106005121465](https://i.loli.net/2021/01/06/gTrJm8UVbRoWuNS.png)

估计你也知道了，`零不能被除`异常被 `handle_user_exception(e)`又抛了出来，在上一层的 `wsgi_app`方法中被捕捉到了，并且交给 `handle_exception`来处理



而 `handle_exception`做的事情也简单



![image-20210106005403417](https://i.loli.net/2021/01/06/tLGvPYHq6ilD3mM.png)



先打日志，把**出错类型 出错值 出错调用栈**全打出来

然后再准备报 `InternalServerError`也就是状态码是`500`的`flask`服务器异常

但是在正式返回之前，会先看一下你有没有 处理`500`错误的`handler`,而我们是有的，于是调用你的`handle_500`函数处理服务器异常

注意这句 `if self.propagate_exception`，`self.propagate_exception`这个属性是为了决定是否传播你的异常，在开发和测试环境下，这个为真，异常会被继续抛到上层，所以我们写的`handler`会不生效，因此，为了看到我们的`500`错误处理器生效，记得在生产环境查看



![image-20210106005724883](https://i.loli.net/2021/01/06/MStYG4J2rfZn9eI.png)

选中 `handler`右键在弹出菜单中选择`evaluate`,可以发现这个`handler`就是我们定义的`handle_500`函数，`handle_500`返回的就是我们自定义的`json`格式







### handle_user_exception处理什么异常

![image-20210106201113736](https://i.loli.net/2021/01/06/9m5kCjGcODFEyYt.png)

比较容易看出，这个方法主要是处理 `HTTPException`和用户自定义的非`500`的异常



看个例子

```python
from flask import Flask, jsonify

flask_app = Flask(__name__)


class ValidationError(ValueError):
    pass


@flask_app.route('/', endpoint="11", methods=["GET", "POST"])
def hello_world():
    raise ValidationError("参数验证失败")


@flask_app.errorhandler(ValidationError)
def validation_error(e):
    return jsonify(
        code=1,
        msg=e.args
    )


if __name__ == '__main__':
    flask_app.run()
```

例如你的代码里面有多处需要抛出和捕获 `ValidationError`这个错误，进一步说，在orm层会抛出这个异常，`view`层捕捉，那么写多次`try`不妨考虑用这种全局异常注册的方式

现在我们看看源码怎么处理的



`handle_user_exception`内部打上断点



![image-20210106224104610](https://i.loli.net/2021/01/06/VdOlaRNkW7cPnqD.png)





![image-20210106224222700](https://i.loli.net/2021/01/06/TEHN6APfX1LxrBS.png)

`handler`就是我们的 `validation_error`函数，最后返回的实际上是 `validation_error`函数的执行结果

## 后端api格式

我们都知道`rest`是一种风格的`api`

他用`http` 码来表示状态，但实际中有时候是不够用的，或者前端、`ios`和安卓都希望不用`http`的什么`400`状态码，

而需要在`json`数据中再定义自己的状态码，`http`状态码统一`200`

像这样

```python
{
    "code":0,
    "msg":"success",
    "data":[]
}
```

这种常见于国内比较大型的项目中

到底是采用`rest`风格的`api`还是自定义状态码，争论很多，具体还是看公司要求

