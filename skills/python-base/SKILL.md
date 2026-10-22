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

# 现代 Python 核心语言与基础层架构规范 (Python Base Skill)

本技能定义了编写 **Python 基础模块、公共工具库、领域实体对象、业务逻辑及微服务基础层** 时的核心规范、质量门禁与渐进式手册索引。

---

## 1. 核心铁律与绝对红线 (Core Redlines)

1. 🚨 **环境自适应感知先行**：
   - 动工前必须前置检查 `pyproject.toml`、`runtime.txt` 或 Dockerfile，感知宿主 Python 版本；
   - 3.8/3.9 环境严禁在未导入 `from __future__ import annotations` 时直接使用 `|` 联合类型。
2. 🚨 **显式优于隐式，严禁滥用 `**kwargs` 穿透黑盒**：
   - 公共 API 必须使用显式命名参数与类型注解，善用仅限关键字参数 `*`；
   - 复杂入参（>4 个）必须封装为 Pydantic `BaseModel` 或 `@dataclass(slots=True)`。
3. 🚨 **最小知识导出与资源上下文闭环**：
   - 模块必须显式定义 `__all__`，内部私有成员统一以单下划线 `_` 开头；
   - 凡涉及文件、网络会话、连接池与锁，**必须通过 `with` / `async with` 上下文管理器封装自动释放**，严禁要求调用方手动调用 `.close()`。
4. 🚨 **基础公共层收紧与方案 B `*x` 扩展命名法**：
   - `common/` 一级目录严格收敛为 7 大类（`enums`, `types`, `exceptions`, `ctxdata`, `utils`, `storage`, `middlewares`）；
   - 工具包统一采用 `*x` 命名（`listx.py`, `dictx.py`, `strx.py`, `timex.py`, `convx.py`），杜绝拼写畸形与标准库同名遮蔽。
5. 🚨 **避坑三大绝命红线**：
   - **严禁使用默认可变参数**（如 `def fn(items=[])`），默认值一律为 `None`；
   - **大数据集强制流式生成器**，严禁一次性全部 `list(...)` 载入导致 OOM；
   - **异常重抛必须使用 `raise ... from err`** 保留原始调用栈，严禁静默 `pass`。

---

## 2. 标准开发工作流 (Standard Workflow)

1. **版本感知**：探测项目 Python 版本基线，对齐类型语法兼容性；
2. **契约建模**：在 `common/types/` 或领域模块中确立 Pydantic / dataclass 强类型实体与枚举；
3. **安全编码**：编写业务函数，强制使用显式参数，涉及资源利用上下文管理器回收；
4. **工具沉淀**：通用计算下沉至 `common/utils/*x.py` 并严格维护 `__all__`；
5. **静态门禁**：运行 `ruff check .` 与 `mypy --strict .` 验证零告警。

---

## 3. 渐进式专业参考手册 (Progressive Disclosure References)

根据具体涉及的业务场景，**按需查阅 `references/` 目录下的专门规约**，严禁将全量子文档一次性无脑全部展开：

| 场景模块 | 参考子文档 | 核心涵盖规范 |
| :--- | :--- | :--- |
| **版本基调与降级** | [01_version_matrix_and_fallbacks.md](references/01_version_matrix_and_fallbacks.md) | • Python 3.10+ 北极星演进基调<br>• Python 3.8/3.9 低版本优雅降级对照表<br>• `__future__ import annotations` 与类型容器选型 |
| **代码封装六大原则** | [02_encapsulation_principles.md](references/02_encapsulation_principles.md) | • 显式传参与防 `**kwargs` 黑盒穿透<br>• 显式 `__all__` 与最小知识原则<br>• 防原始类型偏执与上下文管理器自动释放 |
| **基础层架构与工具集** | [03_common_architecture_and_x_helpers.md](references/03_common_architecture_and_x_helpers.md) | • `common/` 一级分类收紧目录拓扑<br>• 方案 B: `*x` 扩展命名法标准<br>• `listx`/`dictx`/`strx`/`timex`/`convx` 标杆代码 |
| **异常与并发上下文** | [04_exceptions_and_contextvars.md](references/04_exceptions_and_contextvars.md) | • 全局 `BaseBusinessException` 结构化异常体系<br>• `contextvars.ContextVar` 异步与线程安全链路追踪<br>• TraceID 与当前用户全局无感透传 |
| **常量规范与排障武器库** | [05_constants_and_anti_patterns.md](references/05_constants_and_anti_patterns.md) | • `typing.Final` 与 `StrEnum` 规范<br>• 默认可变参数、生成器流式防 OOM 铁律<br>• Ruff 格式化、Mypy 严格检查与 Tracemalloc 排障 |
