---
title: Python 装饰器到底在“装”什么？
date: 2026-04-9 23:30:00
categories:
  - 技术
tags:
  - Python
---

假设你要给项目里的多个函数加上执行时间记录，最直接的做法是在每个函数里手动写：

```
import time

def query_user():
    start = time.time()
    # 业务逻辑
    time.sleep(0.5)
    print(f"耗时: {time.time() - start:.2f}s")
```

这样写有两个问题：逻辑重复（每个函数都要粘贴一遍），而且计时代码混进了业务代码，将来要改统计方式，就得逐个找、逐个改。

装饰器解决的正是这件事：**在不修改原函数的前提下，给它增加额外行为。**

## 一、装饰器的本质

理解装饰器，先接受一个前提：**Python 的函数是对象**，可以赋值给变量，也可以作为参数传给另一个函数。

```
def greet():
    print("hello")

say = greet   # 函数赋值
say()         # 输出: hello
```

**装饰器的本质，就是一个接收函数、返回新函数的函数。** 最简单的结构是这样：

```
def my_decorator(func):
    def wrapper():
        print("执行前")
        func()
        print("执行后")
    return wrapper

def greet():
    print("hello")

greet = my_decorator(greet)  # 用新函数替换旧函数
greet()

'''输出：
执行前
hello
执行后
'''
```

两种写法完全等价，对比来看：

```

@my_decorator
def greet():
    print("hello")
```

**`@my_decorator`** **只是** **`greet = my_decorator(greet)`** **的** **语法糖** **，两者完全等价。**

## 二、标准写法与执行顺序

```
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("调用前")
        result = func(*args, **kwargs)
        print("调用后")
        return result
    return wrapper

@my_decorator
def add(a, b):
    return a + b

print(add(1, 2))
```

有几点要说清楚：

-   **执行顺序**：`@my_decorator` 在模块加载时就执行了，此时 `add` 被替换为 `wrapper`。之后每次调用 `add(1, 2)`，实际运行的是 `wrapper(1, 2)`。
-   **`*args, **kwargs`**：为了让装饰器对任意参数的函数都能生效，wrapper 要透传所有参数，同时把返回值也带回来，否则原函数的返回值会丢失。这个知识点可以看[Day4 Python的函数和参数机制](https://juejin.cn/post/7621871037852745737)。

## 三、带参数的装饰器

有时候装饰器本身也需要配置，比如 `@retry(times=3)`。这时需要三层嵌套：

```
def retry(times=3):          # 第一层：接收装饰器参数
    def decorator(func):     # 第二层：接收被装饰的函数
        def wrapper(*args, **kwargs):  # 第三层：实际执行逻辑
            for i in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    print(f"第 {i+1} 次失败: {e}")
            raise RuntimeError("重试次数耗尽")
        return wrapper
    return decorator

@retry(times=3)
def unstable_request():
    import random
    if random.random() < 0.7:
        raise ConnectionError("网络抖动")
    return "成功"
```

三层的逻辑分工是这样的：外层拿到配置参数，中层拿到目标函数，内层才是真正运行的代码。

## 四、 `functools.wraps`

装饰器替换了原函数，但有个副作用：函数的元信息也被替换了。

```
def my_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def add(a, b):
    """两数相加"""
    return a + b

print(add.__name__)  # 输出: wrapper，不是 add
print(add.__doc__)   # 输出: None
```

这会导致文档生成错误、调试信息混乱。**修复方式是加上** **`functools.wraps`**：

```
from functools import wraps

def my_decorator(func):
    @wraps(func)          # 把原函数的元信息复制过来
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

加上之后，`add.name` 就是 `add`，`add.doc` 也正常了。**建议养成习惯，写装饰器时始终加上** **`@wraps(func)`** **。**

## 五、应用场景

假设项目里有多个对外接口函数，需要统一记录调用日志（入参、耗时、是否异常）。

**不用装饰器的写法：**

```
def get_user(user_id):
    start = time.time()
    print(f"[LOG] 调用 get_user, 参数: {user_id}")
    try:
        result = db.query(user_id)
        print(f"[LOG] 成功, 耗时 {time.time()-start:.2f}s")
        return result
    except Exception as e:
        print(f"[LOG] 异常: {e}")
        raise

def get_order(order_id):
    start = time.time()
    print(f"[LOG] 调用 get_order, 参数: {order_id}")
    # 同样的模板代码...
```

每个函数都要写一遍，日志格式一旦要改，就是全局搜索替换。

**用装饰器的写法：**

```
import time
from functools import wraps

def log_call(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"[LOG] 调用 {func.__name__}, 参数: {args} {kwargs}")
        start = time.time()
        try:
            result = func(*args, **kwargs)
            print(f"[LOG] {func.__name__} 完成, 耗时 {time.time()-start:.2f}s")
            return result
        except Exception as e:
            print(f"[LOG] {func.__name__} 异常: {e}")
            raise
    return wrapper

@log_call
def get_user(user_id):
    # 只写业务逻辑
    return {"id": user_id, "name": "Alice"}

@log_call
def get_order(order_id):
    return {"order_id": order_id, "status": "shipped"}
```

日志逻辑集中在一处，业务函数只做自己的事。要改日志格式，只改 `log_call` 一个地方。

* * *

## 六、总结

装饰器的核心可以用一句话概括：**把与业务无关但需要到处生效的逻辑从函数体里抽出来，统一管理。**

适合用装饰器的典型场景：

-   日志记录、性能统计
-   权限校验、身份认证
-   重试机制、异常兜底
-   缓存（如 `@lru_cache`）
-   参数校验、类型检查

这类逻辑有个共同特点：它们本身不是业务的一部分，却需要附着在业务函数上运行。装饰器提供了一个干净的方式来处理这件事，而不是把它们混进每一个函数里。