---
name: python-base
description: >-
  现代 Python 核心开发规范技能。以 Python 3.10+ 为演进基调，具备完备的环境版本自适应决策机制。
  涵盖类型系统自适应 (3.10+ 原生联合类型 vs 3.8/3.9 __future__ 降级)、默认可变参数避坑铁律、
  高效数据模型 (@dataclass slots)、生成器流式防 OOM、异常链保留及 Pythonic 惯用法。
---

# 现代 Python 核心语言规范技能 (Modern Idiomatic Python Skill)

## 概述 (Overview)

本技能定义了在编写**通用 Python 基础模块、纯数据结构、算法逻辑、公共工具库或领域实体对象**时的核心规范与 Pythonic 惯用法。深刻践行 **“现代基调先行，环境自适应决策”** 的务实工程哲学。

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
| **结构模式匹配** | `match...case` | 降级为清晰字典映射分发（Dispatch Table）或结构化 `if-elif-else` | 3.10 之前的版本不支持 `match` 关键字。 |
| **高效数据模型** | `@dataclass(slots=True)` | 3.10+ 支持 `slots=True`；3.8/3.9 降级为普通 `@dataclass` | 3.8/3.9 不支持 `slots=True` 参数。 |

---

# 2. Pythonic 核心铁律与避坑防线 (Core Idioms & Anti-Patterns)

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

# 3. 专属排障与静态诊断武器库 (Troubleshooting & Tooling)

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
  # 执行被怀疑有内存泄漏的逻辑...
  snapshot = tracemalloc.take_snapshot()
  for stat in snapshot.statistics("lineno")[:5]:
      print(stat)
  ```

---

# 4. Python 基础开发 Checklist

- [ ] 是否已检查项目版本声明（`pyproject.toml` 等）并自适应匹配对应语法？
- [ ] 若在 Python < 3.10 环境使用现代类型语法，是否在顶部显式声明了 `from __future__ import annotations`？
- [ ] 是否存在函数默认可变参数（`[]` 或 `{}`）的致命隐患？
- [ ] 抛出包装异常时是否添加了 `from err` 保留因果链？
- [ ] 海量数据计算是否采用了生成器流式处理？
