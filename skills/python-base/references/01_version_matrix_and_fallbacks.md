# Python 现代版本演进基调与自适应降级矩阵 (Version Baseline & Adaptive Matrix)

## 1. 演进北极星基调 (Python 3.10+)

在无既有历史包袱的新立项中，**统一以 Python 3.10+ 作为现代设计基调**：
- 采用原生强类型联合操作符 `|`（PEP 604）；
- 采用结构模式匹配 `match...case`（PEP 634）；
- 采用高效数据模型 `@dataclass(slots=True)`（PEP 681）。

---

## 2. 环境自适应感知与降级决策表 (Adaptive Sniffing & Fallback)

动工前第一步必须感知运行宿主真实版本（检查 `pyproject.toml`、`runtime.txt`、`.python-version`、`Dockerfile`）：

| 语法/特性维度 | 推荐现代基调 (Python 3.10+) | 低版本优雅降级方案 (Python 3.8 / 3.9) | 核心考量与避坑红线 |
| :--- | :--- | :--- | :--- |
| **联合类型 (Union)** | `str | None`<br/>`int | str` | **首选**：文件顶部引入 `from __future__ import annotations`<br/>**备选**：`from typing import Optional, Union` | 严禁在 3.8/3.9 未导入 `__future__` 时直接使用 `|` 运算，避免 `TypeError` 崩溃。 |
| **通用集合容器类型** | 原生：`list[str]`, `dict[str, Any]` | 3.9 支持原生；3.8 需使用 `from typing import List, Dict` | 避免在现代环境过度导入陈旧 `typing.List`。 |
| **原生字符串枚举** | Python 3.11+ 原生 `enum.StrEnum` | 3.8 ~ 3.10 优雅降级：`class MyEnum(str, Enum):` | 严禁在业务中直接乱传散装裸字符串。 |
| **结构模式匹配** | `match...case` | 降级为清晰字典映射分发（Dispatch Table）或结构化 `if-elif-else` | 3.10 之前的版本不支持 `match` 关键字。 |
| **高效数据模型** | `@dataclass(slots=True)` | 3.10+ 支持 `slots=True`；3.8/3.9 降级为普通 `@dataclass` | 3.8/3.9 不支持 `slots=True` 参数。 |
