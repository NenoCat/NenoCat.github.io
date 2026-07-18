---
title: Python文件读写 with的本质
date: 2026-07-18 23:30:01
categories:
  - 技术
tags:
  - Python
---

文件读写是日常开发中非常常见的一类操作，无论是处理日志、配置文件，还是接口数据落地，都离不开 I/O。

* * *

## 基础：open / read / write

Python 中通过 `open()` 打开文件，返回一个文件对象：

```
# 打开文件（只读模式）
f = open("test.txt", "r", encoding="utf-8")

content = f.read()
print(content)

f.close()
```

常见模式包括：

-   `"r"`：只读（默认）
-   `"w"`：写入（会覆盖原内容）
-   `"a"`：追加写入
-   `"rb"` / `"wb"` ：二进制模式

写文件示例：

```
f = open("output.txt", "w", encoding="utf-8")
f.write("Hello, Python I/O")
f.close()
```

这里需要注意两点：

1.  忘记 `close()` 可能导致资源未释放；
1.  `"w"` 模式会清空文件内容。

* * *

## 文本文件处理

实际开发中，更常见的是逐行处理文本：

```
f = open("test.txt", "r", encoding="utf-8")

for line in f:
    print(line.strip())

f.close()
```

相比 `read()` 一次性读取全部内容，这种方式更适合内容较多的文件。

* * *

## with 语句（上下文管理）

`with` 语句可以自动管理文件资源，无需手动关闭：

```
with open("test.txt", "r", encoding="utf-8") as f:
    content = f.read()
    print(content)
```

等价于：

-   打开文件
-   使用文件
-   自动调用 `close()`

`with` 是处理文件的标准写法，块结束后自动关闭文件，哪怕中间抛了异常。在实际开发中，它可以避免资源泄露问题。

* * *

## 处理大文件（逐行读取）

如果文件有几百 MB，一次性 `read()` 进内存会很危险。推荐使用逐行读取方式：

```
with open("big_file.txt", "r", encoding="utf-8") as f:
    for line in f:
        process(line)
```

或者更细粒度控制：

```
with open("big_file.txt", "r", encoding="utf-8") as f:
    while True:
        line = f.readline()
        if not line:
            break
        process(line)
```

这种方式只会在内存中保留当前行，适合日志分析、数据清洗等场景。

* * *

## JSON 文件读写（高频）

接口返回值、配置文件、爬虫结果，很多场景都是 JSON 格式。Python 内置的 `json` 模块直接搞定：

### 读取 JSON：

```
import json

with open("data.json", "r", encoding="utf-8") as f:
    data = json.load(f)

print(data)
```

### 写入 JSON：

```
import json

data = {"name": "Alice", "age": 18}

with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
```

参数说明：

-   `ensure_ascii=False`：避免中文被转义
-   `indent=2`：美化输出格式

* * *

## 总结

本篇的核心可以归纳为三点：

1.  基础操作：`open / read / write`
1.  推荐写法：使用 `with` 管理文件资源
1.  常见场景：大文件处理 + JSON 读写

文件 I/O 本身不复杂，但在工程中几乎无处不在。熟悉这些模式后，可以直接应用到日志处理、数据落地、接口调试等多个场景中。

* * *

## with 的本质：上下文管理器

`with` 并不只用于文件读写，它的本质是**上下文管理（context management）机制**。文件只是一个常见应用场景。可以把它理解为：**在一段代码执行前后，自动做“准备”和“收尾”工作**。`with` 语句依赖一个对象，这个对象需要实现两个方法：

```
__enter__()
__exit__()
```

执行流程是：

```
with obj as x:
    # 执行代码块
```

等价于：

```
x = obj.__enter__()
try:
    # 执行代码块
finally:
    obj.__exit__()
```

也就是说：

-   `enter()`：进入时执行（资源初始化）
-   `exit()`：退出时执行（资源释放）

* * *

> ## with的其他用法
>
> 1.  ### 线程锁（非常常见）
>
> ```
> import threading
>
> lock = threading.Lock()
>
> with lock:
>     # 临界区代码
>     print("线程安全操作")
> ```
>
> 作用：
>
> -   自动加锁（enter）
> -   自动释放锁（exit）
>
> 避免忘记 `release()` 导致死锁。
>
> * * *
>
> 2.  ### 数据库连接 / 事务管理
>
> 很多数据库库支持：
>
> ```
> with connection:
>     cursor.execute("INSERT INTO table VALUES (1)")
> ```
>
> 或者：
>
> ```
> with connection.cursor() as cursor:
>     cursor.execute("SELECT * FROM table")
> ```
>
> 作用：
>
> -   自动提交或回滚事务
> -   自动关闭 cursor
>
> * * *
>
> 3.  ### 临时修改环境（如上下文切换）
>
> ```
> from decimal import localcontext, Decimal
>
> with localcontext() as ctx:
>     ctx.prec = 2
>     print(Decimal("1.12345") + Decimal("2.34567"))
>
> print(Decimal("1.12345") + Decimal("2.34567"))
> ```
>
> 作用：
>
> -   在 `with` 内修改配置
> -   退出后恢复原状态
>
> * * *
>
> 4.  ### 自定义上下文管理器
>
> 可以自己定义 `with` 的行为：
>
> ```
> class Timer:
>     def __enter__(self):
>         import time
>         self.start = time.time()
>         return self
>
>     def __exit__(self, exc_type, exc_val, exc_tb):
>         import time
>         print("耗时：", time.time() - self.start)
>
> with Timer():
>     sum(range(1000000))
> ```
>
> 用途：
>
> -   统计耗时
> -   日志记录
> -   资源管理
>
> * * *
>
> 5.  ### 更优雅的写法：contextlib
>
> Python 提供了更简洁的方式：
>
> ```
> from contextlib import contextmanager
> import time
>
> @contextmanager
> def timer():
>     start = time.time()
>     yield
>     print("耗时：", time.time() - start)
>
> with timer():
>     sum(range(1000000))
> ```
>
> 相比写类：
>
> -   更少代码
> -   更直观
>
> * * *
>
## 总结

`with` 的核心不是“文件操作”，而是**让“必须成对出现的操作”（获取/释放、开始/结束）变得安全且简洁**

常见组合包括：
-   打开 / 关闭（文件）
-   加锁 / 解锁（线程）
-   开始 / 提交（数据库）
-   设置 / 恢复（环境）