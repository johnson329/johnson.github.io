---
title: flask源码之配置加载(八)
date: 2021-01-15 23:30:54
tags: [flask源码]
categories: flask
keywords: flask源码，配置
---

### 引入

```python
from flask import Flask, jsonify

flask_app = Flask(__name__)

flask_app.config.from_mapping(
    {
        "SECRET_KEY": "you never know the secret key"
    }
)


@flask_app.route('/', endpoint="11", methods=["GET", "POST"])
def hello_world():
    return jsonify(code=0, msg="success")


if __name__ == '__main__':
    flask_app.run()
```

<!--more-->

### 原理

1. `Flask`实例的`config`属性是`Config`对象
2. `Config`类实现了`from_mapping`等方法



### 实现

`config`属性

![image-20210115233513142](https://i.loli.net/2021/01/15/HfDy5klO8NtW93i.png)



看来`Config`对象是`make_config`方法返回的

```python
   def make_config(self, instance_relative=False):
        """Used to create the config attribute by the Flask constructor.
        The `instance_relative` parameter is passed in from the constructor
        of Flask (there named `instance_relative_config`) and indicates if
        the config should be relative to the instance path or the root path
        of the application.

        .. versionadded:: 0.8
        """
        root_path = self.root_path
        if instance_relative:
            root_path = self.instance_path
        defaults = dict(self.default_config)
        defaults["ENV"] = get_env()
        defaults["DEBUG"] = get_debug_flag()
        return self.config_class(root_path, defaults)
```

我们看到了`config`类，就是`self.config_class`

这里暂停一下看 `instance_relative`，这个为`true`的话，会把 `instance_path`也就是你的`Flask`对象的路径，传给`config_class`类，毕竟如果你要从配置文件加载配置，我得知道文件路径在哪里，`flask`允许你根据`Flask`实例的相对路径来定位

默认是`false`，那就是应用的根目录绝对路径

### `Config`类的定义

![image-20210115234401391](https://i.loli.net/2021/01/15/Txj71S6JfOGrM82.png)

原来是继承自`dict`

再看`from_mapping`方法，我去掉了一些

```python
    def from_mapping(self, *mapping, **kwargs):
        mappings = []
        mappings.append(kwargs.items())
        for mapping in mappings:
            for (key, value) in mapping:
                if key.isupper():
                    self[key] = value
        return True
```

大概逻辑就是把传入的字典放进`list`种

然后遍历每个字典，把字典的`key`作为`Config`的属性名，`value`作为属性值

### 如何读取

```python
from flask import Flask, jsonify, current_app

flask_app = Flask(__name__)

flask_app.config.from_mapping(
    {
        "SECRET_KEY": "you never know the secret key"
    }
)


@flask_app.route('/', endpoint="11", methods=["GET", "POST"])
def hello_world():
    # 使用
    sk = current_app.config["SECRET_KEY"]
    return jsonify(code=0, msg="success",data={"sk":sk})


if __name__ == '__main__':
    flask_app.run()

```

记住，`current_app`是代理对象，代理的是`LocalStack`种的`Flask`实例，这个栈一般在请求来的时候才会被`push`

所以你无法在视图函数外面写

```python
current_app.config["SECRET_KEY"]
```

因为此时从栈中 `top`出来的是`None`,`None`怎么又`config`属性呢

### 自定义配置方法



例如你的写`java`的`leader`要你从`yaml`中读取配置，你能怎么办呢？

首先 `fuck`一声

然后这么写，其实只需要给`Config`一个`from_yaml`方法就行了，但是我们不能改这个类，怎么办呢？

动态添加呗

```python
# 定义好读取方法
def from_yaml(self, filename, silent=False):
    filename = os.path.join(self.root_path, filename)
    try:
        with open(filename) as yaml_file:
            obj = yaml.load(yaml_file.read(), Loader=yaml.FullLoader)
    except IOError as e:
        if silent:
            return False
    # obj是个字典，那就不要大费周章了
    return self.from_mapping(obj)


# 然后为 config 对象动态添加 from_yaml 方法
app.config.from_yaml = types.MethodType(from_yaml, app.config)
app.config.from_yaml("config.yaml")
```

然后你把`config.yaml`文件放到`flask`实例下面就好了

如果你的老板突然又怀念`java`的 `properties`

那你怎么写呢？

```python
def from_properties(self, filename, silent=False, encode=None):
    filename = os.path.join(self.root_path, filename)
    try:
        with open(filename) as properties_file:
            obj = {}
            for line in properties_file:
                if line.find('=') > 0:
                    s = line.replace('\n', '').split("=")
                    obj[s[0]] = s[1]
    except IOError as e:
        if silent:
            return False
    return self.from_mapping(obj)
# 为 config 对象动态添加 from_properties 方法
app.config.from_properties = types.MethodType(from_properties, app.config)
app.config.from_properties("config.properties")
app.run()
```

这个工作起码要写两天，谁叫这个`leader`这么事儿多



我们看到在`python`中给实例动态添加方法和动态添加属性有点不同

动态添加属性简单

```python
app.config.new_property="asdfa"
或者
setattr(app.config,"new_property","asdf")
这种根据字符串找到对象属性的做法又叫反射
```

但是给对象动态添加方法就不同了，要用到 `types.MethodType`

```python
app.config.from_properties = types.MethodType(from_properties, app.config)
```

我们甚至还可以这样给类动态添加方法，只要把你的`app.config`换成类就行了

当然，你也可以动态删除一个对象的属性

```python
del 对象.属性名
delattr(对象, “属性名”)
```

