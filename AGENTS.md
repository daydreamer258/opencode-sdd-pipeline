# Codemate SDD Workflow — 项目级 Contract

本项目是一个基于 SDD（Spec-Driven Development）方法论的 agentic workflow，运行在 Codemate 上。

---

## 项目目标

构建一套 **spec-anchored、artifact-driven、implementation-bounded** 的轻量开发工作流，让 AI agent 按阶段、按角色、按权限协作完成 feature 交付。

## 架构概述

Codemate 不支持自定义主 Agent，因此本项目通过 `sdd-workflow` skill 让主 Agent（general）扮演 Orchestrator 角色。子 Agent 不支持嵌套调用，所有调度由主 Agent 统一完成。

## 核心原则

1. **Artifact-first**：所有阶段产出必须落到文件，不依赖会话记忆
2. **Stage-gated**：阶段按顺序推进，不跳过关键前置
3. **Implementation-bounded**：实施受 tasks 约束，不超范围
4. **Human-reviewable**：所有产物必须人类可审阅
5. **强约束执行**：主 Agent 不可直接修改制品，不可替代子 Agent 工作

## Workflow 阶段

```
intake → spec → plan → tasks → implement → review → validate → (report)
```

按任务规模路由：
- `small`：intake → implement → review → validate
- `medium`：完整路径
- `large`：完整路径 + 强制 checkpoint + 人工 review + report

## 角色清单

| 角色 | 类型 | 职责 |
|------|------|------|
| 主 Agent（via sdd-workflow skill） | Orchestrator | 流程调度、用户交互、阶段切换、Skill 逻辑执行 |
| triage | 子 Agent | 需求采集、规模判定 |
| spec-writer | 子 Agent | 规格定义 |
| planner | 子 Agent | 技术方案 |
| task-decomposer | 子 Agent | 任务拆解 |
| implementer | 子 Agent | 代码实施 |
| reviewer | 子 Agent | 代码审查（对照 spec/plan） |
| validator | 子 Agent | 验证核验 |

## 目录约定

```
.codemate/
├── templates/          # artifact 模板
├── agents/             # 子 Agent 定义（name/description/tools frontmatter）
├── skills/             # skill 定义（Orchestrator 逻辑在 sdd-workflow 中）
└── AGENTS.md           # 本文件

features/
└── {id}-{name}/        # 每个 feature 的产物目录
    ├── 00-intake.md
    ├── 01-spec.md
    ├── 02-plan.md
    ├── 03-tasks.md
    ├── 04-implementation-log.md
    ├── 05-validation.md
    └── 06-report.md    # 可选
```

## 基本规则

- 所有 agent 必须参照 `templates/` 中的模板生成产物
- 每个子 Agent 的规则直接写在其定义文件的正文中（`agents/*.md`）
- 子 Agent 不直接与用户交互，通过主 Agent 中转
- **主 Agent 不可直接修改子 Agent 生成的 artifact，必须通过子 Agent 返工**
- **主 Agent 不可替代子 Agent 完成工作——子 Agent 未完成时，必须再次调用同类型子 Agent 续接**
- Implementer 不得执行破坏性命令（`rm -rf`、`git push --force` 等）
- Reviewer 和 Validator 不得修改代码，只做审查和验证
- 高风险 implement 必须经过 checkpoint 确认
- 子 Agent 不可调用其他子 Agent，所有调度由主 Agent 完成
- 所有 skill 逻辑由主 Agent 在流程中执行（stage-gate、spec-check、risk-assess、diff-review）

## 子 Agent Frontmatter 格式

```yaml
---
name: agent-name
description: 一句话描述 agent 职责
tools: read,glob,grep,edit
---
```

- `name`：agent 标识符
- `description`：职责描述
- `tools`：允许使用的工具列表（逗号分隔）

## 强约束说明

本项目设计为适配中等能力模型，因此在提示词中嵌入了更强的约束：

- 每个子 Agent 提示词头部包含 `⚠️ 核心规范` 区块，列出所有 [禁止] 和 [必须] 规则
- `sdd-workflow` skill 中包含 `⚠️ 绝对禁止` 区块，防止主 Agent 越权
- 子 Agent 不依赖 AGENTS.md 自动加载，关键规范直接嵌入各自提示词中
