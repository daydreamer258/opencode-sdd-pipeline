# OpenCode SDD Workflow — 项目级 Contract

本项目是一个基于 SDD（Spec-Driven Development）方法论的 agentic workflow，运行在 OpenCode / Claude Code 上。

---

## 项目目标

构建一套 **spec-anchored、artifact-driven、implementation-bounded** 的轻量开发工作流，让 AI agent 按阶段、按角色、按权限协作完成 feature 交付。

## 核心原则

1. **Artifact-first**：所有阶段产出必须落到文件，不依赖会话记忆
2. **Stage-gated**：阶段按顺序推进，不跳过关键前置
3. **Implementation-bounded**：实施受 tasks 约束，不超范围
4. **Human-reviewable**：所有产物必须人类可审阅

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
| Orchestrator | 主 Agent | 流程调度、用户交互、阶段切换、Skill 调用 |
| Triage | SubAgent | 需求采集、规模判定 |
| Spec Writer | SubAgent | 规格定义 |
| Planner | SubAgent | 技术方案 |
| Task Decomposer | SubAgent | 任务拆解 |
| Implementer | SubAgent | 代码实施 |
| Reviewer | SubAgent | 代码审查（对照 spec/plan） |
| Validator | SubAgent | 验证核验 |

## 目录约定

```
.opencode/
├── templates/          # artifact 模板
├── agents/             # agent 定义（prompt + 规则写在正文中）
├── skills/             # skill 定义
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

- 所有 agent 必须参照 `.opencode/templates/` 中的模板生成产物
- 每个 agent 的规则直接写在其定义文件的正文中（`.opencode/agents/*.md`）
- SubAgent 不直接与用户交互，通过 Orchestrator 中转
- Orchestrator 不直接修改 SubAgent 生成的 artifact，必须通过 SubAgent 返工
- SubAgent 超步数退出时，Orchestrator 启动新的同类型 SubAgent 续接，不自己执行
- Implementer 不得执行破坏性命令（`rm -rf`、`git push --force` 等）
- Reviewer 和 Validator 不得修改代码，只做审查和验证
- 高风险 implement 必须经过 checkpoint 确认
- 所有 skill 在流程中显式调用
