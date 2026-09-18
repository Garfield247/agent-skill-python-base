# agent-skill-python-base

> 🐍 Python 3.10+ 现代基础语言规范、PEP 604 强类型系统、Pythonic 惯用法与数据模型 Agent Skill。

## 🌟 核心特性 (Features)

- **现代强类型系统**：PEP 604 原生联合类型（`X | Y`）、原生容器注解、`typing.Protocol` 结构化子类型接口。
- **Pythonic 避坑铁律**：彻底杜绝默认可变参数陷阱、海量数据生成器流式防 OOM、异常链保留（`raise ... from err`）。
- **高效现代数据模型**：`@dataclass(slots=True, frozen=True)` 节省 40% 内存并支持只读哈希。
- **现代模式匹配**：Python 3.10+ `match-case` 结构化解构。

## 📦 安装与多 Agent 使用指南 (Installation & Multi-Agent Usage)

### 1. Google Antigravity / Gemini Code Assist
- **全局安装（推荐）**：
  ```bash
  git clone git@github.com:Garfield247/agent-skill-python-base.git ~/.gemini/config/skills/python-base
  ```
- **项目工作区局部引入**：
  ```bash
  mkdir -p .agents/skills
  git clone git@github.com:Garfield247/agent-skill-python-base.git .agents/skills/python-base
  ```

### 2. Anthropic Claude (Claude Code / Claude Projects)
- **Claude Code (CLI 终端智能体)**：
  ```bash
  mkdir -p ~/.claude/skills
  git clone git@github.com:Garfield247/agent-skill-python-base.git ~/.claude/skills/python-base
  ```
- **Claude Projects (Web 客户端)**：
  直接将仓库中的 `SKILL.md` 内容复制并粘贴至 Project 的 **Project Knowledge**。

### 3. Cursor / GitHub Copilot / OpenAI Codex
- **Cursor (现代 MDC 规则体系)**：
  ```bash
  mkdir -p .cursor/rules
  git clone git@github.com:Garfield247/agent-skill-python-base.git .cursor/rules/python-base
  ```
- **GitHub Copilot / Codex**：
  将本技能规范注入项目的 `.github/copilot-instructions.md`。

## 📄 开源协议 (License)
本项目采用 [MIT License](LICENSE) 授权。
