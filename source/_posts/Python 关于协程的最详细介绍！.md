---
title: Python 关于协程的最详细介绍！
date: 2026-04-07 23:30:00
categories:
  - 技术
tags:
  - Python
---

## 一、生成器 yield

上一篇Python知识点我们已经详细地介绍过生成器的定义了。让我们先来回顾一下，看一个简单例子：

```
def get_numbers(n):
    result = []
    for i in range(n):
        result.append(i)
    return result

nums = get_numbers(1000000)
```

这段代码会一次性生成一个百万级列表，占用大量内存。换成生成器：

```
def get_numbers(n):
    for i in range(n):
        yield i

nums = get_numbers(1000000)
for num in nums:
    print(num)
```

`yield` 会让函数"暂停"，每次只产出一个值，调用方拿到后再继续执行。两者的区别很直接：

-   **列表**：一次性把所有结果算出来，全部放进内存
-   **生成器**：需要一个，算一个，用完即丢

* * *

### send()：双向通信

生成器有一个不太常用但很重要的能力——可以通过 `send()` 向生成器内部传数据。`send ()` 的官方定义是：**`生成器.send(value)`** **：向生成器发送一个值，并唤醒生成器。** 发送的值，会成为上一个暂停的 `yield` 表达式的返回值；唤醒后，函数继续执行，直到下一个 `yield` 或函数结束。

```
def gen():
    value = yield 1
    print("received:", value)
    yield 2
    
# 创建生成器对象 g
g = gen()   
# 用 next(g) 唤醒生成器       
print(next(g))      # 输出 1
# 用 send(10) 发送数据
print(g.send(10))  # 输出：received: 10 → 再输出 2
```

执行流程：

1.  `next(g)` 运行到 `yield 1`，暂停，返回 `1`

1.  `g.send(10)` 会做两件事：

    1.  **给上一个暂停的 yield 赋值**：把 `10` 传给 `yield 1`，让 `value = 10`；
    1.  **继续执行生成器**，直到遇到下一个 `yield`。

这就不再是单向的数据流，而是"双向通信"。这个特性是后来协程的基础。

> 注意：第一次唤醒生成器，必须用 `next(g)` 或 `g.send(None)` 启动，因为第一次时生成器还没有「上一个 yield」，无法接收数据。

* * *

### yield from：委托生成器

当一个生成器需要调用另一个生成器时，朴素的做法是手动迭代：

```
def inner():
    yield 1
    yield 2
    yield 3

def outer():
    # 上一篇我们说到，for 做的事情 = iter() + next() + 捕获异常
    for val in inner():   # 相当于 next(inner())
        yield val         # outer 拿到 val，执行 yield val，暂停，返回 val；之后继续循环
    yield 4
```

这样写能跑，但很啰嗦。`yield from` 专门用来解决这个问题：

```
def outer():
    # 功能完全一样，代码更简洁。
    yield from inner()
    yield 4

for v in outer():
    print(v)
# 输出：1 2 3 4
```

`yield from` 的作用远不止"语法糖"。它会在 `outer` 和 `inner` 之间建立一条透明的双向通道：

-   **外部调用** **`send()`** **传入的值，会直接透传给** **`inner`**
-   **`inner`** **的** **`return`** **返回值，会成为** **`yield from`** **表达式的结果**

```
def inner():
    received = yield "ready"
    print("inner got:", received)
    return "done"

def outer():
    result = yield from inner()
    print("inner returned:", result)

g = outer()
print(next(g))          # 输出 ready
print(g.send("hello"))  # inner got: hello，然后 inner returned: done
```

这种机制使得生成器可以像调用栈一样层层嵌套，外层不需要关心内层的细节。这也是 Python 早期用生成器模拟协程时的核心工具（PEP 380）。

## 二、协程雏形：用生成器实现任务切换

在早期 Python 中，协程就是用生成器实现的。

```
def task1():
    print("task1 step1")
    yield
    print("task1 step2")

def task2():
    print("task2 step1")
    yield
    print("task2 step2")

t1 = task1()
t2 = task2()
next(t1)
next(t2)
next(t1)
next(t2)
```

输出：

```
task1 step1
task2 step1
task1 step2
task2 step2
```

两个任务"交替执行"——这种手动调度，就是协程最早的形态。但这种写法有几个明显的问题：

-   `yield` 语义混乱，既当返回值，又当暂停点
-   `send()` 用起来容易写错
-   嵌套调用时必须手动处理 `yield from`，代码量大
-   没有统一的错误处理和调度机制

因此，Python 3.4 起引入了 `asyncio`，3.5 起提供了 `async/await` 语法，把协程从"生成器技巧"升级成了语言级别的原生支持。

## 三、协程（Coroutine）

1.  ### 协程的定义

协程是一种可以在执行过程中主动暂停、并在某个时机恢复的函数。与线程不同，协程的切换由程序自己控制，不依赖操作系统调度，因此切换开销极低。

> 一个典型的线程模型是这样的：发起一次网络请求，线程就阻塞在那里等待响应，CPU 资源白白浪费。协程的思路是：等待期间主动让出控制权，让事件循环去处理其他任务，等 I/O 完成了再回来继续执行。

同步生成器 VS 异步协程：

| 同步生成器           | 异步协程              | 作用完全一样        |
| --------------- | ----------------- | ------------- |
| def + yield     | async def + await | 定义可暂停 / 恢复的函数 |
| yield           | await             | 暂停函数，等待完成后恢复  |
| next() / send() | 事件循环              | 唤醒函数继续执行      |
| 生成器对象           | 协程对象              | 暂停 / 恢复的实体    |

2.  ### 协程的适用场景

协程的优势集中在 **I/O** **密集型任务**，典型场景包括：

-   并发发起大量 HTTP 请求（爬虫、接口聚合）
-   数据库的异步读写
-   WebSocket 长连接处理
-   文件读写、消息队列消费

如果任务是 CPU 密集型（大量计算、图像处理），协程帮不上忙，应该用多进程。

3.  ### 基本语法

```
import asyncio

async def fetch_data():
    print("开始请求")
    # 暂停当前协程，等待 asyncio.sleep(1) 完成
    await asyncio.sleep(1)  # 模拟 I/O 等待
    # 一秒之后，自动唤醒协程，从👇开始恢复
    print("请求完成")
    return {"data": 42}

asyncio.run(fetch_data())
```

几个关键点：

-   `async def` 定义的函数是协程函数，调用它不会立刻执行，而是返回一个协程对象
-   `await` 后面跟一个可等待对象（协程、Task、Future），执行到这里会暂停当前协程，把控制权交还给事件循环
-   `asyncio.run()` 创建事件循环并运行，通常作为程序入口

> 为什么要用 `await asyncio.sleep(1)`，不用 `time.sleep(1)`？
>
> -   `time.sleep(1)`：**同步阻塞**，程序卡死不动；
> -   `asyncio.sleep(1)`：**异步非阻塞**，协程暂停，**CPU 可以去做别的事**，1 秒后自动回来。

4.  ### 并发执行多个协程

单独 `await` 一个协程是串行的，`asyncio.gather()` 才能实现并发：

```
import asyncio

async def task(name, delay):
    print(f"{name} 开始")
    await asyncio.sleep(delay)
    print(f"{name} 完成")
    return name

async def main():
    results = await asyncio.gather(
        task("A", 2),
        task("B", 1),
        task("C", 3),
    )
    print("所有结果:", results)

asyncio.run(main())
```

输出顺序：

```
A 开始
B 开始
C 开始
B 完成      # 1秒后
A 完成      # 2秒后
C 完成      # 3秒后
所有结果: ['A', 'B', 'C']
```

三个任务总耗时约 3 秒，而串行执行需要 6 秒。`gather()` 会等所有任务完成，结果按传入顺序返回（和完成顺序无关）。

5.  ### Task：后台任务

`asyncio.create_task()` 可以把协程包装成 Task，立即加入调度，不需要等待它完成：

```
async def background_job():
    await asyncio.sleep(2)
    print("后台任务完成")

async def main():
    # 把这个任务加入事件循环，让它在后台自动运行
    task = asyncio.create_task(background_job())
    print("主流程继续执行")
    # 这 1 秒里，后台任务也在同时运行
    await asyncio.sleep(1)
    print("主流程结束")
    # 如果没有 await task，主函数直接结束，后台任务会被强制杀死
    await task  # # 等background_job()跑完

asyncio.run(main())

'''输出：
主流程继续执行
主流程结束
后台任务完成
'''
```

对比理解：

-   没有 `create_task`：协程必须等 `await` 才会跑
-   加了 `create_task`：**任务在后台自动并发跑，不用等**

6.  ### 超时控制

```
async def slow_task():
    await asyncio.sleep(10)

async def main():
    try:
        await asyncio.wait_for(slow_task(), timeout=3.0)
    except asyncio.TimeoutError:
        print("超时了")

asyncio.run(main())
```

`wait_for()` 超时后会取消协程并抛出 `TimeoutError`，是处理慢接口、防止任务挂死的常用手段。

7.  ### 异步上下文管理器与异步迭代器

协程生态里，资源管理和迭代也有对应的异步版本：

```
# 异步上下文管理器
async with aiofiles.open("file.txt") as f:
    '''
    1. 调用异步打开文件
    2. 暂停当前协程，等待操作系统打开文件
    3. 打开完成 → 自动恢复，把文件对象赋值给 f
    '''
    content = await f.read()

# 异步迭代器
async for line in async_generator():
    '''
    1. 调用异步读文件
    2. 再次暂停协程，等待磁盘读取数据
    3. 读取完成 → 恢复执行，把数据给 content
    '''    
    process(line)
# async for = 每次循环都自动 await，专门处理需要等待才能拿到数据的场景（如流式读取数据、逐条接收网络消息）。
```

8.  ### 与线程的对比

|      | 协程        | 线程                 |
| ---- | --------- | ------------------ |
| 切换方式 | 主动让出（协作式） | 操作系统调度（抢占式）        |
| 切换开销 | 极低        | 较高（上下文切换）          |
| 适合场景 | I/O 密集    | I/O 密集 / 部分 CPU 场景 |
| 并发数量 | 可轻松达到数千   | 受系统限制，通常数百         |
| 共享状态 | 单线程，无竞争条件 | 需要加锁，容易出 bug       |

协程在高并发 I/O 场景下的资源利用率更高，但它是单线程的，一旦某个协程出现阻塞调用（比如误用了同步的 `time.sleep`），整个事件循环都会卡住。

## 四、脉络梳理

```
生成器 yield → 暂停 / 恢复
生成器 send() → 双向通信
生成器嵌套 yield from
异步协程 async/await（基于生成器）
异步任务 create_task（并发）
异步上下文 async with
异步迭代 async for
```
