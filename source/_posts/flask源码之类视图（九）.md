---
title: flask源码之类视图（九）
date: 2021-01-23 02:01:40
tags: [flask源码]
categories: [flask]
---



### 引言

```python
from flask import Flask, request, current_app, jsonify
import uuid

from flask.views import MethodView

flask_app = Flask(__name__)


class HelloView(MethodView):
    def get(self):
        return jsonify(
            code=-1,
            msg="success",
            data=[]
        )


flask_app.add_url_rule("/", HelloView.as_view("hello"))

if __name__ == '__main__':
    flask_app.run()

```

代码很简单，类视图的好处在于，你可以通过写基类，来处理一些公共的操作，你甚至还可以复用一些已经写好的类视图，让你少些很多代码，更多信息请参考`django restframework`

<!--more-->

### 实现

源码在 `flask.views.py`

```python
class MethodView(with_metaclass(MethodViewType, View)):

    def dispatch_request(self, *args, **kwargs):
        meth = getattr(self, request.method.lower(), None)

        # If the request method is HEAD and we don't have a handler for it
        # retry with GET.
        if meth is None and request.method == "HEAD":
            meth = getattr(self, "get", None)

        assert meth is not None, "Unimplemented method %r" % request.method
        return meth(*args, **kwargs)
```

这是一个用到元类来定义自己的类



```python
flask_app.add_url_rule("/", HelloView.as_view("hello"))
```

我们知道，这种方式注册路由，最后`flask`大概会这样执行

```python
obj=HelloView.as_view("hello")
rv=obj()
```

所以我们看看 `as_view`返回了什么

`as_view`定义在父类 `View`中

```python
class View(object):

    #: A list of methods this view can handle.
    methods = None

    #: Setting this disables or force-enables the automatic options handling.
    provide_automatic_options = None

    decorators = ()

    @classmethod
    def as_view(cls, name, *class_args, **class_kwargs):
        """Converts the class into an actual view function that can be used
        with the routing system.  Internally this generates a function on the
        fly which will instantiate the :class:`View` on each request and call
        the :meth:`dispatch_request` method on it.

        The arguments passed to :meth:`as_view` are forwarded to the
        constructor of the class.
        """

        def view(*args, **kwargs):
            self = view.view_class(*class_args, **class_kwargs)
            return self.dispatch_request(*args, **kwargs)

        if cls.decorators:
            view.__name__ = name
            view.__module__ = cls.__module__
            for decorator in cls.decorators:
                view = decorator(view)

        # We attach the view class to the view function for two reasons:
        # first of all it allows us to easily figure out what class-based
        # view this thing came from, secondly it's also used for instantiating
        # the view class so you can actually replace it with something else
        # for testing purposes and debugging.
        view.view_class = cls
        view.__name__ = name
        view.__doc__ = cls.__doc__
        view.__module__ = cls.__module__
        view.methods = cls.methods
        view.provide_automatic_options = cls.provide_automatic_options
        return view
    def dispatch_request(self):
        """Subclasses have to override this method to implement the
        actual view function code.  This method is called with all
        the arguments from the URL rule.
        """
        raise NotImplementedError()
```

原理很简单，就是`as_view`内部定义了一个闭包，然后返回这个闭包函数，所以最后还是相当于执行了视图函数

值得注意的时，这里这里的`view`函数最后调用 `dispatch_request`方法，而这个方法在`View`中没有实现

所以`View`类不能单独使用，或者要定义 `dispatch_request`才能使用

而 `MethodView`定义了 `dispatch_request`方法



```python
class MethodView(with_metaclass(MethodViewType, View)):

    def dispatch_request(self, *args, **kwargs):
        meth = getattr(self, request.method.lower(), None)

        # If the request method is HEAD and we don't have a handler for it
        # retry with GET.
        if meth is None and request.method == "HEAD":
            meth = getattr(self, "get", None)

        assert meth is not None, "Unimplemented method %r" % request.method
        return meth(*args, **kwargs)
```

他的作用很简单，就是拿到`HTTP`请求的方法之后，转换为小写，调用同名的类方法

例如 `HTTP GET->"get"->get方法` 

### 元类

在`python`中类也是对象，类是由元类创建的

```python
class MethodView(with_metaclass(MethodViewType, View)):
```

而 `MethodView`是由元类 `MethodViewType`创建的

也就是说它不是你看到的样子，他还有一些属性或者发放是被`MethodViewType`在创建过程中动态添加的

所以如何给类动态添加属性，就是元类

```python
class MethodViewType(type):
    """Metaclass for :class:`MethodView` that determines what methods the view
    defines.
    """

    def __init__(cls, name, bases, d):
        super(MethodViewType, cls).__init__(name, bases, d)

        if "methods" not in d:
            methods = set()

            for base in bases:
                if getattr(base, "methods", None):
                    methods.update(base.methods)

            for key in http_method_funcs:
                if hasattr(cls, key):
                    methods.add(key.upper())
            if methods:
                cls.methods = methods
```

元类的一个好处是，你可以在方法中拿到你正在创建的类的名字（name），父类（bases），属性（d）

例如我们用 `MethodViewType`去创建 `MethodView`,那这个`name`就是他的`name`

去创建 `HelloView`，那么`name`就是 这个类的名字

同时注意`d`是当前被创建类的属性的集合，不包括父类

这个元类的作用很简单，就是给你要创建类添加 `methods`属性，而这个`methods`属性就相当于你之前在路由中定义的`methods`

```python
@flask_app.route("/",methods=["GET"])
```

也就是说，在类视图中，我们还可以不在路由中定义所接受的方法

至于为什么非要用元类，其实也不是必须要用，或许是作者觉得方便

