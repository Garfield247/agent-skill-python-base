---
name: python-base
description: >-
  Python 3.10+ 现代基础语言核心规范、PEP 604 强类型系统、Pythonic 惯用法与数据模型技能。
  涵盖原生类型提示 (X | Y, list[T])、可变默认参数避坑、dataclass(slots=True) 高效建模、
  生成器流式防 OOM、异常链保留 (raise ... from err) 及 match-case 模式匹配。
---

# Python 3.10+ 现代基础语言规范技能 (Modern Python Mastery Skill)

## 概述 (Overview)

本技能定义了在编写**通用 Python 基础模块、纯数据结构、算法逻辑、公共工具库或领域实体对象**时的核心代码规范与现代 Pythonic 惯用法。深度吸纳 **PEP 8 / PEP 20 (Zen of Python)**、**PEP 604 现代类型系统**以及业界超快速工具链（Ruff/Mypy）的工程实践。

---

# 1. 现代类型系统规范 (Type Hints & PEP 604)

全面拥抱 Python 3.10+ 语法，严禁在现代代码中残留 3.8 以前的陈旧类型导入：

| 场景分类 | 陈旧写法 (禁止) | 现代规范 (强制) |
| :--- | :--- | :--- |
| **可空/联合类型** | `from typing import Optional, Union`<br/>`Optional[str]`, `Union[int, str]` | `str | None`<br/>`int | str` (PEP 604 原生语法) |
| **通用集合容器** | `from typing import List, Dict, Set`<br/>`List[str]`, `Dict[str, Any]` | 原生容器：`list[str]`, `dict[str, Any]`, `set[int]` |
| **可调用对象** | `Callable[[int], str]` | 导入 `from collections.abc import Callable, Sequence, Mapping, Iterator` |
| **鸭子类型抽象** | 强制继承抽象基类 ABC | 使用 `typing.Protocol` 声明结构化子类型接口 |

### Protocol 静态结构接口范例
```python
from typing import Protocol

class Serializable(Protocol):
    """定义鸭子类型契约，无需显式继承"""
    def to_dict(self) -> dict[str, any]: ...

def dump_payload(obj: Serializable) -> dict[str, any]:
    return obj.to_dict()
```

---

# 2. Pythonic 核心铁律与避坑防线 (Core Idioms & Anti-Patterns)

### 🚨 铁律一：默认可变参数死穴 (Mutable Default Argument - 绝对红线)
- 函数默认参数在模块加载时仅求值一次，**绝对禁止使用可变对象（列表、字典、集合）作为默认值**：
  ```python
  # ❌ 致命错误：所有调用者将共享同一个 items 列表，导致脏数据穿透
  def append_item(val: str, items: list[str] = []) -> list[str]:
      items.append(val)
      return items

  # ✅ 唯一正解：使用 None 并在函数体内动态初始化
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

# 3. 现代面向对象与高效数据模型 (Modern Data Modeling)

优先使用标准库 `@dataclass` 替代繁琐的手写 `__init__`，并开启 `slots` 节省 40% 内存：

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass(slots=True, frozen=True)
class UserProfile:
    """现代只读高效数据类
    slots=True: 禁用 __dict__，极大降低内存开销并加速属性访问
    frozen=True: 实例只读不可变，天然线程安全并支持作为字典 Key (自动生成 __hash__)
    """
    user_id: int
    username: str
    tags: tuple[str, ...] = field(default_factory=tuple)
    created_at: datetime = field(default_factory=datetime.now)
```

---

# 4. 现代模式匹配 (Pattern Matching - Python 3.10+)

使用 `match...case` 取代难以维护的多层 `if-elif-else`，实现清晰的解构与分发：

```python
def handle_event(event: dict[str, any]) -> str:
    match event:
        case {"type": "user_login", "user_id": int(uid), "ip": str(ip)}:
            return f"用户登录: {uid} (IP: {ip})"
        case {"type": "order_paid", "amount": float(amt)} if amt > 10000:
            return f"大额订单支付触发风控审核: {amt}"
        case {"type": "order_paid", "amount": float(amt)}:
            return f"常规订单支付: {amt}"
        case _:
            raise ValueError(f"未知事件类型: {event}")
```

---

# 5. 专属排障与静态诊断武器库 (Troubleshooting & Tooling)

- **极速代码检查与格式化 (Ruff - 替代 flake8/black/isort)**：
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

# 6. Python 基础开发 Checklist

- [ ] 类型注解是否全部使用 Python 3.10+ 原生语法（`str | None`，原生 `list/dict`）？
- [ ] 是否存在函数默认可变参数（`[]` 或 `{}`）的致命隐患？
- [ ] 数据模型是否开启了 `@dataclass(slots=True)` 提升内存与访问性能？
- [ ] 抛出包装异常时是否添加了 `from err` 保留异常链？
- [ ] 海量数据计算是否采用了生成器流式处理？
