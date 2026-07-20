---
title: Python中的迭代器与生成器
date: 2026-04-06 23:30:01
categories:
  - 技术
tags:
  - Python
---

在日常开发中，我们经常写 `for` 循环，却很少去思考它是如何工作的。本篇从迭代机制入手，再过渡到生成器。

## 一、可迭代对象 vs 迭代器

先看一个常见的循环：

```
nums = [1, 2, 3]
for n in nums:
    print(n)
```

这里的 `nums` 是**可迭代对象（Iterable）** ，它的特点是：可以被 `for` 遍历。而真正执行遍历的是**迭代器** **（** **Iterator** **）** 。使用 `iter()` 把可迭代对象变成**迭代器。**

> **Python中的可迭代对象有：** **`list`** **、** **`tuple`** **、** **`str`** **、** **`dict`** **、** **`set`** **、** **`range`** **。**

```
nums = [1, 2, 3]
it = iter(nums)  # 把列表变成迭代器

# 每调用一次 next (it)，就从迭代器里取一个值，指针往后走一步。
print(next(it))  # 1
print(next(it))  # 2
print(next(it))  # 3
# 取完了再取，就会报错 StopIteration（迭代结束）
```

总结一下关系：

-   可迭代对象：可以被 `iter()` 转换成迭代器
-   迭代器：可以用 `next()` 一个个取值

* * *

## 二、for 循环的本质

`for` 循环本质上就是一个“自动调用 next 的过程”：

```
nums = [1, 2, 3]
it = iter(nums)

while True:
    try:
        n = next(it)
        print(n)
    except StopIteration:
        break
```

也就是说：

> `for` 做的事情 = `iter()` + `next()` + 捕获异常

理解这一点后，再看生成器会更自然。

* * *

## 三、生成器函数（yield）

生成器的核心在于 `yield`，它会让函数“暂停执行”。你可以这么理解，**`yield`** **= 带暂停功能的 return，** 它和`return`的区别是：

-   `return`：**函数直接结束**
-   `yield`：**函数暂停在这里，下次继续从这行往下跑**

```
def count_up_to(n):  # 定义函数，最多数到 n
    i = 1            # 从 1 开始数
    while i <= n:    # 只要没超过 n，就一直循环
        yield i      # 暂停，返回当前的 i，记住位置
        i += 1       # 下次被唤醒时，执行这行：i+1
```

调用时：

```
# 创建生成器（此时函数还没开始跑）
gen = count_up_to(3)

# 第一次 next() → 跑到 yield 1，暂停，返回 1
print(next(gen))  # 输出 1

# 第二次 next() → 从暂停处继续，i=2，yield 2，暂停
print(next(gen))  # 输出 2

# 第三次 next() → 继续，i=3，yield 3，暂停
print(next(gen))  # 输出 3

# 第四次 next() → 循环结束，函数退出 → 报错 StopIteration
print(next(gen))
```

本质上，生成器就是一种**自动实现了** **迭代器** **协议的对象**。即，Python 看到 `yield`，就**自动在后台**：

-   生成一个对象
-   实现了 `iter`
-   实现了 `next`
-   自动记录暂停位置
-   自动在结束时抛 `StopIteration`

* * *

## 四、生成器表达式

列表推导式 vs 生成器表达式:

```
# 列表推导式，瞬间全部算完，放进内存
nums = [x * x for x in range(5)]

# 生成器表达式，什么都不计算，通过next()取值
gen = (x * x for x in range(5))
```

使用方式对比：

```
# 列表：一次性生成好
nums = [x*x for x in range(5)]
print(nums)       # [0, 1, 4, 9, 16]
print(nums[2])    # 4 可以直接索引
print(nums[3])    # 9 可以随便取

# 生成器：按需生成
gen = (x*x for x in range(5))
print(gen)        # 打印的是一个生成器对象，不是数据
print(next(gen))  # 0  要一个，算一个
print(next(gen))  # 1
print(next(gen))  # 4
```

* * *

## 五、什么时候用生成器？

**当数据 很大 / 很慢 / 只用一次 时，优先考虑生成器。**

### 1）数据量很大（无法一次性加载）

```
def read_big_file(path):
    with open(path) as f:
        for line in f:
            yield line
```

如果改成：

```
lines = f.readlines()
```

当文件很大时，会一次性占用大量内存。生成器按行读取，可以把内存占用控制在一个稳定范围。

* * *

### 2）数据是“流式”的

例如接口分页、消息队列、日志流：

```
def fetch_pages():
    page = 1
    while True:
        data = request_api(page)
        if not data:
            break
        yield from data
        page += 1
```

特点是：

-   数据不是一次性准备好的
-   需要“边产生边消费”

* * *

### 3）只需要遍历一次

如果数据只会被消费一次（例如过滤日志、扫描文件），生成器更合适：

```
gen = (line for line in read_log("app.log") if "ERROR" in line)
```

相比列表，不会提前做无用计算。

## 总结

| 概念     | 说明                                  |
| ------ | ----------------------------------- |
| 可迭代对象  | 实现了 __iter__ 的对象，可被 for 循环遍历        |
| 迭代器    | 经 iter() 转换后，支持 next() 逐步取值         |
| 生成器函数  | 含 yield 的函数，调用后返回生成器对象              |
| 生成器表达式 | (expr for x in iterable)，圆括号版的列表推导式 |
| 核心优势   | 惰性求值，按需产出，内存占用与数据量无关