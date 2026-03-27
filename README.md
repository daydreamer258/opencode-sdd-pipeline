# Codemate SDD Pipeline

基于 **SDD（Spec-Driven Development）** 方法论的 Agentic 开发工作流，运行在 [Codemate](https://codemate.ai) 上。

> 规范是中枢，产物是事实，流程是约束。

本项目编排了 7 个专业化子 Agent，由主 Agent（通过 `sdd-workflow` skill）扮演 Orchestrator 角色统一调度，通过阶段门控工作流协作完成从需求采集到代码验证的全流程——确保每个 feature 都以 Spec 为锚点、以 Artifact 为驱动、以 Task 为实施边界。

---

## 特性

- **阶段门控工作流** — 各阶段按序推进，每次切换前自动检查前置条件
- **多 Agent 协作** — 7 个子 Agent（Triage、Spec Writer、Planner、Task Decomposer、Implementer、Reviewer、Validator）由主 Agent 统一调度
- **迭代式精炼** — 需求、规格、方案、任务阶段均与用户反复迭代，直到明确定稿
- **风险感知执行** — 实施前自动评估风险等级，中/高风险需用户人工确认
- **规模路由** — 小任务跳过 spec/plan/tasks 阶段，大任务强制 checkpoint 和报告
- **Artifact 优先** — 每个阶段产出版本化的 Markdown 文件，不依赖会话记忆
- **强约束设计** — 主 Agent 不可直接修改制品、不可替代子 Agent 工作，适配中等能力模型

## 快速开始

1. 将本仓库的 `agents/`、`skills/`、`templates/` 目录和 `AGENTS.md` 文件复制到你项目的 `.codemate/` 文件夹下：

```
your-project/
└── .codemate/
    ├── agents/
    ├── skills/
    ├── templates/
    └── AGENTS.md
```

2. 启动 Codemate，开始 SDD 工作流：

```
/sdd-workflow 实现一个用户搜索功能，支持关键词匹配和结果排序
```

或手动初始化 feature 目录：

```
/feature-init 001 user-search
```

主 Agent 会通过 `sdd-workflow` skill 扮演 Orchestrator 角色，引导你走完每个阶段。

---

## 工作原理

```
intake → spec → plan → tasks → implement → review → validate → (report)
```

| 规模 | 路径 |
|------|------|
| **small** | intake → implement → review → validate |
| **medium** | intake → spec → plan → tasks → implement → review → validate |
| **large** | 完整路径 + 强制 checkpoint + 人工 review + report |

迭代型阶段（intake、spec、plan、tasks）遵循循环模式：子 Agent 写入/更新 artifact 并提出待澄清问题，主 Agent 在子 Agent 与用户之间来回传递，直到双方都没有问题。执行型阶段（implement、review、validate）一次性执行完毕并返回结果。

完整的架构图、工作流程图和 Agent/Skill 详细说明请参见 **[ARCHITECTURE.md](ARCHITECTURE.md)**。

---

## 项目结构

```
agents/                        # 子 Agent 定义（复制到 .codemate/）
├── triage.md                  # 需求采集与规模路由
├── spec-writer.md             # 功能规格定义
├── planner.md                 # 技术方案与取舍
├── task-decomposer.md         # 任务拆解（INVEST 原则）
├── implementer.md             # 代码实施
├── reviewer.md                # 代码审查（对照 spec/plan）
└── validator.md               # 验收标准核验

skills/                        # Skill 定义（复制到 .codemate/）
├── sdd-workflow/SKILL.md      # 主流程入口（内含 Orchestrator 调度逻辑）
├── feature-init/SKILL.md      # Feature 目录脚手架
├── stage-gate/SKILL.md        # 阶段切换门控
├── spec-check/SKILL.md        # Spec 质量审查
├── risk-assess/SKILL.md       # 实施前风险评估
└── diff-review/SKILL.md       # 变更-规格一致性审查

templates/                     # Artifact 模板（复制到 .codemate/）
├── 00-intake.md
├── 01-spec.md
├── 02-plan.md
├── 03-tasks.md
├── 04-implementation-log.md
├── 05-validation.md
└── 06-report.md

AGENTS.md                      # 项目级 Contract（复制到 .codemate/）
```

运行时，每个 feature 的产物生成在 `features/{id}-{name}/` 目录下。

---

## 与 OpenCode 版本的区别

本分支（codemate）针对 Codemate 平台做了以下适配：

| 项目 | OpenCode 版 | Codemate 版 |
|------|------------|-------------|
| Orchestrator | 独立 primary agent（`orchestrator.md`） | 主 Agent + `sdd-workflow` skill 驱动 |
| 子 Agent frontmatter | `mode`/`steps`/`permission` 格式 | `name`/`description`/`tools` 格式 |
| AGENTS.md | 自动加载 | 需通过 skill/subAgent 提示词嵌入规范 |
| 约束强度 | 常规 | 增强（适配中等能力模型） |
| 子 Agent 嵌套调用 | Reviewer 调用 diff-review | 主 Agent 统一调用，子 Agent 不嵌套 |

---

## 核心原则

1. **Artifact 优先** — 所有阶段产出必须落到文件，不依赖会话记忆
2. **阶段门控** — 各阶段按序推进，stage-gate 在每次切换前检查前置条件
3. **用户定稿制** — 每个 artifact 必须用户明确确认后才能进入下一阶段
4. **主 Agent 代理** — 子 Agent 不直接与用户交互，所有沟通通过主 Agent 中转
5. **实施有界** — 代码变更受 task 列表约束，不超范围
6. **Review + Validate 双重把关** — Reviewer 审查设计一致性，Validator 核验验收标准
7. **强约束执行** — 主 Agent 不可直接修改制品，不可替代子 Agent 工作

---

## 环境要求

- 已安装并配置 [Codemate](https://codemate.ai)

---

## 贡献

欢迎提 Issue 或 Pull Request！

---

## 许可证

MIT
