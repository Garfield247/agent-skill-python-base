---
name: python-base
description: >-
  现代 Python 核心开发规范技能。以 Python 3.10+ 为演进基调，具备完备的环境版本自适应决策机制。
  涵盖 Python 代码封装六大原则 (防 kwargs 穿透、显式 __all__、防原始类型偏执、上下文管理器资源治理)、
  common/pkg 基础层分层架构标准 (一级分类收紧、二级按需嵌套与方案 B *x 扩展工具包)、
  核心工具模块生产级标杆 (listx/dictx/strx/timex/convx)、时间格式化模板常量与时区感知、
  现代常量与不可变数据规范 (typing.Final、StrEnum/IntEnum、@dataclass 冻结配置)、
  默认可变参数避坑铁律、生成器流式防 OOM、异常链保留及 Pythonic 惯用法。
---

# 现代 Python 核心语言与基础层架构规范 (Modern Idiomatic Python & Base Architecture)

## 概述 (Overview)

本技能定义了在编写 **Python 基础模块、公共工具库、领域实体对象、业务逻辑及微服务基础层** 时的核心规范与 Pythonic 最佳实践。
深刻践行 **“现代基调先行，环境自适应决策；一级分类收紧，工具轻量扩展；强类型防线闭环，拒绝黑盒穿透”** 的务实工业级工程哲学。

---

# 1. 架构基调与版本自适应决策 (Version Baseline & Adaptive Matrix)

### 1.1 演进基调 (North Star Baseline)
在新立项或无旧版本历史包袱时，**统一以 Python 3.10+ 作为现代设计基调**，充分利用原生强类型联合操作符（PEP 604）、模式匹配与语言级性能优化。

### 1.2 环境自适应感知与降级决策表 (Adaptive Sniffing & Fallback)
在面对既有存量项目时，**第一原则是自适应当前项目的真实运行宿主版本**（检查 `pyproject.toml`、`runtime.txt`、`.python-version`、`Dockerfile` 等）：

| 语法/特性维度 | 推荐现代基调 (Python 3.10+) | 低版本优雅降级方案 (Python 3.8 / 3.9) | 核心考量 / 避坑红线 |
| :--- | :--- | :--- | :--- |
| **联合类型 (Union)** | `str | None`<br/>`int | str` | **首选**：文件顶部引入 `from __future__ import annotations`<br/>**备选**：`from typing import Optional, Union` | 严禁在 3.8/3.9 环境未导入 `__future__` 时直接使用 `|` 运算，避免 `TypeError` 崩溃。 |
| **通用集合类型** | 原生：`list[str]`, `dict[str, Any]` | 3.9 支持原生；3.8 需使用 `from typing import List, Dict` | 避免陈旧 `typing` 容器在现代环境过度导入。 |
| **原生字符串枚举** | Python 3.11+ 原生 `enum.StrEnum` | 3.8 ~ 3.10 优雅降级：`class MyEnum(str, Enum):` | 严禁在业务中直接乱传散装裸字符串。 |
| **结构模式匹配** | `match...case` | 降级为清晰字典映射分发（Dispatch Table）或结构化 `if-elif-else` | 3.10 之前的版本不支持 `match` 关键字。 |
| **高效数据模型** | `@dataclass(slots=True)` | 3.10+ 支持 `slots=True`；3.8/3.9 降级为普通 `@dataclass` | 3.8/3.9 不支持 `slots=True` 参数。 |

---

# 2. Python 代码封装六大核心原则 (Pythonic Encapsulation Principles)

代码封装质量直接决定系统的健壮度与可维护性。在封装函数、模块与类时，必须严格遵守以下六大原则：

### 2.1 原则一：显式优于隐式，严禁滥用 `**kwargs` 穿透黑盒
- **反模式**：在业务函数、服务层方法或通用工具中随意声明 `def process_data(*args, **kwargs): other_func(**kwargs)`。这会导致 IDE 失去类型补全、无法静态分析必填项，重构时极易引发致命的 `KeyError` 或 `TypeError`。
- **规范**：
  1. 公共对外 API 必须使用显式命名参数与类型注解；
  2. 善用 **仅限关键字参数（Keyword-Only Arguments）** 语法 `*`，强制调用方显式传参以提升调用处的可读性；
  3. 参数超过 4 个时，应封装为强类型参数模型（Pydantic `BaseModel` 或 `@dataclass(slots=True)`）。
  ```python
  # ❌ 反模式：无语义黑盒穿透，调用者完全不知道传什么
  def search_users(**kwargs):
      return repo.query(**kwargs)

  # ✅ 规范写法 1：仅限关键字参数，强制显式调用
  def search_users(*, keyword: str, page: int = 1, page_size: int = 20) -> list[User]:
      ...

  # ✅ 规范写法 2：复合复杂参数封装为强类型模型
  from pydantic import BaseModel, Field

  class UserSearchQuery(BaseModel):
      keyword: str = Field(..., min_length=1)
      page: int = Field(default=1, ge=1)
      page_size: int = Field(default=20, le=100)

  def search_users(query: UserSearchQuery) -> list[User]:
      ...
  ```

### 2.2 原则二：最小知识原则 (Law of Demeter) 与显式对外暴露 (`__all__`)
- **规范**：
  1. 包级（`__init__.py`）与模块级必须明确公开接口与私有实现的边界；
  2. 模块级对外导出的符号必须显式定义 `__all__ = ["ExportA", "ExportB"]`，杜绝 `from module import *` 产生命名空间污染；
  3. 内部辅助函数、模块级单例实例、私有常量必须使用单前导下划线 `_`（如 `_helper()`、`_CACHE_INSTANCE`），明确告知外部调用者不可强依赖。

### 2.3 原则三：防原始类型偏执 (Avoid Primitive Obsession) 与强类型防线
- **规范**：
  1. 杜绝在业务代码中滥用裸 `dict`、裸 `tuple` 或裸 `str` 充当业务实体；
  2. 实体与状态：使用 `StrEnum`/`IntEnum` 约束业务状态；
  3. 领域 ID 区分：使用 `typing.NewType`（如 `UserId = NewType("UserId", int)`、`OrderId = NewType("OrderId", int)`），防止不同业务维度的 ID 混传；
  4. 只读复合值对象：使用 `@dataclass(frozen=True, slots=True)`；可变业务模型：使用 Pydantic v2 `BaseModel`。

### 2.4 原则四：资源生命周期必须使用上下文管理器 (`with` / `async with`)
- **规范**：
  1. 凡涉及资源申请与释放（文件句柄、网络会话、数据库连接、分布式锁、临时全局配置置换）的代码封装，**绝对禁止要求外部调用者显式手动调用 `.close()` 或 `.release()`**；
  2. 必须实现上下文管理器（`__enter__/__exit__` 或 `__aenter__/__aexit__`），或使用 `@contextmanager` / `@asynccontextmanager` 装饰器封装，确保即便发生异常也能 100% 自动安全回收。
  ```python
  from contextlib import contextmanager
  from typing import Iterator

  @contextmanager
  def temporary_setting(key: str, temp_value: Any) -> Iterator[None]:
      original_value = config.get(key)
      config.set(key, temp_value)
      try:
          yield
      finally:
          # 保证无论内部是否抛出异常，都会被安全恢复
          config.set(key, original_value)
  ```

### 2.5 原则五：内部自闭环并发/协程安全
- **规范**：
  1. 凡涉及共享状态维护的类（单例、本地缓存、流水号生成器），其线程安全（`threading.Lock`）或异步协程安全（`asyncio.Lock`）**必须在内部方法内部自闭环**；
  2. 坚决禁止将锁对象暴露给调用者在外部加锁释放，杜绝死锁与漏锁风险。

### 2.6 原则六：拒绝透传空壳封装
- **规范**：
  1. 严禁编写仅仅只是简单原样调用标准库或第三方库、没有任何校验、错误转换、指标监控或业务增强的“洋葱皮空壳代码”；
  2. 每一个工具函数必须提供明确的生产级价值（如提供安全的异常捕获降级、严格的时区对齐、保序去重或敏感脱敏）。

---

# 3. 基础公共层分层架构标准 (`common/`：一级收紧，二级按需嵌套)

在 Python 工程体系（FastAPI、Django、微服务、离线自动化脚本）中，公共基础层统一建议置于根目录或核心包下的 `common/`（或 `pkg/`）中。

### 3.1 一级目录按分类收紧原则 (收紧为 6~7 个核心基础大类)
严禁在基础公共层根目录下随意自建零碎的一级散乱模块，一级目录必须高度收敛：

```
common/                       # 基础公共层根目录
├── __init__.py
├── enums/                    # [核心 1] 全局跨域业务枚举定义中心
│   ├── __init__.py
│   ├── order_enum.py
│   └── user_enum.py
├── types/                    # [核心 2] 跨域纯类型契约、泛型与值对象 (Pydantic/Dataclass/TypedDict)
│   ├── __init__.py
│   ├── pagination.py
│   └── result.py
├── exceptions/               # [核心 3] 全局统一业务异常体系与错误码
│   ├── __init__.py
│   ├── base.py               # BaseBusinessException 基类
│   └── error_codes.py        # 业务错误码定义
├── ctxdata/                  # [核心 4] 全异步/多线程安全上下文 (基于 contextvars.ContextVar)
│   ├── __init__.py
│   └── trace.py              # RequestID / TraceID / CurrentUser 上下文提取与透传
├── utils/                    # [核心 5] 通用无状态纯计算工具集 (方案 B: *x 扩展模块)
│   ├── __init__.py
│   ├── listx.py              # 切片/列表/迭代器高级扩展
│   ├── dictx.py              # 字典/深层结构高级扩展
│   ├── strx.py               # 字符串安全截断与敏感信息脱敏
│   ├── timex.py              # 统一时区感知与自然日对齐
│   └── convx.py              # 安全类型转换与兜底降级
├── storage/                  # [核心 6] 外部基础持久化与缓存客户端连接封装
│   ├── __init__.py
│   ├── redis_client.py
│   └── mysql_client.py
└── middlewares/              # [核心 7] Web/RPC 通用拦截器与中间件
    ├── __init__.py
    └── request_log.py
```

### 3.2 二级按需拆分与命名铁律 (方案 B: `*x` 扩展风格)
1. **命名红线**：
   - 坚决杜绝多单词无间隔挤压拼接（**严禁** `maputil.py`、`stringutil.py`、`listutil.py`，违背命名规范）；
   - 统一采用 **方案 B：`*x` 扩展命名法**（`listx.py`、`dictx.py`、`strx.py`、`timex.py`、`convx.py`）：
     - 短促精炼，杜绝挤压拼接；
     - 彻底避免与 Python 标准库同名模块冲突（避免与 `string`, `datetime`, `itertools` 命名遮蔽）；
     - 业务调用语义自然直观：`from common.utils import listx, dictx, timex`。
2. **二级拆分决策矩阵**：
   - **单文件独立模块**：代码行数小于 500 行的工具集，以单个 `.py` 文件承载（如 `common/utils/listx.py`）；
   - **升级为子包目录**：当某个工具领域具备多个复杂实现（如 `crypto/` 包含 `aes.py`, `rsa.py`, `sm4.py`），升格为子包目录，并必须提供 `__init__.py` 显式维护 `__all__`。

---

# 4. 核心工具包规范与高频标杆实现 (方案 B: `*x` 风格)

### 4.1 `listx.py`：切片与迭代器扩展 (Lists & Iterables)
```python
# common/utils/listx.py
from collections.abc import Callable, Hashable, Iterable, Iterator
from itertools import islice
from typing import TypeVar

T = TypeVar("T")
K = TypeVar("K", bound=Hashable)

__all__ = ["chunk", "unique", "flatten", "diff", "intersect"]

def chunk(iterable: Iterable[T], size: int) -> Iterator[list[T]]:
    """流式分块迭代器 (防 OOM)，将可迭代对象按指定大小切分"""
    if size <= 0:
        raise ValueError("size 必须大于 0")
    iterator = iter(iterable)
    while batch := list(islice(iterator, size)):
        yield batch

def unique(iterable: Iterable[T], key: Callable[[T], K] | None = None) -> list[T]:
    """保序去重：保持原序列元素初次出现的相对顺序"""
    seen: set[Hashable] = set()
    result: list[T] = []
    for item in iterable:
        val = key(item) if key else item
        if val not in seen:
            seen.add(val)
            result.append(item)
    return result

def flatten(iterable: Iterable[Iterable[T]]) -> list[T]:
    """单层展开嵌套序列"""
    return [item for sublist in iterable for item in sublist]

def diff(a: Iterable[T], b: Iterable[T]) -> list[T]:
    """计算保序差集：存在于 a 中但不存在于 b 中的元素"""
    set_b = set(b)
    return [x for x in a if x not in set_b]

def intersect(a: Iterable[T], b: Iterable[T]) -> list[T]:
    """计算保序交集：同时存在于 a 和 b 中的元素 (以 a 为顺序准绳)"""
    set_b = set(b)
    return [x for x in a if x in set_b]
```

### 4.2 `dictx.py`：字典与嵌套数据扩展 (Dictionaries)
```python
# common/utils/dictx.py
from collections.abc import Callable, Hashable, Iterable, Mapping
from copy import deepcopy
from typing import Any, TypeVar

T = TypeVar("T")
K = TypeVar("K", bound=Hashable)

__all__ = ["get_in", "deep_merge", "pick", "omit", "key_by", "group_by"]

def get_in(d: Mapping[str, Any], path: Iterable[str], default: Any = None) -> Any:
    """安全深层嵌套取值，防止 KeyError 或 TypeError 崩溃
    用法: get_in(user_dict, ["profile", "address", "city"], default="未知")
    """
    curr: Any = d
    for key in path:
        if isinstance(curr, Mapping) and key in curr:
            curr = curr[key]
        else:
            return default
    return curr

def deep_merge(base: Mapping[str, Any], override: Mapping[str, Any]) -> dict[str, Any]:
    """递归深度合并两个字典 (不改变原字典，返回全新合并结果)"""
    result = deepcopy(dict(base))
    for k, v in override.items():
        if k in result and isinstance(result[k], dict) and isinstance(v, Mapping):
            result[k] = deep_merge(result[k], v)
        else:
            result[k] = deepcopy(v)
    return result

def pick(d: Mapping[str, Any], keys: Iterable[str]) -> dict[str, Any]:
    """白名单字段提取"""
    key_set = set(keys)
    return {k: v for k, v in d.items() if k in key_set}

def omit(d: Mapping[str, Any], keys: Iterable[str]) -> dict[str, Any]:
    """黑名单字段剔除"""
    key_set = set(keys)
    return {k: v for k, v in d.items() if k not in key_set}

def key_by(items: Iterable[T], key_func: Callable[[T], K]) -> dict[K, T]:
    """将列表转为以 key_func 计算结果为主键的字典 (后者覆盖前者)"""
    return {key_func(item): item for item in items}

def group_by(items: Iterable[T], key_func: Callable[[T], K]) -> dict[K, list[T]]:
    """将列表元素按 key_func 分组为字典列表"""
    groups: dict[K, list[T]] = {}
    for item in items:
        groups.setdefault(key_func(item), []).append(item)
    return groups
```

### 4.3 `strx.py`：字符串安全截断与敏感脱敏 (Strings & Masking)
```python
# common/utils/strx.py
import re

__all__ = ["safe_truncate", "mask_phone", "mask_id_card", "to_snake_case", "to_camel_case"]

_SNAKE_RE1 = re.compile(r"(.)([A-Z][a-z]+)")
_SNAKE_RE2 = re.compile(r"([a-z0-9])([A-Z])")

def safe_truncate(text: str, max_length: int, suffix: str = "...") -> str:
    """字符安全截断，防止字符截断异常"""
    if not text or max_length <= 0 or len(text) <= max_length:
        return text
    if len(suffix) >= max_length:
        return text[:max_length]
    return text[: max_length - len(suffix)] + suffix

def mask_phone(phone: str) -> str:
    """手机号脱敏 (保留前 3 后 4，中 4 位掩码)，非法手机号安全遮蔽"""
    if not phone or len(phone) < 7:
        return "****"
    if len(phone) == 11:
        return f"{phone[:3]}****{phone[7:]}"
    return f"{phone[:2]}****{phone[-2:]}"

def mask_id_card(id_card: str) -> str:
    """身份证号脱敏 (保留前 3 后 4，中间掩码)"""
    if not id_card or len(id_card) < 8:
        return "******************"
    return f"{id_card[:3]}***********{id_card[-4:]}"

def to_snake_case(name: str) -> str:
    """驼峰转下划线蛇形命名 (PascalCase/camelCase -> snake_case)"""
    sub = _SNAKE_RE1.sub(r"_", name)
    return _SNAKE_RE2.sub(r"_", sub).lower()

def to_camel_case(name: str, pascal: bool = False) -> str:
    """蛇形转驼峰命名 (snake_case -> camelCase 或 PascalCase)"""
    components = name.split("_")
    if not components:
        return name
    if pascal:
        return "".join(x.title() for x in components)
    return components[0].lower() + "".join(x.title() for x in components[1:])
```

### 4.4 `timex.py`：统一时区感知与时间常量 (Time & Timezone)
```python
# common/utils/timex.py
from datetime import datetime, time, timezone
from typing import Final
from zoneinfo import ZoneInfo

__all__ = [
    "DATETIME_FORMAT", "DATE_FORMAT", "TIME_FORMAT", "ISO8601_FORMAT",
    "SHANGHAI_TZ", "UTC_TZ",
    "now_shanghai", "now_utc", "start_of_day", "end_of_day",
    "to_timestamp_ms", "from_timestamp_ms", "format_datetime",
]

# 统一收敛的标准时间格式常量
DATETIME_FORMAT: Final[str] = "%Y-%m-%d %H:%M:%S"
DATE_FORMAT: Final[str] = "%Y-%m-%d"
TIME_FORMAT: Final[str] = "%H:%M:%S"
COMPACT_DATETIME_FORMAT: Final[str] = "%Y%m%d%H%M%S"
COMPACT_DATE_FORMAT: Final[str] = "%Y%m%d"
ISO8601_FORMAT: Final[str] = "%Y-%m-%dT%H:%M:%S%z"

# 统一收敛的时区常量
SHANGHAI_TZ: Final[ZoneInfo] = ZoneInfo("Asia/Shanghai")
UTC_TZ: Final[timezone] = timezone.utc

def now_shanghai() -> datetime:
    """获取当前上海时区的感知时间 (Timezone-Aware)"""
    return datetime.now(SHANGHAI_TZ)

def now_utc() -> datetime:
    """获取当前 UTC 时区的感知时间 (Timezone-Aware)"""
    return datetime.now(UTC_TZ)

def start_of_day(dt: datetime | None = None) -> datetime:
    """获取指定日期的零点 (00:00:00.000000)，默认以上海时区为准"""
    target = dt if dt is not None else now_shanghai()
    return datetime.combine(target.date(), time.min, tzinfo=target.tzinfo)

def end_of_day(dt: datetime | None = None) -> datetime:
    """获取指定日期的末尾 (23:59:59.999999)，默认以上海时区为准"""
    target = dt if dt is not None else now_shanghai()
    return datetime.combine(target.date(), time.max, tzinfo=target.tzinfo)

def to_timestamp_ms(dt: datetime | None = None) -> int:
    """转换为毫秒级时间戳"""
    target = dt if dt is not None else now_shanghai()
    return int(target.timestamp() * 1000)

def from_timestamp_ms(ts_ms: int, tz: ZoneInfo | timezone = SHANGHAI_TZ) -> datetime:
    """从毫秒级时间戳转换为时区感知时间"""
    return datetime.fromtimestamp(ts_ms / 1000.0, tz=tz)

def format_datetime(dt: datetime | None = None, fmt: str = DATETIME_FORMAT) -> str:
    """使用标准常量格式化时间"""
    target = dt if dt is not None else now_shanghai()
    return target.strftime(fmt)
```

### 4.5 `convx.py`：安全类型转换与兜底降级 (Conversions & Fallback)
```python
# common/utils/convx.py
from typing import Any

__all__ = ["to_int", "to_float", "to_bool"]

_TRUTHY_STRINGS = frozenset({"true", "1", "yes", "on", "t", "y"})
_FALSY_STRINGS = frozenset({"false", "0", "no", "off", "f", "n"})

def to_int(val: Any, default: int = 0) -> int:
    """安全将入参转为整数，发生 ValueError/TypeError 时返回默认兜底值"""
    if val is None:
        return default
    try:
        return int(val)
    except (ValueError, TypeError):
        return default

def to_float(val: Any, default: float = 0.0) -> float:
    """安全将入参转为浮点数"""
    if val is None:
        return default
    try:
        return float(val)
    except (ValueError, TypeError):
        return default

def to_bool(val: Any, default: bool = False) -> bool:
    """智能解析布尔语义 (支持 "true", "1", "yes" 等)"""
    if val is None:
        return default
    if isinstance(val, bool):
        return val
    if isinstance(val, (int, float)):
        return bool(val)
    if isinstance(val, str):
        cleaned = val.strip().lower()
        if cleaned in _TRUTHY_STRINGS:
            return True
        if cleaned in _FALSY_STRINGS:
            return False
    return default
```

---

# 5. 异常体系与上下文追踪规范 (Exceptions & ContextVars)

### 5.1 统一业务异常基类体系 (`common/exceptions/base.py`)
严禁在业务逻辑中直接随手抛出原生的 `raise Exception("msg")`，必须通过统一继承的结构化异常体系进行抛转：

```python
# common/exceptions/base.py
from typing import Any

class BaseBusinessException(Exception):
    """全局统一业务异常基类 (拒绝对外暴露裸 500)"""

    def __init__(self, code: int, message: str, data: Any = None) -> None:
        super().__init__(message)
        self.code = code
        self.message = message
        self.data = data

    def to_dict(self) -> dict[str, Any]:
        return {"code": self.code, "msg": self.message, "data": self.data}
```

### 5.2 异步与线程安全上下文管理 (`common/ctxdata/trace.py`)
严禁在异步高并发环境中使用模块级全局变量存储请求私有数据。必须使用 Python 3.7+ 原生 `contextvars.ContextVar`：

```python
# common/ctxdata/trace.py
import contextvars
from typing import Final

_TRACE_ID_CTX: Final[contextvars.ContextVar[str]] = contextvars.ContextVar("trace_id", default="")
_USER_ID_CTX: Final[contextvars.ContextVar[int | None]] = contextvars.ContextVar("user_id", default=None)

def get_trace_id() -> str:
    return _TRACE_ID_CTX.get()

def set_trace_id(trace_id: str) -> contextvars.Token[str]:
    return _TRACE_ID_CTX.set(trace_id)

def get_current_user_id() -> int | None:
    return _USER_ID_CTX.get()

def set_current_user_id(user_id: int | None) -> contextvars.Token[int | None]:
    return _USER_ID_CTX.set(user_id)
```

---

# 6. 现代常量与不可变数据工程规范 (Constants & Immutability)

### 6.1 命名约定与 `typing.Final` 静态保护
- 常量全部大写蛇形（`UPPER_SNAKE_CASE`）；
- 强制使用 `typing.Final` 注解，利用 Mypy 等类型检查器阻断二次修改。
  ```python
  from typing import Final

  DEFAULT_PAGE_SIZE: Final[int] = 20
  MAX_RETRY_COUNT: Final[int] = 3
  ```

### 6.2 状态与分类严格采用枚举类
```python
from enum import Enum, unique
import sys

if sys.version_info >= (3, 11):
    from enum import StrEnum
else:
    class StrEnum(str, Enum):
        pass

@unique
class OrderStatus(StrEnum):
    PENDING = "pending"
    PAID = "paid"
    COMPLETED = "completed"
    CANCELLED = "cancelled"
```

---

# 7. Pythonic 核心铁律与避坑防线 (Core Idioms & Anti-Patterns)

### 🚨 铁律一：默认可变参数死穴 (Mutable Default Argument - 绝对红线)
- 默认参数仅求值一次，**绝对禁止使用 `[]`, `{}` 等可变对象作为函数默认值**：
  ```python
  # ❌ 错误：共享列表跨请求污染
  def append_item(val: str, items: list[str] = []) -> list[str]: ...

  # ✅ 唯一正解：使用 None 并在内部初始化
  def append_item(val: str, items: list[str] | None = None) -> list[str]:
      target = items if items is not None else []
      target.append(val)
      return target
  ```

### 🚨 铁律二：生成器流式防 OOM (Generators over Full Lists)
- 海量数据、日志或数据库游标，必须使用 `(x for x in ...)` 或 `yield` 生成器，严禁一次性全量列表载入内存。

### 🚨 铁律三：异常链保留原则 (Exception Chaining)
- 转换或重抛异常时，**强制使用 `raise ... from err`** 保留原始异常根因堆栈。严禁裸 `except:`，严禁静默 `pass`。

---

# 8. 专属排障与静态诊断武器库 (Troubleshooting & Tooling)

- **极速代码检查与格式化 (Ruff)**：
  ```bash
  ruff check . --fix
  ruff format .
  ```
- **严格类型静态检查 (Mypy)**：
  ```bash
  mypy --strict .
  ```
- **内存泄漏与大对象分析 (Tracemalloc)**：
  ```python
  import tracemalloc
  tracemalloc.start()
  snapshot = tracemalloc.take_snapshot()
  for stat in snapshot.statistics("lineno")[:5]:
      print(stat)
  ```

---

# 9. Python 基础开发审查 Checklist

- [ ] **版本自适应**：是否已检查 `pyproject.toml` 并自适应当前运行环境？
- [ ] **防 kwargs 穿透**：公共 API 是否使用了显式命名参数或类型模型，而非裸 `**kwargs`？
- [ ] **最小知识导出**：模块与包是否显式定义了 `__all__`，私有成员是否均带有前导 `_`？
- [ ] **目录分层收敛**：基础公共层是否遵循一级收紧（`enums`, `types`, `exceptions`, `ctxdata`, `utils`, `storage`, `middlewares`）？
- [ ] **工具包命名风格**：工具包是否采用方案 B `*x` 扩展风格（`listx`, `dictx`, `strx`, `timex`, `convx`），彻底避开 `maputil` 违规与标准库同名遮蔽？
- [ ] **资源自动释放**：凡涉及外部资源是否均通过 Context Manager (`with`/`async with`) 封装？
- [ ] **时区感知与常量**：时间处理是否采用感知时间与 `ZoneInfo`，是否彻底收敛为时间常量？
- [ ] **默认参数与流式**：是否彻底消除了默认可变参数，是否在大数据集上使用了流式生成器？
