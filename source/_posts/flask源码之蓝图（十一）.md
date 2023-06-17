---
title: flask源码之蓝图（十一）
date: 2021-01-23 20:17:52
tags: [flask源码]
categories: [flask]
---



## 引言

蓝图（Blueprint）是`flask`内置模块化工具。`flask`一开始没有考虑大型项目，后来在源码中增加了蓝图来组织大型项目结构。

都说强扭的瓜不甜，可`flask`偏要看一下有多苦，苦不苦只有用的程序员知道。

当一个`flask`单体项目逐渐变大，如果还把所有代码写在单文件，项目维护起来就会出现各种问题。例如代码合并冲突。此时拆分代码或者说模块化就会成为我们自然的选择

<!--more-->

## 使用

### 定义蓝图

`auth.py`

```python
from flask import Blueprint, jsonify

auth_bp = Blueprint("auth", __name__)


@auth_bp.route("/login")
def login():
    return jsonify(
        code=-1,
        msg="success",
        data=[]
    )
```

定义一个蓝图很简单，只需要在模块中创建蓝图对象，然后把之前的`flask`对象换成蓝图对象

### 注册蓝图

`flask_app.py`

```python
from flask import Flask
from auth import auth_bp

flask_app = Flask(__name__)

flask_app.register_blueprint(auth_bp)
if __name__ == '__main__':
    flask_app.run()
```

把蓝图对象注册到`flask`对象

我们看到，用了蓝图之后，蓝图对象仿佛成为`flask`对象分身一样，帮助`flask`完成他的工作，包括

1. 路由
2. 异常注册
3. 钩子函数
4. 模板和静态文件路径

所以蓝图可以看成是子`flask`

## 原理

原理很简单，以蓝图路由为例

蓝图的路由被添加到了`Blueprint`对象中，在最后执行 `flask_app.register_blueprint(auth_bp)`的时候，路由的字典最终才被添加到`Flask`对象上了

## 实现

### 一、`bluprints.route`

```python
 def route(self, rule, **options):
        """Like :meth:`Flask.route` but for a blueprint.  The endpoint for the
        :func:`url_for` function is prefixed with the name of the blueprint.
        """

        def decorator(f):
            endpoint = options.pop("endpoint", f.__name__)
            self.add_url_rule(rule, endpoint, f, **options)
            return f

        return decorator
```

一如`flask`，最终执行了 `add_url_rule`

### 二、`add_url_rule`

```python
 def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
        """Like :meth:`Flask.add_url_rule` but for a blueprint.  The endpoint for
        the :func:`url_for` function is prefixed with the name of the blueprint.
        """
        if endpoint:
            assert "." not in endpoint, "Blueprint endpoints should not contain dots"
        if view_func and hasattr(view_func, "__name__"):
            assert (
                "." not in view_func.__name__
            ), "Blueprint view function name should not contain dots"
        self.record(lambda s: s.add_url_rule(rule, endpoint, view_func, **options))
```

就做了一件事，执行`record`方法，传入了一个匿名函数，这个匿名函数就是执行 `add_url_rule`

```python
lambda s: s.add_url_rule(rule, endpoint, view_func, **options)
```

我们再看`record`方法

### 三、`record`方法	

```python
    def record(self, func):
        if self._got_registered_once and self.warn_on_modifications:
            from warnings import warn

            warn(
                Warning(
                    "The blueprint was already registered once "
                    "but is getting modified now.  These changes "
                    "will not show up."
                )
            )
        self.deferred_functions.append(func)
```

`self.deferred_functions`是一个`list`

所以，简而言之，`blueprints.route`方法做的事情就是向`deferred_functions`这个`list`中添加一个匿名函数

也就是说，路由注册到了蓝图对象了

那蓝图对象注册到`flask`对象发生了什么呢？

### 四、`register_blueprint`方法

`register_blueprint`是`flask`对象的方法

例如 `flask_app.register_blueprint(auth_bp)`

```python
@setupmethod
def register_blueprint(self, blueprint, **options):

    first_registration = False

    if blueprint.name in self.blueprints:
        assert self.blueprints[blueprint.name] is blueprint, (
            "A name collision occurred between blueprints %r and %r. Both"
            ' share the same name "%s". Blueprints that are created on the'
            " fly need unique names."
            % (blueprint, self.blueprints[blueprint.name], blueprint.name)
        )
        else:
            self.blueprints[blueprint.name] = blueprint
            self._blueprint_order.append(blueprint)
            first_registration = True

            blueprint.register(self, options, first_registration)
```

主要看最后一句，其实就是调用传入蓝图的`register`方法

### 五、`blueprints.register`方法

去掉一些无关代码，大概就是这样

```python
    def register(self, app, options, first_registration=False):
        self._got_registered_once = True
        state = self.make_setup_state(app, options, first_registration)
        for deferred in self.deferred_functions:
            deferred(state)

```



`state`是`BlueprintSetupState`对象，他有一个`add_url_rule`方法

```python
class BlueprintSetupState(object):

    def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
 
        if self.url_prefix is not None:
            if rule:
                rule = "/".join((self.url_prefix.rstrip("/"), rule.lstrip("/")))
            else:
                rule = self.url_prefix
        options.setdefault("subdomain", self.subdomain)
        if endpoint is None:
            endpoint = _endpoint_from_view_func(view_func)
        defaults = self.url_defaults
        if "defaults" in options:
            defaults = dict(defaults, **options.pop("defaults"))
        self.app.add_url_rule(
            rule,
            "%s.%s" % (self.blueprint.name, endpoint),
            view_func,
            defaults=defaults,
            **options
        )

```

注意到 `deferred_functions`是匿名函数 

```python
lambda s: s.add_url_rule(rule, endpoint, view_func, **options)
```



所以

```python
for deferred in self.deferred_functions:
	deferred(state)
```

其实就是相当于匿名函数的`s`变成了 `BlueprintSetupState`对象，也就是遍历 `deferred_functions`并借用`BlueprintSetupState`完成 `add_url_rule`

注意到此时的`add_url_rule`结尾调用的是

```python
self.app.add_url_rule(
            rule,
            "%s.%s" % (self.blueprint.name, endpoint),
            view_func,
            defaults=defaults,
            **options
        )
```

也就是调用`flask`对象的 `add_url_rule`方法



综上，当执行 `register_blueprint`的时候，`flask`才会把蓝图对象的路由注册到自身

蓝图执行`route`的过程只不过把路由临时存在自身的一个`list`中而已，所以叫 `deferred_functions`，翻译过来就是延迟函数

### 六、其他

同理，当我们用蓝图注册错误

```python
def register_error_handler(self, code_or_exception, f):
    self.record_once(
        lambda s: s.app._register_error_handler(self.name, code_or_exception, f)
    )
```

其实也是把被注册的函数放在一个`list`中，延迟注册到`flask`对象