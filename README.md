# OpenCode SDD Workflow

基于 **SDD（Spec-Driven Development）** 方法论的 agentic 开发工作流，运行在 [OpenCode](https://opencode.ai) 上。

核心理念：**规范是中枢，产物是事实，流程是约束。**

---

## 系统架构总览

```mermaid
graph TB
    User((用户))
    Orch[Orchestrator<br/>主 Agent]

    subgraph SubAgents[SubAgents 不可与用户直接交互]
        Triage[Triage Agent<br/>需求采集与分流]
        Spec[Spec Writer Agent<br/>规格定义]
        Planner[Planner Agent<br/>技术方案]
        TaskD[Task Decomposer Agent<br/>任务拆解]
        Impl[Implementer Agent<br/>代码实施]
        Rev[Reviewer Agent<br/>代码审查]
        Valid[Validator Agent<br/>验证核验]
    end

    subgraph Skills[Skills 显式调用]
        SDDSkill[sdd-workflow<br/>主流程入口]
        FeatInit[feature-init<br/>目录初始化]
        RiskA[risk-assess<br/>风险评估]
        StageG[stage-gate<br/>阶段检查]
        SpecC[spec-check<br/>Spec 质量审查]
        DiffR[diff-review<br/>变更一致性审查]
    end

    User <-->|对话交互| Orch
    Orch -->|调用| Triage
    Orch -->|调用| Spec
    Orch -->|调用| Planner
    Orch -->|调用| TaskD
    Orch -->|调用| Impl
    Orch -->|调用| Rev
    Orch -->|调用| Valid
    Orch -.->|显式调用| FeatInit
    Orch -.->|显式调用| RiskA
    Orch -.->|显式调用| StageG
    Orch -.->|显式调用| SpecC
    Rev -.->|显式调用| DiffR

    Triage -->|返回 artifact + 疑问| Orch
    Spec -->|返回 artifact + 疑问| Orch
    Planner -->|返回 artifact + 疑问| Orch
    TaskD -->|返回 artifact + 疑问| Orch
    Impl -->|返回执行结果| Orch
    Rev -->|返回审查结论| Orch
    Valid -->|返回验证结论| Orch
```

**关键约束**：
- SubAgent 不能与用户直接交互，所有用户沟通由 Orchestrator 代理
- Orchestrator 不可直接修改 SubAgent 生成的 artifact，必须将修改意见传给 SubAgent 执行
- Orchestrator 不可自己执行 SubAgent 的工作，SubAgent 超步数退出时必须启动新的同类型 SubAgent 续接
- 所有 Skill 在流程中显式调用

---

## 完整工作流程

```mermaid
flowchart TD
    Start([用户提出需求]) --> Init[/feature-init Skill/]
    Init --> Intake[Intake 阶段<br/>Triage Agent 迭代循环]
    Intake --> IntakeDone{用户定稿?}
    IntakeDone -->|否, 有修改意见| Intake
    IntakeDone -->|是| SG1[/stage-gate Skill/]
    SG1 --> Route{规模路由}

    Route -->|small| SG4[/stage-gate Skill/]
    Route -->|medium / large| SpecPhase[Spec 阶段<br/>Spec Writer Agent 迭代循环]

    SpecPhase --> SC[/spec-check Skill/]
    SC --> SpecDone{用户定稿?}
    SpecDone -->|否| SpecPhase
    SpecDone -->|是| SG2[/stage-gate Skill/]
    SG2 --> PlanPhase[Plan 阶段<br/>Planner Agent 迭代循环]

    PlanPhase --> PlanDone{用户定稿?}
    PlanDone -->|否| PlanPhase
    PlanDone -->|是| SG3[/stage-gate Skill/]
    SG3 --> TaskPhase[Tasks 阶段<br/>Task Decomposer Agent 迭代循环]

    TaskPhase --> TaskDone{用户定稿?}
    TaskDone -->|否| TaskPhase
    TaskDone -->|是| SG4

    SG4[/stage-gate Skill/] --> Checkpoint[/risk-assess Skill/]
    Checkpoint --> RiskLevel{风险等级}
    RiskLevel -->|Low| ImplPhase
    RiskLevel -->|Medium| UserConfirm{用户确认<br/>checkpoint?}
    RiskLevel -->|High| ForceReview[强制停止<br/>用户 review plan + tasks]
    UserConfirm -->|确认| ImplPhase
    ForceReview -->|放行| ImplPhase

    ImplPhase[Implement 阶段<br/>Implementer Agent 执行] --> ReviewPhase[Review 阶段<br/>Reviewer Agent 审查<br/>内部调用 diff-review Skill]
    ReviewPhase --> ReviewResult{审查结论}
    ReviewResult -->|通过| ValidPhase
    ReviewResult -->|需要修改| ImplPhase

    ValidPhase[Validate 阶段<br/>Validator Agent 验证] --> ValidResult{验证结论}
    ValidResult -->|通过| IsLarge{是 large 任务?}
    ValidResult -->|有条件通过| IsLarge
    ValidResult -->|不通过| ImplPhase

    IsLarge -->|是| Report[生成 06-report.md]
    IsLarge -->|否| Done([Workflow 完成])
    Report --> Done
```

---

## 迭代循环模式（核心交互模型）

Intake / Spec / Plan / Tasks 四个阶段都使用相同的**迭代循环**模式。SubAgent 每次调用都会写入/更新 artifact 并返回待澄清问题，Orchestrator 负责在 SubAgent 和用户之间来回传递，直到双方都没有问题为止。

```mermaid
sequenceDiagram
    participant U as 用户
    participant O as Orchestrator<br/>(主 Agent)
    participant S as SubAgent<br/>(如 Triage)

    Note over O: 阶段开始
    O->>S: 首次调用：传入需求/上下文
    S->>S: 探索代码 + 写入 artifact
    S-->>O: 返回: artifact 摘要 + 待澄清问题

    O->>U: 展示 artifact 摘要 + 待澄清问题
    U-->>O: 回答问题 + 提出修改意见

    loop 迭代直到定稿
        O->>S: 迭代调用：传入用户回答 + 修改意见
        S->>S: 更新 artifact
        S-->>O: 返回: 更新摘要 + 新的待澄清问题(如有)
        O->>U: 展示更新后的 artifact + 新问题(如有)
        U-->>O: 回答 / 修改意见 / 确认通过
    end

    Note over O: 用户确认定稿 → 调用 stage-gate skill → 进入下一阶段
```

**定稿条件**：SubAgent 没有更多疑问 **且** 用户明确表示没有问题。只要任一方仍有疑问或修改意见，循环就继续。

---

## SubAgent 超步数续接机制

当 SubAgent 因超出最大步数（steps 上限）退出时，Orchestrator **不可自己接手执行**，必须启动新的同类型 SubAgent 续接。

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant S1 as SubAgent #1
    participant S2 as SubAgent #2 (续接)

    O->>S1: 调用：传入任务
    S1->>S1: 执行任务...
    Note over S1: 达到步数上限
    S1->>S1: 更新 implementation-log 记录进度
    S1-->>O: 退出：返回已完成任务 + 未完成任务

    O->>O: 读取进度，准备续接参数
    O->>S2: 续接调用：传入未完成任务 + 当前进度
    S2->>S2: 从未完成任务继续执行
    S2-->>O: 返回：完成摘要
```

---

## 执行型 SubAgent 交互模型

Implementer、Reviewer、Validator 是**执行型** SubAgent，不走迭代循环，而是执行完毕后一次性返回结果。

```mermaid
sequenceDiagram
    participant U as 用户
    participant O as Orchestrator
    participant I as Implementer Agent
    participant R as Reviewer Agent
    participant V as Validator Agent

    Note over O: Implement 阶段
    O->>I: 调用：传入 feature 目录 + tasks
    I->>I: 逐个执行 tasks 中的代码改动
    I->>I: 持续更新 04-implementation-log.md
    I-->>O: 返回: 完成摘要

    Note over O: Review 阶段
    O->>R: 调用：传入 feature 目录
    R->>R: 对照 spec/plan 审查代码变更
    R->>R: 调用 diff-review skill
    R-->>O: 返回: 审查结论 + 问题列表
    O->>U: 展示审查结果

    alt 审查通过
        Note over O: 进入 Validate 阶段
    else 审查不通过
        O->>U: 展示问题
        Note over O: 回到 Implement 修复
    end

    Note over O: Validate 阶段
    O->>V: 调用：传入 feature 目录
    V->>V: 逐条核验 AC + 运行测试
    V->>V: 写入 05-validation.md
    V-->>O: 返回: pass/fail 结论
    O->>U: 展示验证结果
```

---

## SubAgent 详细说明

### Triage Agent（迭代型）

- **功能**：接收原始需求，结构化为 intake 文档，评估任务规模（small/medium/large），识别待澄清项
- **调用时机**：Workflow 启动，用户提出新需求
- **输出**：`00-intake.md` + 待澄清问题 + 规模判定
- **Steps**：50
- **调用的 Skill**：无

---

### Spec Writer Agent（迭代型）

- **功能**：根据 intake 编写功能规格定义，定义"做什么"和"为什么做"，确保验收标准可测试、边界情况已识别
- **调用时机**：Intake 定稿后（仅 medium/large 任务）
- **输出**：`01-spec.md` + 待澄清问题
- **Steps**：50
- **调用的 Skill**：无（Orchestrator 在 Spec 产出后调用 `spec-check` skill）

---

### Planner Agent（迭代型）

- **功能**：根据 spec 制定技术方案，定义"怎么做"，识别涉及模块、接口契约、实现顺序、风险与取舍
- **调用时机**：Spec 定稿后
- **输出**：`02-plan.md` + 待澄清问题 + 技术取舍
- **Steps**：50
- **调用的 Skill**：无

---

### Task Decomposer Agent（迭代型）

- **功能**：将 plan 拆解为 agent 可执行的离散小任务（INVEST 原则），定义依赖关系和完成标准
- **调用时机**：Plan 定稿后
- **输出**：`03-tasks.md` + 待澄清问题 + 粒度建议
- **Steps**：50
- **调用的 Skill**：无

---

### Implementer Agent（执行型）

- **功能**：按照 tasks 逐个执行代码改动，持续更新 implementation-log，遵守 scope 约束
- **调用时机**：Tasks 定稿 + `stage-gate` skill 通过 + `risk-assess` skill checkpoint 通过后
- **前置 Skill 调用**（由 Orchestrator 执行）：
  - `stage-gate`：检查 `03-tasks.md` 已定稿
  - `risk-assess`：评估风险等级，medium/high 需用户确认
- **输出**：`04-implementation-log.md` + 代码变更 + 完成摘要
- **Steps**：200（支持复杂任务）
- **超步数处理**：优先更新 implementation-log 记录进度后退出，Orchestrator 启动新 Implementer 续接
- **调用的 Skill**：无

---

### Reviewer Agent（执行型）

- **功能**：在 implement 完成后，审查代码变更是否符合 spec 和 plan 中的设计，检查代码质量和一致性
- **调用时机**：Implement 完成，用户审核通过后
- **输出**：审查结论（通过 / 需要修改 / 不通过）+ 问题列表 + 修改建议
- **Steps**：50
- **调用的 Skill**：
  - `diff-review`：Reviewer 在审查过程中调用，检查代码变更与 spec/tasks 的一致性

---

### Validator Agent（执行型）

- **功能**：逐条核验 spec 验收标准，运行测试命令，检查 spec-实现一致性，给出明确的 pass/fail 结论
- **调用时机**：Reviewer 审查通过后
- **输出**：`05-validation.md` + pass/fail 结论 + 问题列表
- **Steps**：50
- **调用的 Skill**：无

---

## Skill 调用时机与条件

```mermaid
flowchart TD
    subgraph Workflow[完整流程中的 Skill 调用点]
        S1[/"① feature-init<br/>调用者: Orchestrator<br/>时机: Workflow 启动"/]
        S2[/"② spec-check<br/>调用者: Orchestrator<br/>时机: Spec Agent 产出后<br/>条件: 规模 >= medium"/]
        S3[/"③ stage-gate<br/>调用者: Orchestrator<br/>时机: 每次阶段切换前"/]
        S4[/"④ risk-assess<br/>调用者: Orchestrator<br/>时机: 进入 Implement 前<br/>条件: stage-gate 通过"/]
        S5[/"⑤ diff-review<br/>调用者: Reviewer Agent<br/>时机: Review 阶段审查过程中"/]
    end

    S1 --> S3
    S3 --> S2
    S2 --> S3
    S3 --> S4
    S4 --> S5
```

| Skill | 调用者 | 调用时机 | 调用条件 | 是否用户可触发 |
|-------|--------|---------|---------|:---:|
| **sdd-workflow** | 用户 | 启动 SDD 流程 | 用户输入 `/sdd-workflow` | 是 |
| **feature-init** | Orchestrator / 用户 | Workflow 启动初期 | 无 | 是 |
| **stage-gate** | Orchestrator | 每次阶段切换前 | 无（每次切换必须调用） | 否 |
| **spec-check** | Orchestrator | Spec Agent 产出 `01-spec.md` 后 | 规模 >= medium（即有 Spec 阶段时） | 否 |
| **risk-assess** | Orchestrator | 进入 Implement 前 | `stage-gate` 通过后 | 否 |
| **diff-review** | Reviewer Agent | Review 阶段审查过程中 | Implement 完成后 | 否 |

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

每个阶段切换前，Orchestrator 显式调用 `stage-gate` Skill 检查前置条件。不满足条件不允许进入下一阶段。

```mermaid
flowchart LR
    subgraph Gate["stage-gate Skill 检查项"]
        G1[前置 artifact 文件存在?]
        G2[关键字段非空?]
        G3[用户已定稿?]
    end

    G1 & G2 & G3 -->|全部通过| Pass[Gate Pass<br/>进入下一阶段]
    G1 & G2 & G3 -->|任一失败| Fail[Gate Fail<br/>阻止进入]
```

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
| Validate (pass) | Report | 仅 large 任务建议 |

---

## Implement Checkpoint 机制

进入 Implement 前，Orchestrator 显式调用 `risk-assess` Skill 评估风险。

```mermaid
flowchart TD
    Tasks[Tasks 定稿] --> SG[/stage-gate Skill/]
    SG --> RA[/risk-assess Skill/]

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

## 目录结构

```
.opencode/
├── agents/                         # Agent 定义
│   ├── orchestrator.md             # 主 Agent（primary）
│   ├── triage.md                   # 需求采集 SubAgent
│   ├── spec-writer.md              # 规格定义 SubAgent
│   ├── planner.md                  # 技术方案 SubAgent
│   ├── task-decomposer.md          # 任务拆解 SubAgent
│   ├── implementer.md              # 代码实施 SubAgent (steps: 200)
│   ├── reviewer.md                 # 代码审查 SubAgent
│   └── validator.md                # 验证核验 SubAgent
├── skills/                         # Skill 定义
│   ├── sdd-workflow/SKILL.md       # 主流程入口（用户可触发）
│   ├── feature-init/SKILL.md       # 目录初始化（用户可触发）
│   ├── risk-assess/SKILL.md        # 风险评估（Orchestrator 调用）
│   ├── stage-gate/SKILL.md         # 阶段检查（Orchestrator 调用）
│   ├── spec-check/SKILL.md         # Spec 质量审查（Orchestrator 调用）
│   └── diff-review/SKILL.md        # 变更一致性审查（Reviewer 调用）
├── templates/                      # Artifact 模板（00-06）
├── docs/                           # 研究与提案文档
├── AGENTS.md                       # 项目级 Contract
└── README.md                       # 本文件

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

---

## 快速开始

```bash
# 启动 SDD 工作流
/sdd-workflow 实现一个用户搜索功能，支持关键词匹配和结果排序

# 或手动初始化 feature 目录
/feature-init 001 user-search
```

---

## 核心规则

1. **Artifact-first** — 所有阶段产出必须落到文件，不依赖会话记忆
2. **Stage-gated** — 阶段按顺序推进，切换前必须调用 `stage-gate` skill 检查
3. **用户定稿制** — 每个 artifact 必须用户明确确认后才能进入下一阶段
4. **SubAgent 不直接与用户交互** — 所有用户沟通通过 Orchestrator 代理
5. **Orchestrator 不直接修改 artifact** — 修改必须通过对应 SubAgent 执行
6. **Orchestrator 不代替 SubAgent 执行** — SubAgent 超步数退出时启动新的同类型 SubAgent 续接
7. **Skill 显式调用** — 所有 Skill 在流程中显式调用，不隐式依赖
8. **Implementation-bounded** — 代码实施受 tasks 约束，不超范围
9. **Review + Validate 双重把关** — Reviewer 审查设计一致性，Validator 核验验收标准
10. **Validate 不可跳过** — validate fail 不可视为完成
