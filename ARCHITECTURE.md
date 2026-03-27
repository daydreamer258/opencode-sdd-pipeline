# 架构说明

本文档描述 Codemate SDD Pipeline 的内部架构。

---

## 系统架构总览

```mermaid
graph TB
    User((用户))
    Main[主 Agent<br/>via sdd-workflow skill<br/>扮演 Orchestrator]

    subgraph SubAgents[子 Agents — 不可与用户直接交互]
        Triage[triage<br/>需求采集]
        Spec[spec-writer<br/>规格定义]
        Planner[planner<br/>技术方案]
        TaskD[task-decomposer<br/>任务拆解]
        Impl[implementer<br/>代码实施]
        Rev[reviewer<br/>代码审查]
        Valid[validator<br/>验证核验]
    end

    subgraph Skills[Skills — 逻辑嵌入主 Agent 执行]
        SDDSkill[sdd-workflow<br/>主流程入口 + Orchestrator 逻辑]
        FeatInit[feature-init<br/>目录初始化]
        RiskA[risk-assess<br/>风险评估]
        StageG[stage-gate<br/>阶段检查]
        SpecC[spec-check<br/>Spec 质量审查]
        DiffR[diff-review<br/>变更一致性审查]
    end

    User <-->|对话交互| Main
    Main -->|调用| Triage
    Main -->|调用| Spec
    Main -->|调用| Planner
    Main -->|调用| TaskD
    Main -->|调用| Impl
    Main -->|调用| Rev
    Main -->|调用| Valid
    Main -.->|执行逻辑| FeatInit
    Main -.->|执行逻辑| RiskA
    Main -.->|执行逻辑| StageG
    Main -.->|执行逻辑| SpecC
    Main -.->|执行逻辑| DiffR

    Triage -->|返回 artifact + 疑问| Main
    Spec -->|返回 artifact + 疑问| Main
    Planner -->|返回 artifact + 疑问| Main
    TaskD -->|返回 artifact + 疑问| Main
    Impl -->|返回执行结果| Main
    Rev -->|返回审查结论| Main
    Valid -->|返回验证结论| Main
```

**与 OpenCode 版的关键区别**：
- 没有独立的 Orchestrator agent，主 Agent 通过 `sdd-workflow` skill 获得 Orchestrator 指令
- 子 Agent 不可嵌套调用（如 Reviewer 不再直接调用 diff-review），所有 skill 逻辑由主 Agent 执行
- 子 Agent frontmatter 使用 `name/description/tools` 格式

---

## 完整工作流程

```mermaid
flowchart TD
    Start([用户提出需求]) --> Init[/feature-init 初始化/]
    Init --> Intake[Intake 阶段<br/>triage 子 Agent 迭代循环]
    Intake --> IntakeDone{用户定稿?}
    IntakeDone -->|否, 有修改意见| Intake
    IntakeDone -->|是| SG1[/stage-gate 检查/]
    SG1 --> Route{规模路由}

    Route -->|small| SG4[/stage-gate 检查/]
    Route -->|medium / large| SpecPhase[Spec 阶段<br/>spec-writer 子 Agent 迭代循环]

    SpecPhase --> SC[/spec-check 检查/]
    SC --> SpecDone{用户定稿?}
    SpecDone -->|否| SpecPhase
    SpecDone -->|是| SG2[/stage-gate 检查/]
    SG2 --> PlanPhase[Plan 阶段<br/>planner 子 Agent 迭代循环]

    PlanPhase --> PlanDone{用户定稿?}
    PlanDone -->|否| PlanPhase
    PlanDone -->|是| SG3[/stage-gate 检查/]
    SG3 --> TaskPhase[Tasks 阶段<br/>task-decomposer 子 Agent 迭代循环]

    TaskPhase --> TaskDone{用户定稿?}
    TaskDone -->|否| TaskPhase
    TaskDone -->|是| SG4

    SG4[/stage-gate 检查/] --> Checkpoint[/risk-assess 评估/]
    Checkpoint --> RiskLevel{风险等级}
    RiskLevel -->|Low| ImplPhase
    RiskLevel -->|Medium| UserConfirm{用户确认<br/>checkpoint?}
    RiskLevel -->|High| ForceReview[强制停止<br/>用户 review plan + tasks]
    UserConfirm -->|确认| ImplPhase
    ForceReview -->|放行| ImplPhase

    ImplPhase[Implement 阶段<br/>implementer 子 Agent 执行] --> DiffRev[/diff-review 检查/]
    DiffRev --> ReviewPhase[Review 阶段<br/>reviewer 子 Agent 审查]
    ReviewPhase --> ReviewResult{审查结论}
    ReviewResult -->|通过| ValidPhase
    ReviewResult -->|需要修改| ImplPhase

    ValidPhase[Validate 阶段<br/>validator 子 Agent 验证] --> ValidResult{验证结论}
    ValidResult -->|通过| IsLarge{是 large 任务?}
    ValidResult -->|有条件通过| IsLarge
    ValidResult -->|不通过| ImplPhase

    IsLarge -->|是| Report[生成 06-report.md]
    IsLarge -->|否| Done([Workflow 完成])
    Report --> Done
```

---

## 迭代循环模式（核心交互模型）

Intake、Spec、Plan、Tasks 四个阶段都使用相同的**迭代循环**模式。子 Agent 每次调用都会写入/更新 artifact 并返回待澄清问题，主 Agent 负责在子 Agent 和用户之间来回传递，直到双方都没有问题为止。

```mermaid
sequenceDiagram
    participant U as 用户
    participant M as 主 Agent<br/>(Orchestrator)
    participant S as 子 Agent<br/>(如 triage)

    Note over M: 阶段开始
    M->>S: 首次调用：传入需求/上下文
    S->>S: 探索代码 + 写入 artifact
    S-->>M: 返回: artifact 摘要 + 待澄清问题

    M->>U: 展示 artifact 摘要 + 待澄清问题
    U-->>M: 回答问题 + 提出修改意见

    loop 迭代直到定稿
        M->>S: 迭代调用：传入用户回答 + 修改意见
        S->>S: 更新 artifact
        S-->>M: 返回: 更新摘要 + 新的待澄清问题(如有)
        M->>U: 展示更新后的 artifact + 新问题(如有)
        U-->>M: 回答 / 修改意见 / 确认通过
    end

    Note over M: 用户确认定稿 → 执行 stage-gate 检查 → 进入下一阶段
```

**定稿条件**：子 Agent 没有更多疑问 **且** 用户明确表示没有问题。只要任一方仍有疑问或修改意见，循环就继续。

---

## 子 Agent 续接机制

当子 Agent 因工作量过大未能完成时，主 Agent **不可自己接手执行**，必须调用新的同类型子 Agent 续接。

```mermaid
sequenceDiagram
    participant M as 主 Agent
    participant S1 as 子 Agent #1
    participant S2 as 子 Agent #2 (续接)

    M->>S1: 调用：传入任务
    S1->>S1: 执行任务...
    Note over S1: 未能完成所有工作
    S1->>S1: 更新 implementation-log 记录进度
    S1-->>M: 退出：返回已完成任务 + 未完成任务

    M->>M: 读取进度，准备续接参数
    M->>S2: 续接调用：传入未完成任务 + 当前进度
    S2->>S2: 从未完成任务继续执行
    S2-->>M: 返回：完成摘要
```

---

## 执行型子 Agent 交互模型

Implementer、Reviewer、Validator 是**执行型**子 Agent，不走迭代循环，而是执行完毕后一次性返回结果。

```mermaid
sequenceDiagram
    participant U as 用户
    participant M as 主 Agent
    participant I as implementer
    participant R as reviewer
    participant V as validator

    Note over M: Implement 阶段
    M->>I: 调用：传入 feature 目录 + tasks
    I->>I: 逐个执行 tasks 中的代码改动
    I->>I: 持续更新 04-implementation-log.md
    I-->>M: 返回: 完成摘要

    Note over M: diff-review 检查
    M->>M: 执行 diff-review 逻辑

    Note over M: Review 阶段
    M->>R: 调用：传入 feature 目录
    R->>R: 对照 spec/plan 审查代码变更
    R-->>M: 返回: 审查结论 + 问题列表
    M->>U: 展示审查结果

    alt 审查通过
        Note over M: 进入 Validate 阶段
    else 审查不通过
        M->>U: 展示问题
        Note over M: 回到 Implement 修复
    end

    Note over M: Validate 阶段
    M->>V: 调用：传入 feature 目录
    V->>V: 逐条核验 AC + 运行测试
    V->>V: 写入 05-validation.md
    V-->>M: 返回: pass/fail 结论
    M->>U: 展示验证结果
```

---

## 子 Agent 一览

| Agent | 类型 | 职责 | 产出 | 可用工具 |
|-------|------|------|------|---------|
| **triage** | 迭代型 | 需求采集、规模判定 | `00-intake.md` | read,glob,grep,edit |
| **spec-writer** | 迭代型 | 功能规格定义 | `01-spec.md` | read,glob,grep,edit |
| **planner** | 迭代型 | 技术方案设计 | `02-plan.md` | read,glob,grep,bash,edit |
| **task-decomposer** | 迭代型 | 任务拆解（INVEST 原则） | `03-tasks.md` | read,glob,grep,edit |
| **implementer** | 执行型 | 代码实施 | `04-implementation-log.md` + 代码变更 | read,glob,grep,bash,edit |
| **reviewer** | 执行型 | 代码审查 | 审查结论 | read,glob,grep,bash,edit |
| **validator** | 执行型 | 验收标准核验 + 测试 | `05-validation.md` | read,glob,grep,bash,edit |

---

## Skill 一览

| Skill | 调用者 | 调用时机 | 用户可触发 |
|-------|--------|---------|:---:|
| **sdd-workflow** | 用户 | 启动 SDD 流程，主 Agent 获得 Orchestrator 指令 | 是 |
| **feature-init** | 主 Agent / 用户 | Workflow 启动初期 | 是 |
| **stage-gate** | 主 Agent（内联执行） | 每次阶段切换前 | 否 |
| **spec-check** | 主 Agent（内联执行） | Spec 产出后 | 否 |
| **risk-assess** | 主 Agent（内联执行） | 进入 Implement 前 | 否 |
| **diff-review** | 主 Agent（内联执行） | Review 阶段开始前 | 否 |

**注意**：在 Codemate 中，子 Agent 不可调用 skill 或其他子 Agent。所有 skill 逻辑由主 Agent 参照对应 skill 文件内联执行。

---

## 规模路由

Triage Agent 在 Intake 阶段评估任务规模，决定后续走哪条路径。用户可以覆盖 Agent 的判断。

```mermaid
flowchart LR
    Intake[Intake 定稿] --> Size{规模判定}
    Size -->|"small<br/>bug fix / 单文件 / 配置"| PathS["intake → implement<br/>→ review → validate"]
    Size -->|"medium<br/>新功能 / 多文件"| PathM["intake → spec → plan → tasks<br/>→ implement → review → validate"]
    Size -->|"large<br/>架构变更 / 跨模块"| PathL["完整路径 + 强制 checkpoint<br/>+ 人工 review + report"]
```

---

## Stage Gate 机制

每个阶段切换前，主 Agent 执行 `stage-gate` 检查逻辑。不满足条件不允许进入下一阶段。

| 从 | 到 | 前置条件 |
|----|-----|---------|
| Intake | Spec | `00-intake.md` 用户已定稿，规模 >= medium |
| Intake | Implement | `00-intake.md` 用户已定稿，规模 = small |
| Spec | Plan | `01-spec.md` 用户已定稿 |
| Plan | Tasks | `02-plan.md` 用户已定稿 |
| Tasks | Implement | `03-tasks.md` 用户已定稿 + checkpoint 通过 |
| Implement | Review | `04-implementation-log.md` 已生成 |
| Review | Validate | Reviewer 审查通过或用户确认 |
| Validate (fail) | Implement | 重新进入修复 |
| Validate (pass) | Report | 仅 large 任务 |

---

## 风险评估 Checkpoint

进入 Implement 前，主 Agent 执行 `risk-assess` 评估逻辑。

```mermaid
flowchart TD
    Tasks[Tasks 定稿] --> SG[/stage-gate 检查/]
    SG --> RA[/risk-assess 评估/]

    RA --> Low{Low 风险}
    RA --> Med{Medium 风险}
    RA --> High{High 风险}

    Low -->|自动继续| Impl[进入 Implement]
    Med -->|生成 checkpoint 摘要| UserMed[用户确认后继续]
    High -->|强制停止| UserHigh[用户 review plan + tasks<br/>手动放行]

    UserMed --> Impl
    UserHigh --> Impl
```

**评估维度**：变更文件数量、核心模块影响、数据变更、外部依赖、不可逆操作。

---

## 关键约束

- 子 Agent **不能**与用户直接交互——所有用户沟通由主 Agent 代理
- 主 Agent **不可**直接修改子 Agent 生成的 artifact——必须将修改意见传给子 Agent 执行
- 主 Agent **不可**自己执行子 Agent 的工作——子 Agent 未完成时必须调用新的同类型子 Agent 续接
- 子 Agent **不可**调用其他子 Agent 或 skill——所有调度由主 Agent 完成
- 所有 Skill 逻辑由主 Agent **内联执行**

---

## 目录结构

```
.codemate/
├── agents/                         # 子 Agent 定义
│   ├── triage.md                   # 需求采集
│   ├── spec-writer.md              # 规格定义
│   ├── planner.md                  # 技术方案
│   ├── task-decomposer.md          # 任务拆解
│   ├── implementer.md              # 代码实施
│   ├── reviewer.md                 # 代码审查
│   └── validator.md                # 验证核验
├── skills/                         # Skill 定义
│   ├── sdd-workflow/SKILL.md       # 主流程入口 + Orchestrator 逻辑
│   ├── feature-init/SKILL.md       # 目录初始化（用户可触发）
│   ├── risk-assess/SKILL.md        # 风险评估（主 Agent 内联执行）
│   ├── stage-gate/SKILL.md         # 阶段检查（主 Agent 内联执行）
│   ├── spec-check/SKILL.md         # Spec 质量审查（主 Agent 内联执行）
│   └── diff-review/SKILL.md        # 变更一致性审查（主 Agent 内联执行）
├── templates/                      # Artifact 模板（00-06）
└── AGENTS.md                       # 项目级 Contract

features/
└── {id}-{name}/                    # 每个 feature 的产物目录
    ├── 00-intake.md
    ├── 01-spec.md
    ├── 02-plan.md
    ├── 03-tasks.md
    ├── 04-implementation-log.md
    ├── 05-validation.md
    └── 06-report.md                # 可选
```
