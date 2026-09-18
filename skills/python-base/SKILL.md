---
name: python-base
description: >-
  现代 Python 核心开发规范技能。以 Python 3.10+ 为演进基调，具备完备的环境版本自适应决策机制。
  涵盖现代常量与不可变数据规范 (typing.Final 静态保护、StrEnum/IntEnum、@dataclass 冻结配置)、
  时间格式化模板常量规范 (收敛 strftime/strptime、时区感知)、类型系统自适应 (3.10+ 原生联合类型 vs 3.8/3.9 __future__ 降级)、
  默认可变参数避坑铁律、生成器流式防 OOM、异常链保留及 Pythonic 惯用法。
---

# 现代 Python 核心语言规范技能 (Modern Idiomatic Python Skill)

## 概述 (Overview)

本技能定义了在编写**通用 Python 基础模块、纯数据结构、算法逻辑、公共工具库或领域实体对象**时的核心规范与 Pythonic 惯用法。深刻践行 **“现代基调先行，环境自适应决策”** 的务实工程哲学，吸收现代类型系统与不可变常量设计实践。

---

# 1. 架构基调与版本自适应决策 (Version Baseline & Adaptive Matrix)

### 1.1 演进基调 (North Star Baseline)
在新立项或无旧版本历史包袱时，**统一以 Python 3.10+ 作为现代设计基调**，充分利用原生强类型系统（PEP 604）、模式匹配与语言级性能优化。

### 1.2 环境自适应感知与降级决策表 (Adaptive Sniffing & Fallback)
在面对既有存量项目时，**第一原则是尊重当前项目的真实运行环境**（优先探测 `pyproject.toml`、`runtime.txt`、`.python-version`、`Dockerfile` 或环境变量）：

| 语法/特性维度 | 推荐现代基调 (Python 3.10+) | 低版本优雅降级方案 (Python 3.8 / 3.9) | 核心考量 / 避坑红线 |
| :--- | :--- | :--- | :--- |
| **联合类型 (Union)** | `str | None`<br/>`int | str` | **首选**：文件顶部引入 `from __future__ import annotations`<br/>**备选**：`from typing import Optional, Union`<br/>使用 `Optional[str]`, `Union[int, str]` | 严禁在 3.8/3.9 环境未导入 `__future__` 时直接使用 `|` 运算，避免 `TypeError` 崩溃。 |
| **通用集合类型** | 原生：`list[str]`, `dict[str, Any]` | 3.9 支持原生；3.8 需使用 `from typing import List, Dict` | 避免陈旧 `typing` 容器在现代环境过度导入。 |
| **原生字符串枚举** | Python 3.11+ 原生 `enum.StrEnum` | 3.8 ~ 3.10 优雅降级：`class MyEnum(str, Enum):` | 严禁在业务中直接乱传散装裸字符串。 |
| **结构模式匹配** | `match...case` | 降级为清晰字典映射分发（Dispatch Table）或结构化 `if-elif-else` | 3.10 之前的版本不支持 `match` 关键字。 |
| **高效数据模型** | `@dataclass(slots=True)` | 3.10+ 支持 `slots=True`；3.8/3.9 降级为普通 `@dataclass` | 3.8/3.9 不支持 `slots=True` 参数。 |

---

# 2. 现代常量与不可变数据工程规范 (Constants & Immutability)

虽然 Python 语言底层没有类似 `const` 的绝对禁止重写关键字，但现代工业级工程必须通过**命名规范、静态类型检查器与不可变容器**构建坚固的防线，彻底杜绝散装魔法值：

### 2.1 命名约定与 `typing.Final` 静态保护 (Python 3.8+)
- **命名规范**：模块级常量必须全部使用大写字母加下划线命名（`UPPER_SNAKE_CASE`）；
- **类型保护**：强制使用 `typing.Final` 进行类型注解。Mypy、Pyright 等现代静态类型检查器以及 IDE 会在检测到二次重赋值时立即报错拦截：
  ```python
  from typing import Final

  # ✅ 声明不可变常量
  DEFAULT_PAGE_SIZE: Final[int] = 20
  MAX_TIMEOUT_SECONDS: Final[float] = 30.0
  API_VERSION_PREFIX: Final[str] = "/api/v1"
  ```

### 2.2 严格使用枚举类替代魔法字符串/数字 (Enum Standards)
- **业务状态与分类**：严禁在函数入参或条件判断中裸写散装字符串（如 `if status == "active"`）；
- **标准写法**：
  ```python
  from enum import Enum, IntEnum, unique
  import sys

  # 兼容 Python 3.11+ 原生 StrEnum 与低版本优雅降级
  if sys.version_info >= (3, 11):
      from enum import StrEnum
  else:
      class StrEnum(str, Enum):
          pass

  @unique
  class OrderStatus(StrEnum):
      # 订单生命周期状态枚举
      PENDING = "pending"
      PAID = "paid"
      FULFILLING = "fulfilling"
      COMPLETED = "completed"
      CANCELLED = "cancelled"

  @unique
  class HttpErrorCode(IntEnum):
      # 标准业务错误码
      INVALID_ARGUMENT = 40001
      UNAUTHORIZED = 40101
      RESOURCE_NOT_FOUND = 40401
  ```

### 2.3 复合配置常量的冻结保护 (`@dataclass(frozen=True)`)
- 对于由多个参数组成的不可变配置项、常量结构体，使用 `@dataclass(frozen=True)`：
  ```python
  from dataclasses import dataclass
  from typing import Final

  @dataclass(frozen=True, slots=True)
  class DatabaseConfig:
      host: str
      port: int
      db_name: str

  # ✅ 全局只读配置常量实例（任何修改尝试都会抛出 FrozenInstanceError）
  DEFAULT_DB_CONFIG: Final[DatabaseConfig] = DatabaseConfig(
      host="127.0.0.1", port=3306, db_name="primary_db"
  )
  ```

---

# 3. 时间格式化与时间常量规范 (Time Formatting & Constants)

在 Python 中处理 `datetime` 时，各业务模块四处散落裸写的 `"%Y-%m-%d %H:%M:%S"` 是造成格式不一致和代码异味的主要诱因。

### 3.1 时间格式化模板强制常量化
- 严禁在调用 `dt.strftime(...)` 或 `datetime.strptime(...)` 时手写魔法字符串；
- 必须统一从公共工具包（如 `common/time_utils.py` 或 `constants.py`）中导入时间模板常量：

```python
from typing import Final

# ✅ 统一收敛的标准时间格式模板常量
DATETIME_FORMAT: Final[str] = "%Y-%m-%d %H:%M:%S"
DATE_FORMAT: Final[str] = "%Y-%m-%d"
TIME_FORMAT: Final[str] = "%H:%M:%S"
COMPACT_DATETIME_FORMAT: Final[str] = "%Y%m%d%H%M%S"
COMPACT_DATE_FORMAT: Final[str] = "%Y%m%d"
ISO8601_FORMAT: Final[str] = "%Y-%m-%dT%H:%M:%S%z"
LOG_TIMESTAMP_FORMAT: Final[str] = "%Y-%m-%d %H:%M:%S.%f"
```

### 3.2 现代 ISO 8601 原生解析优先
- 对于标准 ISO 8601 格式字符串（如 API 返回的时间戳），优先使用 Python 3.7+ 原生支持的 `datetime.fromisoformat()`，性能更高且无需手动拼装正则模板：
  ```python
  from datetime import datetime
  # ✅ 原生高效解析
  dt = datetime.fromisoformat("2026-09-18T13:30:00+08:00")
  ```

### 3.3 时区意识与防坑原则 (Timezone-Awareness)
- **警惕 Naive Datetime**：严禁在分布式系统中混用无时区信息的时间对象（Naive）与有时区信息的时间对象（Aware），比较时会直接引发 `TypeError`；
- **标准工业级实践**：
  - 存储与传输统一采用 **UTC 时间戳** 或 **显式携带时区偏移的 ISO 字符串**；
  - 仅在呈现给最终用户时，才转换为用户本地时区（如 `ZoneInfo("Asia/Shanghai")`）。

---

# 4. Pythonic 核心铁律与避坑防线 (Core Idioms & Anti-Patterns)

无论项目处于哪种 Python 版本，以下三项铁律全版本通用且强制遵循：

### 🚨 铁律一：默认可变参数死穴 (Mutable Default Argument - 绝对红线)
- 函数默认参数在模块加载时仅求值一次，**绝对禁止使用可变对象（列表、字典、集合）作为默认值**：
  ```python
  # ❌ 致命错误：所有调用者共享同一个 items 列表，造成跨请求脏数据污染
  def append_item(val: str, items: list[str] = []) -> list[str]:
      items.append(val)
      return items

  # ✅ 唯一正解：全版本通用安全模式（使用 None 并在函数体内动态初始化）
  def append_item(val: str, items: list[str] | None = None) -> list[str]:
      target = items if items is not None else []
      target.append(val)
      return target
  ```

### 🚨 铁律二：生成器流式防 OOM (Generators over Full Lists)
- 在处理未知规模的数据集、大型日志行、数据库游标或遍历目录时，**严禁使用全量列表推导式 `[x for x in ...]` 一次性将海量数据载入内存**；
- 必须使用生成器表达式 `(x for x in ...)` 或 `yield` 迭代器按需消费，维持恒定 $O(1)$ 内存复杂度。

### 🚨 铁律三：异常链保留原则 (Exception Chaining)
- 捕获底层错误并转换为高层业务异常时，**必须使用 `raise ... from err` 显式维持原始调用栈**；
- 严禁裸 `except:`，严禁静默 `pass`：
  ```python
  # ✅ 正确：保留底层原始异常堆栈，便于定位根因
  try:
      raw_val = int(user_input)
  except ValueError as e:
      raise InvalidArgumentError(f"输入非有效整数: {user_input}") from e
  ```

---

# 5. 专属排障与静态诊断武器库 (Troubleshooting & Tooling)

- **极速代码检查与格式化 (Ruff)**：
  ```bash
  ruff check . --fix
  ruff format .
  ```
- **严格类型静态检查 (Mypy)**：
  ```bash
  mypy --strict .
  ```
- **内存泄漏与大对象分配分析 (Tracemalloc)**：
  ```python
  import tracemalloc
  tracemalloc.start()
  snapshot = tracemalloc.take_snapshot()
  for stat in snapshot.statistics("lineno")[:5]:
      print(stat)
  ```

---

# 6. Python 基础开发审查 Checklist

- [ ] **版本自适应**：是否已检查 `pyproject.toml` 并自适应匹配当前运行环境？
- [ ] **常量与不可变数据**：
  - [ ] 模块级常量是否全部使用 `UPPER_SNAKE_CASE` 并注解了 `Final[...]`？
  - [ ] 业务状态分类是否使用了 `StrEnum` / `IntEnum` 替代散装魔法字符串？
  - [ ] 复合配置常量是否使用 `@dataclass(frozen=True)` 进行了冻结保护？
- [ ] **时间格式化规范**：
  - [ ] 时间模板是否统一引用了 `DATETIME_FORMAT` 等常量，消除了内联魔法字符串？
  - [ ] 是否规避了 Naive 与 Aware 时区的非法混用？
- [ ] **函数默认参数**：是否存在可变对象（`[]` 或 `{}`）的默认值隐患？
- [ ] **异常与因果链**：抛出包装业务异常时是否添加了 `from err`？
- [ ] **流式与内存控制**：未知规模数据处理是否采用了生成器流式处理？
