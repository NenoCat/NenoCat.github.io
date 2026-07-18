---
title: Python 关于线程和进程的最详细介绍！
date: 2026-07-18 23:30:00
categories:
  - 技术
tags:
  - Python
---

今天我们来学习 Python 里面两个非常重要的概念——**线程（Thread）和进程（Process），** 理解两者的区别、用法及底层机制，都能帮助我们写出更高效、更稳健的代码。本篇文章包含的内容如下：

```
并发与并行
    │
    ├─ 并发（Concurrency）：交替执行，看起来像同时
    └─ 并行（Parallelism）：同时执行，真正的同时

进程 vs 线程
    │
    ├─ 进程：资源分配单位，隔离性好，开销大
    └─ 线程：调度单位，共享内存，开销小
```

```
Python线程
    │
    ├─ threading模块
    │     ├─ Thread对象创建
    │     ├─ start()启动
    │     └─ join()等待
    │
    ├─ GIL（全局解释器锁）
    │     ├─ 只影响CPU密集型
    │     └─ I/O密集型不受影响
    │
    ├─ 同步机制
    │     ├─ Lock（互斥锁）
    │     └─ 防止死锁
    │
    ├─ 通信机制
    │     └─ Queue（推荐）
    │
    └─ 线程池
          └─ ThreadPoolExecutor
```

```
Python进程
    │
    ├─ multiprocessing模块
    │     ├─ Process对象创建
    │     ├─ start()启动
    │     └─ join()等待
    │
    ├─ 进程通信
    │     ├─ Queue
    │     ├─ Pipe
    │     └─ 共享内存（Value, Array）
    │
    └─ 进程池
          └─ ProcessPoolExecutor
```
本文所有代码均基于 Python 3.13 测试通过。
## 一、进程与线程的基本概念

**进程（Process）** ，你可以把它想象成**一座独立的工厂**。每座工厂都有自己的地盘、设备、资源（比如电、水）。一座工厂倒闭了，不会影响到另一座工厂。

**线程（Thread）** ，你可以把它想象成工厂里的**工人**。同一座工厂里的工人们共享工厂的所有资源（设备、原料仓库等），但每个工人有自己的工位和工具。

用图来表示大概是这样的：

```
     进程A                    进程B
┌─────────────┐          ┌─────────────┐
│  线程1 线程2 │          │  线程1 线程2 │
│  (共享资源)  │          │  (共享资源)  │
└─────────────┘          └─────────────┘
     工厂A                     工厂B
```

**进程和线程的核心差异对比：**

|          | 进程                     | 线程                   |
| -------- | ---------------------- | -------------------- |
| 定义       | 正在运行的程序，是资源分配的基本单位     | 进程内的执行单元，是CPU调度的基本单位 |
| 创建方式     | fork()、multiprocessing | threading            |
| 内存关系     | 独立的内存空间                | 共享进程的内存空间            |
| 通信方式     | 复杂（Queue、Pipe、共享内存）    | 简单（直接读写共享变量）         |
| 开销       | 大（需要复制整个进程）            | 小（共享资源）              |
| 隔离性      | 强，一个崩溃不影响另一个           | 弱，一个崩溃可能导致整个进程崩溃     |
| 能否共享全局变量 | 不能                     | 能                    |

总的来说：

-   **进程是"资源分配"的单位，线程是"调度执行"的单位**
-   **一个进程至少包含一个线程（主线程）**
-   **线程依赖于进程，一个进程可以有多个线程**

## 二、Python 多线程

### 2.1 threading 模块

Python 提供了 `threading` 模块来操作线程。创建线程有两种方式，我们先看第一种：

1.  **直接创建线程对象。**

```
import threading
import time

def task(name, seconds):
    """模拟一个任务"""
    print(f"[{name}] 开始执行")
    time.sleep(seconds)
    print(f"[{name}] 执行完成，耗时 {seconds} 秒")   

t1 = threading.Thread(target=task, args=("线程A", 2))   # 创建线程对象
t2 = threading.Thread(target=task, args=("线程B", 1))  
# 线程A启动，主线程继续往下走（不等待）
t1.start()  
# 线程B启动，主线程继续往下走（不等待）
t2.start()  

# 此时：主线程、线程A、线程B 三者同时在跑

# 主线程暂停，等线程A结束
t1.join()  
# 主线程暂停，等线程B结束
t2.join()
print("所有任务执行完毕")

"""输出：
[线程A] 开始执行
[线程B] 开始执行
[线程B] 执行完成，耗时 1 秒
[线程A] 执行完成，耗时 2 秒
所有任务执行完毕
"""
```

**执行流程：**

1.  定义一个 `task` 函数，作为线程要执行的任务
1.  创建两个 `Thread` 对象，指定 `target` 为要执行的函数
1.  调用 `start()` 方法启动线程
1.  调用 `join()` 方法等待线程结束，确保子线程完成后再继续执行后续代码，

执行流程图解：

```
时间线 →
主线程:  start t1 ──→ start t2 ──→ [等t1] ──→ [等t2] ──→ print("所有任务...")
          ↓              ↓           ↑           ↑
线程A:   [========2秒========] ──────┘
线程B:   [==1秒==] ─────────────────┘
```

2.  **使用类的方式创建线程**

有时候我们需要一个更灵活的方式，可以继承 `Thread` 类：

```
import threading

class MyThread(threading.Thread):
    """
    自定义线程类，继承自 threading.Thread
    通过重写 run() 方法定义线程执行的任务
    """
    def __init__(self, name, count):
        super().__init__()
        self.name = name
        self.count = count
    # 线程启动后自动执行的方法（方法名必须是 run，不可更改）
    def run(self):
        for i in range(self.count):
            print(f"{self.name}: 第 {i+1} 次执行")

# 创建并启动所有线程（实现并发）
threads = []

for i in range(3):
    t = MyThread(name=f"工人{i+1}", count=3)
    threads.append(t)
    t.start()       

# 统一等待所有线程完成    
for t in threads:
    t.join()            

# 所有线程完成后，主线程继续执行
print("全部工人收工！") 
```

**输出：**

```
工人1: 第 1 次执行
工人1: 第 2 次执行
工人1: 第 3 次执行
工人2: 第 1 次执行
工人2: 第 2 次执行
工人3: 第 1 次执行
工人3: 第 2 次执行
工人3: 第 3 次执行
工人2: 第 3 次执行
全部工人收工！
```

* * *

### 2.2 Python 中的 GIL

1.  #### GIL的定义

GIL 的全称是 **Global Interpreter Lock** **（** **全局解释器锁** **）** 。它是 Python 解释器（CPython）内部的一个机制。

简单来说，**在同一时刻，一个 Python 进程里只能有一个线程执行 Python 字节码**。这意味着，就算你创建了多个线程，同一时刻也只有一个线程在工作。所以 Python 的多线程并不能真正实现 CPU 的并行计算。

Python 使用引用计数来管理内存。每个对象都有一个引用计数，当计数为0时，对象就会被销毁。如果多个线程同时修改引用计数，可能会导致计数不准确，进而引发内存泄漏或访问已释放的内存。

为了解决这个问题，Python 的设计者做了一个简单的决定：**同一时刻只允许一个线程执行 Python 代码**。虽然这限制了并行性，但大大简化了内存管理。因此，GIL 的存在主要是为了**保证 Python 内存管理的安全性**。

2.  #### **GIL** **的影响：CPU密集型 vs** **I/O** **密集型**

GIL 对不同类型的任务影响不同：

-   多线程在 **CPU 密集任务**中，无法真正并行
-   多线程在 **I/O** **密集任务**中，依然有效（因为 I/O 会释放 GIL）

  


对于**CPU密集型任务（计算密集型）** ：

```
import threading
import time

def cpu_task(n):
    """CPU密集型：纯粹计算"""
    count = 0
    for i in range(n):
        count += i ** 2
    return count

# 测试单线程
start = time.time()
for _ in range(4):
    cpu_task(10000000)
single_time = time.time() - start

# 测试多线程
start = time.time()
threads = []
for _ in range(4):
    t = threading.Thread(target=cpu_task, args=(10000000,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
    
multi_time = time.time() - start

print(f"单线程耗时: {single_time:.2f} 秒")
print(f"多线程耗时: {multi_time:.2f} 秒")
print(f"加速比: {single_time/multi_time:.2f}x")
```

**输出：**

```
单线程耗时: 5.21 秒
多线程耗时: 5.15 秒
加速比: 1.01x
```

> 可以看到，由于这是 CPU 密集型任务，在 Python 中受 GIL限制，多线程不会真正并行，你会发现单线程和多线程执行的时间差不多，甚至可能因为线程切换开销而更慢。

  


**对**于**I/O** **密集型任务**：

```
import threading
import time

def io_task(duration):
    """I/O密集型：模拟等待"""
    time.sleep(duration)

# 测试单线程
start = time.time()
for _ in range(10):
    io_task(0.5)
single_time = time.time() - start

# 测试多线程
start = time.time()
threads = []
for _ in range(10):
    t = threading.Thread(target=io_task, args=(0.5,))
    threads.append(t)
    t.start()
for t in threads:
    t.join()
multi_time = time.time() - start

print(f"单线程耗时: {single_time:.2f} 秒")
print(f"多线程耗时: {multi_time:.2f} 秒")
print(f"加速比: {single_time/multi_time:.2f}x")
```

**输出：**

```
单线程耗时: 5.00 秒
多线程耗时: 0.50 秒
加速比: 9.93x
```

**总结**：

-   **CPU密集型任务**：多线程反而更慢，因为GIL导致无法并行
-   **I/O** **密集型任务**：多线程效果显著，因为线程在等待I/O时可以释放GIL

* * *

### 2.3 线程同步：锁（Lock）

前面说了，同一个进程的线程共享内存。这既是优点，也是隐患。如果多个线程同时修改同一个变量，就会出现"竞态条件"（Race Condition）。

  


不加锁的问题演示：

```
import threading

# 全局计数器
counter = 0

def increment():
    global counter
    for _ in range(1000000):
        # 非原子操作：读取 -> 加1 -> 写入
        # 线程切换可能发生在任意两步之间，导致覆盖
        counter += 1

# 创建两个线程
t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)

t1.start()
t2.start()
t1.join()
t2.join()

print(f"计数器最终值: {counter}")
print(f"期望值: 2000000")
```

**输出：**

```
计数器最终值: 1437823
期望值: 2000000
ps: 可能你的输出都是2000000，可能是因为更高版本的Python使得 counter += 1 这种简单操作
在字节码层面往往"碰巧"是原子的，不会中途切换线程
```

原因：`counter += 1` 看似是一个操作，实际上分解成三步：

1.  读取 counter 的值
1.  计算 counter + 1
1.  写回 counter

当两个线程同时执行时，可能发生：

-   线程A读取counter=100
-   线程B读取counter=100
-   线程A计算101，写入counter=101
-   线程B计算101，写入counter=101

两个线程都读到 100，各自加 1 后都写回 101，本该是 102，结果却是 101——这就是数据竞争。

* * *

1.  #### **使用Lock解决问题**

```
import threading

def increment():
    global counter
    for _ in range(1000000):
        with lock:           # 获取锁
            counter += 1     # 受保护的操作
        # with 块结束，锁自动释放

counter = 0
lock = threading.Lock()

t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)

t1.start()
t2.start()
t1.join()
t2.join()

print(f"计数器最终值: {counter}")
print(f"期望值: 2000000")
```

**输出：**

```
计数器最终值: 2000000
期望值: 2000000
```

**关键点**：使用 `with lock:` 语句来确保锁的获取和释放。如果不用 with，就需要手动 `lock.acquire()` 和 `lock.release()`。这个知识点可以看[Day5 Python文件读写 with的本质](https://juejin.cn/post/7624100510838145050)。

2.  #### 死锁问题

锁虽好，但用不好会出问题，最典型的就是**死锁** **（** **Deadlock** **）** 。死锁是多线程/多进程编程中最棘手的问题之一。当两个或多个线程互相持有对方需要的资源，同时又都在等待对方释放资源时，就会形成循环等待，导致所有相关线程永远阻塞，程序陷入停滞状态。

死锁的四个必要条件：

| 条件        | 说明            | 破解思路       |
| --------- | ------------- | ---------- |
| **互斥**    | 资源一次只能被一个线程占用 | 难以改变（锁的本质） |
| **占有且等待** | 持有资源的同时等待其他资源 | 一次性申请所有资源  |
| **不可抢占**  | 已获得的资源不能被强制剥夺 | 允许超时放弃     |
| **循环等待**  | 形成等待链环        | 统一资源获取顺序   |

**如何避免** **死锁** **呢？**

1.  统一加锁顺序（所有线程都先拿A再拿B）
1.  使用 `try-finally` 确保锁释放
1.  使用 `threading.RLock()` 允许同一线程重复获取锁（在特定场景下非常有用，但并不是常用的通用锁方案。）

```
# 方案1：统一获取顺序（推荐）
def safe_thread_1():
    with lock_a:  # 先A
        with lock_b:  # 再B
            pass

def safe_thread_2():
    with lock_a:  # 也是先A
        with lock_b:  # 再B
            pass

# 方案2：使用超时
if lock_a.acquire(timeout=2):
    try:
        if lock_b.acquire(timeout=2):
            try:
                pass
            finally:
                lock_b.release()
    finally:
        lock_a.release()

# 方案3：使用 threading.RLock()
rlock = threading.RLock()

def func_a_safe():
    with rlock:
        print("func_a")
        func_b_safe()  # 同一线程可以重复获取

def func_b_safe():
    with rlock:  # 计数器+1，不会阻塞
        print("func_b")
```

* * *

### 2.4 线程通信：Queue

虽然线程可以共享变量，但直接共享变量容易出问题。Python 的 `queue.Queue` 是专门设计用于线程间通信的线程安全队列。

**Queue的优点**：

-   线程安全，无需手动加锁
-   提供了阻塞机制

| 方法                     | 行为           | 适用场景    |
| ---------------------- | ------------ | ------- |
| `put(item)`            | 队列满时阻塞，直到有空间 | 生产者控制流速 |
| `put(item, timeout=1)` | 阻塞最多1秒，超时抛异常 | 避免无限等待  |
| `put_nowait(item)`     | 队列满时立即抛异常    | 非阻塞写入   |
| `get()`                | 队列空时阻塞，直到有数据 | 消费者等待数据 |
| `get(timeout=1)`       | 阻塞最多1秒，超时抛异常 | 定时轮询    |
| `get_nowait()`         | 队列空时立即抛异常    | 非阻塞读取   |

-   生产者-消费者模式的天然实现

Queue 是生产者-消费者模式的最佳载体，这种模式解耦了数据生产和消费的节奏：

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  生产者A    │     │             │     │   消费者X   │
│  (速度快)   │────→│   Queue     │────→│  (处理慢)   │
├─────────────┤     │  (缓冲区)   │     ├─────────────┤
│  生产者B    │     │             │     │   消费者Y   │
│  (速度快)   │────→│  线程安全    │────→│  (处理慢)   │
└─────────────┘     │  自动阻塞    │     └─────────────┘
                    │  容量控制    │
                    └─────────────┘
                    
优势：
- 生产者和消费者速度不匹配时，Queue 作为缓冲平衡负载
- 双方无需知道对方存在，通过 Queue 解耦
- 支持多个生产者和多个消费者并发工作
```

完整示例：

```
import queue
import threading
import time
import random

# 创建有界队列，防止内存无限增长
task_queue = queue.Queue(maxsize=20)
NUM_PRODUCERS = 3
NUM_CONSUMERS = 2

def producer(pid):
    """生产者：生成任务"""
    for i in range(5):
        task = f"任务-{pid}-{i}"
        task_queue.put(task)  # 队列满时自动阻塞
        print(f"[生产者{pid}] 提交: {task}")
        time.sleep(random.uniform(0.1, 0.3))  # 模拟生产耗时
    print(f"[生产者{pid}] 完成")

def consumer(cid):
    """消费者：处理任务"""
    while True:
        try:
            # 阻塞等待，1秒超时用于检查退出条件
            task = task_queue.get(timeout=1)
            print(f"[消费者{cid}] 处理: {task}")
            time.sleep(random.uniform(0.2, 0.5))  # 模拟处理耗时
            task_queue.task_done()  # 标记完成，让join()能正常工作
        except queue.Empty:
            # 队列为空且超时，检查是否应该退出
            if task_queue.empty():
                print(f"[消费者{cid}] 空闲退出")
                break

# 启动生产者
producers = []
for i in range(NUM_PRODUCERS):
    t = threading.Thread(target=producer, args=(i,))
    t.start()
    producers.append(t)

# 启动消费者
consumers = []
for i in range(NUM_CONSUMERS):
    t = threading.Thread(target=consumer, args=(i,))
    t.start()
    consumers.append(t)

# 等待所有生产者完成
for t in producers:
    t.join()

# 等待队列中所有任务被处理完
task_queue.join()
print("所有任务处理完毕")
```

**输出示例：**

```
[生产者0] 提交: 任务-0-0
[生产者1] 提交: 任务-1-0
[生产者2] 提交: 任务-2-0
[消费者0] 处理: 任务-0-0
[消费者1] 处理: 任务-1-0
[生产者2] 提交: 任务-2-1
[生产者1] 提交: 任务-1-1
[生产者0] 提交: 任务-0-1
[生产者2] 提交: 任务-2-2
[生产者1] 提交: 任务-1-2
……
```

* * *

### 2.5 线程池：ThreadPoolExecutor

想象一下：如果每来一个任务就创建一个新线程，任务完成后又销毁，频繁的创建和销毁会有很大开销。

线程池的思路是：**预先创建一批线程，需要时拿来用，用完放回池子里**。

它的核心优势在于：

-   减少线程创建/销毁的开销：复用已有线程
-   控制并发数量：避免系统资源耗尽
-   简化任务管理：自动调度、结果收集

线程池可以由`concurrent.futures.ThreadPoolExecutor`实现：

```
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def task(n):
    time.sleep(1)
    return f"任务{n}完成"

# 创建线程池（最多3个线程）
with ThreadPoolExecutor(max_workers=3) as executor:
    # 方式1：提交单个任务
    future = executor.submit(task, 1)
    result = future.result()  # 阻塞获取结果
    
    # 方式2：批量提交（推荐）
    futures = [executor.submit(task, i) for i in range(5)]
    # as_completed：按任务完成顺序返回结果
    for future in as_completed(futures):
        print(future.result())
    
    # 方式3：map（保持顺序）
    # map 会等前一个结果取出后，才继续迭代，天然节流。
    results = executor.map(task, range(5))
```

其中：

Future 对象： 代表"未来的结果"，是任务的句柄。可以：

-   `future.result()` — 阻塞获取结果
-   `future.done()` — 检查是否完成（非阻塞）
-   `future.cancel()` — 尝试取消（未开始时有效）

**输出：**

```
任务1完成
任务0完成
任务2完成
任务4完成
任务3完成
```

三种方式对比：

```
时间轴 →
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

方式1（单个提交）:
[提交1]────[等待1s]────[获取结果]  耗时: 1s

方式2（批量+as_completed）:
[提交0,1,2,3,4] 
   ↓
[线程0: 任务0]════╗
[线程1: 任务1]════╬════╗  ← 1s后完成，立即打印
[线程2: 任务2]════╝    ║
[线程0: 任务3]═════════╬════╗  ← 2s后完成，立即打印  
[线程1: 任务4]═════════╝    ║
                            ↓
                         全部完成
耗时: 2s（5个任务/3线程 = 2批）

方式3（map）:
执行过程同方式2，但输出强制按 0,1,2,3,4 顺序
```

如何选择 `max_workers`？

-   IO密集型（网络请求、文件读写）：可适当调高（10-50）
-   CPU密集型：线程数 ≤ CPU 核心数（GIL限制，建议用进程池）

## 三、Python 多进程

### 3.1 multiprocessing

Python 的 `multiprocessing` 模块允许我们创建多个进程，每个进程有独立的内存空间，绕过 GIL。

```
import multiprocessing

def cpu_task(n):
    """CPU密集型任务"""
    count = 0
    for i in range(n):
        count += i ** 2
    return count

if __name__ == '__main__':
    # 使用进程池，自动管理4个进程
    with multiprocessing.Pool(4) as pool:
        # 4个进程同时计算，返回4个结果组成的列表
        results = pool.map(cpu_task, [10000000] * 4)
    
    print(f"结果: {results}")
    print("所有进程完成！")
    
'''输出：
结果: [333333283333335000000, 333333283333335000000, 333333283333335000000, 333333283333335000000]
所有进程完成！
'''
```

* * *

### 3.2 进程间通信

进程之间不能直接共享内存，需要通过特殊机制通信。

1.  #### Queue（队列）

```
import multiprocessing
import time

def producer(q):
    """生产者：发送消息到队列"""
    for i in range(5):
        message = f"消息-{i}"
        q.put(message)
        print(f"[生产者] 发送: {message}")
        time.sleep(0.1)  # 模拟生产耗时
    q.put(None)  # 发送结束信号

def consumer(q, name):
    """消费者：从队列接收消息"""
    while True:
        item = q.get()
        if item is None:  # 收到结束信号
            break
        print(f"[消费者-{name}] 收到: {item}")
        time.sleep(0.2)  # 模拟处理耗时

if __name__ == "__main__":
    # 创建队列（必须在主进程创建，再传给子进程）
    q = multiprocessing.Queue()
    
    # 创建进程
    p = multiprocessing.Process(target=producer, args=(q,))
    c = multiprocessing.Process(target=consumer, args=(q, "A"))
    
    # 启动进程
    c.start()  # 消费者先启动，等待接收
    p.start()  # 生产者后启动，开始发送
    
    # 等待完成
    p.join()   # 等生产者结束
    c.join()   # 等消费者结束（生产者已发None信号，消费者会自己退出）
    
    print("进程通信完成！")
```

> **注意**：Queue 必须在主进程中创建，然后传给子进程。

* * *

2.  #### Pipe（管道）

```
import multiprocessing

def sender(conn):
    """发送者：通过管道发送消息"""
    messages = ["你好，我是发送者", "这是第二条消息", "结束"]
    
    for msg in messages:
        conn.send(msg)
        print(f"[发送者] 发送: {msg}")
    
    conn.close()  # 发送完毕，关闭连接

def receiver(conn):
    """接收者：通过管道接收消息"""
    while True:
        try:
            msg = conn.recv()
            print(f"[接收者] 收到: {msg}")
            
            # 收到结束信号则退出
            if msg == "结束":
                print("[接收者] 收到结束信号，停止接收")
                break
                
        except EOFError:
            # 管道已关闭，没有更多数据
            print("[接收者] 管道已关闭")
            break
    
    conn.close()

if __name__ == "__main__":
    # 创建双向管道，返回两个连接对象
    parent_conn, child_conn = multiprocessing.Pipe()
    
    # 创建进程：sender用child_conn，receiver用parent_conn
    p1 = multiprocessing.Process(target=sender, args=(child_conn,))
    p2 = multiprocessing.Process(target=receiver, args=(parent_conn,))
    
    # 启动进程
    p2.start()  # 接收者先启动，等待数据
    p1.start()  # 发送者后启动
    
    # 等待完成
    p1.join()
    p2.join()
    
    print("Pipe 通信完成！")
```

**执行流程：**

1.  `Pipe()` 返回一对连接对象
1.  父进程持有 `parent_conn`，子进程持有 `child_conn`
1.  发送端用 `send()`，接收端用 `recv()`
1.  管道是双向的，也可以双向通信

**输出：**

```
[发送者] 发送: 你好，我是发送者
[接收者] 收到: 你好，我是发送者
[发送者] 发送: 这是第二条消息
[接收者] 收到: 这是第二条消息
[发送者] 发送: 结束
[接收者] 收到: 结束
[接收者] 收到结束信号，停止接收
Pipe 通信完成！
```

* * *

3.  #### 共享内存（Value, Array）

对于需要频繁访问的小数据，可以使用共享内存：

```
import multiprocessing

def increment(counter, lock):
    """对共享计数器执行100万次自增（带锁保护）"""
    for _ in range(100000):
        with lock:  # 获取锁，确保同一时刻只有一个进程修改
            counter.value += 1

if __name__ == "__main__":
    # 创建共享变量：有符号整数（'i'），初始值为0
    counter = multiprocessing.Value('i', 0)
    
    # 创建锁，用于同步访问共享变量
    lock = multiprocessing.Lock()
    
    # 创建2个进程
    processes = []
    for _ in range(2):
        p = multiprocessing.Process(target=increment, args=(counter, lock))
        processes.append(p)
        p.start()
    
    # 等待所有进程完成
    for p in processes:
        p.join()
    
    print(f"计数器最终值: {counter.value}")
    print(f"期望值: 200000")
    print(f"结果正确: {counter.value == 2000000}")
```

**执行流程：**

1.  `Value('i', 0)` 创建一个共享的整数，初始值为0
1.  `'i'` 是类型码，表示有符号整数
1.  进程通过 `counter.value` 访问和修改共享值
1.  需要加锁保证操作的原子性

* * *

### 3.3 进程池：ProcessPoolExecutor

和线程池类似，进程池使用 `ProcessPoolExecutor`：

```
import concurrent.futures
import multiprocessing
import time

def cpu_task(n):
    """CPU密集型任务：大量计算"""
    count = 0
    for i in range(n):
        count += i ** 2
    return count

def io_task(n):
    """I/O密集型任务：模拟等待（如网络请求、文件读写）"""
    time.sleep(0.1)
    return f"IO任务{n}完成"

if __name__ == "__main__":
    # ========== CPU密集型任务 ==========
    print("=== CPU密集型任务（用进程池）===")
    start = time.time()
    
    # ProcessPoolExecutor：利用多核CPU并行计算
    with concurrent.futures.ProcessPoolExecutor(max_workers=4) as executor:
        # 提交4个任务，每个计算500万次平方和
        futures = [executor.submit(cpu_task, 5000000) for _ in range(4)]
        
        # 按完成顺序获取结果
        for future in concurrent.futures.as_completed(futures):
            result = future.result()
            print(f"任务完成，结果: {result}")
    
    print(f"总耗时: {time.time() - start:.2f} 秒\n")

    # ========== I/O密集型任务 ==========
    print("=== I/O密集型任务（用线程池）===")
    start = time.time()
    
    # ThreadPoolExecutor：线程切换开销小，适合等待型任务
    with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
        # 提交8个I/O任务
        futures = [executor.submit(io_task, i) for i in range(8)]
        
        # 按完成顺序获取结果
        for future in concurrent.futures.as_completed(futures):
            print(f"获取: {future.result()}")
    
    print(f"总耗时: {time.time() - start:.2f} 秒")
```

**执行流程：**

1.  `ProcessPoolExecutor` 会自动管理进程
1.  提交的任务会在多个进程中并行执行
1.  自动完成进程的生命周期管理

## 四、线程 vs 进程

| 对比项    | 线程         | 进程           |
| ------ | ---------- | ------------ |
| 内存     | 共享进程内存     | 独立内存空间       |
| GIL    | 受GIL限制     | 不受GIL限制      |
| 创建速度   | 快          | 慢            |
| 通信     | 简单（共享变量）   | 复杂（需IPC）     |
| 稳定性    | 一个崩溃可能影响其他 | 隔离性好，崩溃不影响其他 |
| CPU密集型 | 效果差        | 效果好          |
| I/O密集型 | 效果好        | 效果也可以，但开销大   |
| 资源消耗   | 小          | 大            |

**用线程的场景**：

-   **I/O** **密集型任务**：网络请求、文件读写、API调用等，线程在等待时可以释放GIL
-   **需要频繁通信**：线程间共享数据非常方便
-   **资源受限**：线程开销小

**用进程的场景**：

-   **CPU密集型任务**：需要真正并行计算，如图像处理、数值计算
-   **稳定性要求高**：进程隔离，一个崩溃不影响其他
-   **需要绕过** **GIL**：Python多线程无法绑过GIL，只有多进程可以

## 五、注意事项

1.  #### GIL不会影响I/O操作

有一个错误的理解是，因为GIL的存在，很多人认为Python多线程完全没用。

然而，GIL只影响CPU计算，不影响I/O操作。当线程在等待网络、文件等I/O时，会自动释放GIL，所以**多线程** **对I/O密集型任务仍然非常有效**。

* * *

2.  #### daemon线程的注意事项

```
import threading
import time

def daemon_task():
    """守护线程：后台运行，主线程结束自动退出"""
    while True:
        print("守护线程: 还在运行...")
        time.sleep(1)

def main_task():
    """主任务"""
    print("主任务开始")
    time.sleep(3)
    print("主任务完成")

if __name__ == "__main__":
    # 创建守护线程（daemon=True）
    # 特点：主线程结束时，守护线程会被强制终止
    d_thread = threading.Thread(target=daemon_task, daemon=True)
    d_thread.start()
    
    # 执行主任务
    main_task()
    
    print("主进程结束")
    # 到这里主线程结束，守护线程d_thread也随之终止
```

**注意**：

-   如果不设置 `daemon=True`，主进程会一直等待守护线程结束
-   设置为守护线程后，主进程退出时守护线程会自动终止。因此，守护线程的任务可能没有完成就被强制终止。

* * *

3.  #### 进程间不能共享全局变量

`multiprocessing.Process` 创建的是独立进程，不是线程。每个进程有独立的虚拟内存空间。

```
import multiprocessing

# 全局变量
global_var = 100

def worker():
    global global_var
    global_var = 200
    print(f"子进程: global_var = {global_var}")

if __name__ == "__main__":
    print(f"主进程开始: global_var = {global_var}")
    
    p = multiprocessing.Process(target=worker)
    p.start()
    p.join()
    
    print(f"主进程结束: global_var = {global_var}")
    # 仍然是100，子进程的修改没有影响主进程
    # 因为 multiprocessing 创建的是独立进程，拥有各自的内存空间
    
'''输出：
主进程开始: global_var = 100
子进程: global_var = 200
主进程结束: global_var = 100
'''
```

**关键点**：子进程修改的全局变量只存在于子进程的内存空间中，主进程看不到。因此，子进程修改`global_var`只影响自己的内存副本

* * *

4.  #### 正确的关闭方式

```
import concurrent.futures
import time

def long_task():
    """模拟耗时任务"""
    time.sleep(5)
    return "完成"

def graceful_shutdown(executor):
    """优雅关闭线程池"""
    print("正在关闭线程池...")
    executor.shutdown(wait=True)  # 等待所有任务完成
    print("线程池已关闭")

def main():
    # 方法1：使用 with 语句（推荐，自动管理资源）
    with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
        future = executor.submit(long_task)
        
        try:
            # 设置 1 秒超时
            result = future.result(timeout=1)
            print(f"任务结果: {result}")
        except concurrent.futures.TimeoutError:
            print("任务超时！")
            future.cancel()  # 尝试取消任务（如果还未开始）
        except Exception as e:
            print(f"任务出错: {e}")
    
    # with 语句退出时会自动调用 shutdown，无需手动调用
    print("程序结束")

    # 方法2：不使用 with，手动管理（展示 graceful_shutdown 的用法）
    # executor = concurrent.futures.ThreadPoolExecutor(max_workers=4)
    # try:
    #     future = executor.submit(long_task)
    #     result = future.result(timeout=6)  # 给足够时间完成
    #     print(f"任务结果: {result}")
    # except concurrent.futures.TimeoutError:
    #     print("任务超时！")
    # finally:
    #     graceful_shutdown(executor)

if __name__ == "__main__":
    main()
```

**关键点**：

-   使用 `with` 语句或手动调用 `shutdown()`
-   `wait=True` 会等待所有任务完成
-   使用 `Future.result(timeout=...)` 防止无限等待

## 六、总结

综上，Python中的线程与进程虽各有特性，但核心用途都是实现多任务处理：**线程依托** **共享内存** **实现轻量级调度，适合** **I/O** **密集型场景；进程凭借独立资源隔离，能突破** **GIL** **限制，适配CPU密集型任务。** 掌握threading与multiprocessing模块的使用，理解GIL的影响、同步与通信机制，以及线程池、进程池的应用，能让我们根据实际场景灵活选择合适的并发方式，大幅提升Python程序的运行效率与稳定性。

| 任务类型                 | 推荐方案                                     |
| -------------------- | ---------------------------------------- |
| **CPU 密集型**          | `ProcessPoolExecutor`（多进程，绕过 GIL，利用多核）   |
| **I/O** **密集型**      | `ThreadPoolExecutor` 或 `asyncio`（协程，轻量级） |
| **混合型**              | 根据 CPU/I/O 比例选择，或组合使用                    |
| **需要** **内存** **隔离** | `Process`（独立内存空间，数据安全）                   |
| **需要共享数据**           | `Thread`（共享内存，但需注意加锁防止竞态）