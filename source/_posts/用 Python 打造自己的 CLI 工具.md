---
title: 用 Python 打造自己的 CLI 工具
date: 2026-04-13 23:30:04
categories:
  - 技术
tags:
  - Python
---

CLI（ Command Line Interface）最近特别火，我还特意写了一篇文章分析了一下：[AI 时代，为什么说万物皆可 CLI？](https://nenocat.github.io/AI%20%E6%97%B6%E4%BB%A3%EF%BC%8C%E4%B8%BA%E4%BB%80%E4%B9%88%E8%AF%B4%E4%B8%87%E7%89%A9%E7%9A%86%E5%8F%AF%20CLI%EF%BC%9F/)。在码农眼中，命令行工具在日常开发中出现的频率很高，不管是数据处理脚本、自动化任务，还是代码辅助工具，背后基本都有 CLI 的影子。因此，本篇内容就来详细地讲一讲CLI是怎么写的，以及它的价值。

## 一、sys.argv

最简单的 CLI，本质上就是“读取命令行参数”。

```
import sys

print(sys.argv)
```

运行：

```
python script.py hello world
```

输出：

```
['.\script.py', 'hello', 'world']
```

可以看到：

-   `sys.argv[0]` 是脚本名
-   后面的元素是输入参数

举一个简单例子：实现一个加法工具

```
import sys

a = int(sys.argv[1])
b = int(sys.argv[2])

print(a + b)
```

运行：`python add.py 3 5`，会输出`8`。

这种方式适合快速验证，但存在明显问题：

-   参数位置必须严格对应
-   不支持参数说明
-   扩展性较差

## 二、argparse

**`argparse`** **是 Python** **标准库** **中专门用于构建** **命令行工具** **的模块，可以让** **CLI** **更清晰、更易用。**

1.  ### 基本用法

```
import argparse

# description 参数用于设置帮助信息描述，当用户运行 -h 或 --help 时会显示
parser = argparse.ArgumentParser(description="简单加法工具")
# type=int 表示该参数会被自动转换为整数类型，help 是该参数的帮助说明，会在帮助信息中显示
parser.add_argument("a", type=int, help="第一个数字")
parser.add_argument("b", type=int, help="第二个数字")
# 调用 parse_args() 方法解析命令行传入的参数
# 返回一个命名空间对象（Namespace），可以通过属性访问各个参数值
args = parser.parse_args()

print(args.a + args.b)
```

运行：` python add.py  `3 5，输出：`8`。若运行：`python add.py -h`，输出：

```
usage: script.py [-h] a b

简单加法工具

positional arguments:
  a           第一个数字
  b           第二个数字

options:
  -h, --help  show this help message and exit
```

2.  ### 支持可选参数

```
parser.add_argument(
    "--verbose",           # 参数名，以 -- 开头表示是可选参数
    action="store_true",   # 当用户提供了 --verbose，将args.verbose设置为true
    help="输出详细信息"     # 帮助文档中显示的说明文字
)
```

```
if args.verbose:
    print(f"{args.a} + {args.b} = {args.a + args.b}")
else:
    print(args.a + args.b)
```

运行：`python add.py 3 5 --verbose`，则输出为：`3 + 5 = 8`

## 三、一个简单日志统计 CLI

假设有一个日志文件 `log.txt`：

```
INFO: start process
ERROR: failed to load
INFO: retry
ERROR: timeout
```

我们希望通过 CLI 统计某种日志级别的数量：

```
import argparse

def count_logs(file_path, level):
    count = 0
    with open(file_path, "r") as f:
        for line in f:
            if line.startswith(level):
                count += 1
    return count

parser = argparse.ArgumentParser(description="日志统计工具")

parser.add_argument("file", help="日志文件路径")
parser.add_argument("--level", default="ERROR", help="日志级别")

args = parser.parse_args()

result = count_logs(args.file, args.level)
print(f"{args.level} count: {result}")
```

运行：`python log_cli.py log.txt --level ERROR`，输出：`ERROR count: 2`

## 四、CLI 的真正价值

换个角度想，命令行接口的本质是**把一段逻辑暴露成可调用的接口**，这个接口不只是给人用的。

1.  ### **自动化入口**

CLI 工具天然适合接入 cron 定时任务或 CI/CD 流水线。上面这个日志工具，加一行 cron 就变成了每天自动跑的监控任务：

```
# 每天早上 8 点检查昨天的错误日志，结果写入文件
0 8 * * * python log_filter.py /var/log/app.log --level ERROR >> /tmp/daily_report.txt
```

脚本不需要改动，只是换了一种触发方式。

2.  ### **工具接口**

当系统里有多个 CLI 工具时，它们可以通过管道或脚本串联，形成处理流水线：

bash

```
# 提取错误日志 → 发送到另一个分析脚本
python log_filter.py app.log --level ERROR | python analyze.py --format json
```

这是 Unix 哲学的体现：每个工具只做一件事，组合起来完成复杂任务。设计 CLI 工具时，养成**输出结构化内容**（纯文本、JSON）的习惯，工具之间的协作会顺滑很多。

3.  ### **AI Agent 的执行外壳**

大模型本身不能直接操作文件、调用服务、执行计算，它需要借助外部工具。CLI 工具天然适合充当这个角色。

一个 CLI 工具改造成 Agent 工具的成本很低，核心逻辑不变，只需要加一层包装：

```
# 把核心逻辑提取成纯函数
def filter_logs(file: str, level: str = "ERROR", tail: int = 0) -> str:
    path = Path(file)
    if not path.exists():
        return f"文件不存在：{file}"
    
    lines = path.read_text(encoding="utf-8").splitlines()
    matched = [l for l in lines if level.upper() in l]
    if tail:
        matched = matched[-tail:]
    
    return "\n".join(matched) if matched else f"未找到 {level} 级别日志"


# 包装成 LangChain Tool（示意）
from langchain.tools import StructuredTool

log_tool = StructuredTool.from_function(
    func=filter_logs,
    name="log_filter",
    description="从日志文件中提取指定级别的日志记录，支持按条数截取"
)
```

Agent 拿到这个工具后，面对"帮我看看今天的日志有没有报错"这类问题，会自动决定调用它、传入正确的参数、把结果整合进回答里。

关键在于：**把核心逻辑写成函数，** **CLI** **是一种调用方式，Agent 是另一种调用方式**，二者共用同一套实现。一开始就把逻辑和接口分开写，后面扩展的代价几乎为零。

## 五、AI 时代的 CLI 设计

知道 CLI 可以对接 Agent 还不够，真正落地时会遇到一个问题：**给人用的 CLI 和给 Agent 用的 CLI，设计思路其实不一样。**

人读报错信息，理解语境，可以判断下一步怎么做。Agent 读输出，需要的是可解析的结构、明确的状态码、以及没有歧义的返回格式。写给 Agent 调用的工具，有几个地方值得专门设计。

1.  ### 输出要机器可读

人眼友好的输出往往对机器不友好。加一个 `--json` 开关，让工具在需要时输出结构化内容：

```
import json

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("file")
    parser.add_argument("--level", default="ERROR")
    parser.add_argument("--tail", type=int, default=0)
    parser.add_argument("--json", action="store_true", dest="json_output",
                        help="以 JSON 格式输出结果")
    args = parser.parse_args()

    path = Path(args.file)
    lines = path.read_text(encoding="utf-8").splitlines()
    matched = [l for l in lines if args.level.upper() in l]
    if args.tail:
        matched = matched[-args.tail:]

    if args.json_output:
        print(json.dumps({
            "file": args.file,
            "level": args.level,
            "count": len(matched),
            "records": matched
        }, ensure_ascii=False))
    else:
        for line in matched:
            print(line)
```

Agent 调用时加上 `--json`，拿到的是可以直接解析的结构，不需要从自然语言里提取信息，出错的概率低很多。

2.  ### 用退出码传递执行状态

命令行有一个常被忽略的机制：**退出码（exit code）** 。`sys.exit(0)` 表示成功，非 0 表示失败，具体数值可以自定义含义。

Agent 或自动化脚本通过退出码判断工具是否执行成功，决定要不要重试或走降级逻辑：

```
import sys

def main():
    ...
    path = Path(args.file)
    if not path.exists():
        # 输出给人看的信息走 stderr，不污染 stdout 的结构化输出
        print(f"文件不存在：{args.file}", file=sys.stderr)
        sys.exit(1)  # 非 0 表示失败

    matched = [...]
    if not matched:
        sys.exit(2)  # 2 表示"执行成功但没有结果"，有别于报错

    # 正常输出
    ...
    sys.exit(0)
```

在 shell 脚本或 Agent 的工具调用层，可以用 `$?` 或返回码分支处理不同情况。`$?`：Linux/macOS 里上一条命令的退出码，`-eq 2`：等于 2。

```
# 运行日志过滤脚本 → 检查有没有 ERROR
#  没找到 ERROR → 脚本退出码 = 2 → 显示“日志干净”
#  找到 ERROR   → 脚本退出码 ≠ 2 → 不显示任何内容
python log_filter.py app.log --level ERROR
if [ $? -eq 2 ]; then
    echo "日志干净，无错误"
fi
```

3.  ### 把描述写清楚，方便模型理解

Agent 决定要不要调用某个工具，依据是工具的描述。描述写得含糊，模型就容易用错或不用。一份好的工具描述应该包含三个要素：**做什么、接收什么、返回什么**。

```
log_tool = StructuredTool.from_function(
    func=filter_logs,
    name="log_filter",
    description=(
        "从本地日志文件中按级别过滤日志记录。"
        "接收文件路径、日志级别（DEBUG/INFO/WARNING/ERROR/CRITICAL）和可选的条数限制。"
        "返回匹配的日志行列表，无结果时返回空列表。"
        "适用于排查错误、统计异常频次等场景。"
    )
)
```

这三点加在一起，工具对 Agent 来说就是一个职责清晰、行为可预期的模块，而不是一个黑盒。

4.  ### 保持幂等性

Agent 执行任务时可能因为网络波动、超时等原因重试同一个工具调用。如果工具有副作用（写文件、发请求、修改数据），重复执行可能造成问题。

设计工具时尽量保证**相同输入多次执行结果一致**，有副作用的操作加上检查：

```
def write_report(output_path: str, content: str, overwrite: bool = False) -> str:
    path = Path(output_path)
    if path.exists() and not overwrite:
        return f"文件已存在，跳过写入：{output_path}（传入 overwrite=True 强制覆盖）"
    path.write_text(content, encoding="utf-8")
    return f"写入成功：{output_path}"
```

对于读操作，天然幂等，不需要额外处理。对于写操作，加一个 `--overwrite` 开关比静默覆盖要好，Agent 和人都能感知到这个行为。

## 六、总结

`sys.argv` 够用，但只适合参数极少的临时脚本。只要工具需要给别人用，或者参数超过两个，`argparse` 就该上了。更重要的是对 CLI 工具定位的理解：写好了可以接入定时任务、串联成流水线、也可以直接成为 AI Agent 的能力模块。给 Agent 用的工具，在输出格式、退出码、描述和幂等性上多花一点心思，后续接入时会省很多调试时间。

这条从脚本到 Agent 工具的路径，技术门槛并不高，差别主要在设计意识上——**一开始就把逻辑和接口分开，后面想怎么接都方便。**