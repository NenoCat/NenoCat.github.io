---
title: Python中的模块与包管理 
date: 2026-07-18 23:30:01
categories:
  - 技术
tags:
  - Python
---

在 Python 中，模块不仅是代码组织的基础，也是实现复用的关键。从简单的 import，到合理的目录结构，再到工具模块抽象，都是在为一个目标服务：让代码更清晰、更容易维护。

* * *

## 一、import 的基本用法

Python包最常见的导入方式有三种：

```
import math

print(math.sqrt(16))
```

```
from math import sqrt

print(sqrt(16))
```

```
import math as m

print(m.sqrt(16))
```

区别在于命名空间的引入方式。`import xxx` 会保留模块名作为前缀，而 `from xxx import yyy` 会直接导入具体对象。

> 需要注意避免命名冲突，例如：
>
> ```
> from datetime import datetime
> ```
>
> 如果变量名也叫 `datetime`，就容易覆盖原对象。

* * *

## 二、文件即模块

Python 中，每一个 `.py` 文件本质上就是一个模块。

假设有一个文件 `utils.py`：

```
def add(a, b):
    return a + b
```

在另一个文件中可以直接导入：

```
import utils

print(utils.add(1, 2))
```

运行时，Python 会在 `sys.path` 中查找这个模块路径。

* * *

当项目变大时，通常会使用目录来组织模块：

```
project/
│
├── main.py
└── utils/
    ├── __init__.py
    ├── math_utils.py
    └── file_utils.py
```

**`init.py`** **的作用是让该目录成为一个“包”。同时可以在其中定义对外暴露的接口：**

```
# utils/__init__.py
from .math_utils import add
```

这样在外部可以直接：

```
from utils import add
```

而不需要关心具体文件位置。

* * *

## 三、如何写可复用工具模块

一个常见问题是：写的代码只能在当前文件用，无法复用。

改进方式是将通用逻辑抽离成模块，例如：

```
# utils/logger.py
def log(message):
    print(f"[LOG] {message}")
```

在其他文件中复用：

```
from utils.logger import log

log("程序启动")
```

建议遵循几点：

-   **函数职责单一（一个函数只做一件事）**
-   **不依赖具体业务（避免写死路径或参数）**
-   **提供清晰接口（函数名表达用途）**

* * *

## 四、Python 包管理

1.  ### 常用包管理工具

#### （1）pip：最基础的包管理工具

Python 官方推荐的包管理工具，用于安装第三方库。

```
pip install requests
```

指定版本：

```
pip install requests==2.31.0
```

导出依赖：

```
pip freeze > requirements.txt
```

安装依赖：

```
pip install -r requirements.txt
```

这是最常见的一套流程，适用于大多数项目。

* * *

#### （2）venv：虚拟环境

不同项目可能依赖不同版本的库，如果混在一起容易出问题。虚拟环境可以做到隔离。

创建虚拟环境：

```
python -m venv venv
```

激活环境（Mac / Linux）：

```
source venv/bin/activate
```

激活环境（Windows）：

```
venv\Scripts\activate
```

激活后安装的包只作用于当前项目。

* * *

#### （3）进阶工具：poetry / pipenv

当项目规模变大后，`pip + requirements.txt` 会逐渐暴露一些问题：依赖版本不稳定（不同机器装出来不一致） 、需要手动维护 requirements.txt 容易混乱 、缺少“开发依赖 / 生产依赖”的区分 等问题。这时候，可以考虑使用更完整的依赖管理工具，比如 Poetry 和 Pipenv。

这些工具在 pip 基础上做了封装，提供更完整的依赖管理能力，例如：

-   自动解决依赖冲突
-   统一管理项目配置（`pyproject.toml`）

如果是中大型项目，可以考虑使用。

* * *

#### （4） UV：越来越常见的包管理工具

**uv** 是由 Astral 推出的工具，目标是提供更快、更统一的 Python 依赖管理体验。如果你觉得 pip + venv 这一套流程比较繁琐，可以了解一下 uv。它的定位可以理解为：**用一个工具，同时替代 pip、venv、pip-tools 等。**

**相比传统方案，** **UV** **安装速度明显更快（** **Rust** **实现）、自动管理** **虚拟环境** **（不用手动 venv）、依赖解析更稳定（类似 poetry，但更轻量）。**

（1）安装 uv

```
pip install uv
```

或官方推荐方式（更快）：

```
curl -Ls https://astral.sh/uv/install.sh | sh
```

（2）创建项目 + 虚拟环境

```
uv init
```

会自动生成：

```
pyproject.toml
```

并隐式创建虚拟环境（无需手动 `venv`）。

（3）安装依赖

```
uv add requests
```

指定版本：

```
uv add requests==2.31.0
```

它会自动更新 `pyproject.toml` 和锁文件。

（4）运行代码

```
uv run python main.py
```

不用手动激活虚拟环境。

（5）导出依赖（兼容 pip）

```
uv export > requirements.txt
```

* * *

2.  ### requirements.txt 的实际作用

在团队开发或部署时，通常不会手动一个个安装库，而是通过依赖文件统一管理：

```
flask==2.3.2
requests==2.31.0
pandas==2.0.3
```

新环境只需要：

```
pip install -r requirements.txt
```

就能恢复完整运行环境。

* * *

3.  ### 如何把自己的代码变成“包”

前面讲的是“用别人的包”，这里讲“让自己的代码可被别人用”。

一个简单的包结构：

```
my_package/
├── my_package/
│   ├── __init__.py
│   └── utils.py
├── setup.py
```

示例 `setup.py`：

```
from setuptools import setup, find_packages

setup(
    name="my_package",
    version="0.1",
    packages=find_packages(),
)
```

安装本地包：

```
pip install .
```

安装后，只要在你自己的电脑上（还没上传到PyPI，只是安装在了你的电脑），你就可以在任意项目下使用：

```
from my_package.utils import xxx
```

这一步的意义在于：**把“项目代码”变成“可复用库”** 。

## 总结

这一节的核心在于：**模块不仅是代码拆分工具，更是复用的基础单元**。

从简单的 import，到合理的目录结构，再到工具模块抽象，都是在为一个目标服务：让代码更清晰、更容易维护。