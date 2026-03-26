# OpenCode Workflow v1 提案（修订版）

本文是面向 `opencode/` 目录的第一版 workflow 提案修订版。

基于以下三份研究文档整合而成：

- `opencode-能力盘点.md` — OpenCode runtime 能力边界
- `sdd-我们应该吸收什么-不该照搬什么.md` — SDD 方法论取舍
- `sdd-优秀工作流实践与模板建议.md` — 阶段化最佳实践

---

## 1. 核心定位

> **OpenCode workflow v1 = spec-anchored、artifact-driven、rule-aware、agent-operable、implementation-bounded 的轻量工作流。**

关键取舍：

- **吸收**：规范中枢、阶段分离、agent 可配置、implement checkpoint、validation gate
- **不做**：spec-as-source、重型命令体系、全自动 archive/delta、完整 constitution 治理
- **允许**：小任务走轻量路径，不强制所有任务走完整流程

---

## 2. 设计原则

### 2.1 规范是中枢，不是附件
artifacts 是 workflow 核心，会话只是交互入口。agent 围绕明确 artifacts 工作，而不是"聊着聊着顺手写点文件"。

### 2.2 OpenCode 是 runtime，不是流程本身
OpenCode 提供 Plan/Build 模式、subagent、permissions、project config。但真正的 workflow 由我们定义，只利用对 workflow 最关键的能力。

### 2.3 implement 必须比前面阶段更受控
implement 开始接触真实代码，必须保留 risk assessment、checkpoint、validation gate。

### 2.4 不同规模不同流程
小任务可以跳过 spec/plan/tasks 直接 implement；大任务强制走完整流程 + 人工 review。避免"所有任务都走重流程"导致的摩擦。

---

## 3. Workflow 阶段定义

v1 包含 **6 个核心阶段 + 1 个可选阶段**：

```
intake → spec → plan → tasks → implement → validate → (report)
```

### 阶段路由（按任务规模）

| 规模 | 路径 | 说明 |
|------|------|------|
| `small` | intake → implement → validate | bug fix、小调整，可跳过 spec/plan/tasks |
| `medium` | intake → spec → plan → tasks → implement → validate | 标准路径 |
| `large` | 完整路径 + 强制 checkpoint + 人工 review + report | 高风险/大范围改动 |

规模由 Intake 阶段的 Triage Agent 评估，用户可覆盖。

---

### 3.1 Intake（需求采集与分流）

**目的**：将原始需求结构化，判断任务规模，决定后续路径。

**产物**：`00-intake.md`

**重点内容**：
- 背景与动机
- 原始需求（保持原文）
- 问题定义
- 目标（可检查的 checklist）
- 约束条件
- 非目标（防止范围蔓延）
- 待澄清项
- **任务规模判定**（small / medium / large）

**Stage Gate**：用户确认 intake 内容后方可进入下一阶段。

---

### 3.2 Spec（规格定义）

**目的**：定义"做什么"和"为什么做"，不涉及技术方案。

**产物**：`01-spec.md`

**重点内容**：
- 问题陈述
- 用户与场景
- 期望行为（具体描述，不模糊）
- 验收标准（每条可测试/可验证）
- 边界情况
- 假设
- 非目标

**Stage Gate**：spec 必须 human-reviewable。用户确认后方可进入 plan。

---

### 3.3 Plan（技术计划）

**目的**：定义"怎么做"，将 spec 落实到技术方案。

**产物**：`02-plan.md`

**重点内容**：
- 技术方案摘要
- 涉及模块与文件
- 接口与契约
- 数据流
- 实现顺序
- 约束与取舍（选 A 不选 B 的理由）
- **风险评估**（含整体风险等级）
- 验证方案

**Stage Gate**：plan 必须覆盖 spec 中所有验收标准。

---

### 3.4 Tasks（任务分解）

**目的**：将 plan 拆成 agent 可执行的离散小任务。

**产物**：`03-tasks.md`

**重点内容**：
- 任务总览表（编号、状态、依赖、风险）
- 每个任务的：目标、变更范围、前置依赖、完成标准、实现提示、风险等级
- 遵循 INVEST 原则：小、清楚、可验证、可独立完成

**Stage Gate**：所有任务依赖关系清晰，覆盖 plan 中所有步骤。

---

### 3.5 Implement（实施）

**目的**：基于 spec + plan + tasks 做真实代码改动。

**产物**：`04-implementation-log.md`

**重点内容**：
- 实施摘要（状态、风险）
- **Checkpoint 记录**（每次进入 implement 前的风险评估）
- 任务执行记录（变更文件、决策、偏离、问题）
- 未完成项
- 计划外变更

**核心规则**：
- implement 前必须做 risk assessment
- `low` 风险：可自动继续
- `medium` / `high` 风险：必须 checkpoint，等待用户确认
- 实施过程中持续更新 implementation-log
- 不得超出 tasks 定义的范围

---

### 3.6 Validate（验证）

**目的**：验证实现是否符合 spec / plan / tasks，判断是否可通过。

**产物**：`05-validation.md`

**重点内容**：
- 验收标准逐条核验（对照 spec AC）
- 执行的检查与命令（实际跑了什么）
- Spec-实现一致性检查（有无功能漂移、超范围实现）
- 未解决问题
- **明确的通过/不通过结论**

**核心规则**：
- validate 必须回答"与 spec 是否一致"
- validate fail 不可视为完成
- 不能只写"已验证通过"，必须有可追溯依据

---

### 3.7 Report（收尾报告，可选）

**目的**：复盘 workflow 本身，沉淀经验，迭代 workflow。

**产物**：`06-report.md`

**重点内容**：
- 交付总结
- 做得好/不好的地方
- Workflow 本身的问题
- 模板/Prompt 问题
- 经验沉淀与改进建议

**触发条件**：`large` 任务建议必填，`medium` 可选，`small` 不需要。

---

## 4. 四层架构

### 4.1 Artifact Layer（事实层）

feature 级别的工作产物，每个 feature 一个目录：

```
features/{{feature-id}}-{{feature-name}}/
├── 00-intake.md
├── 01-spec.md
├── 02-plan.md
├── 03-tasks.md
├── 04-implementation-log.md
├── 05-validation.md
└── 06-report.md          # 可选
```

模板位于 `.opencode/templates/`，agent 参照模板生成产物。

### 4.2 Instruction Layer（规则层）

project-level 的 agent 工作约束：

- `AGENTS.md` — 项目级 contract，定义角色、目录约定、基本规则
- `agents/orchestrator.md` — 包含 workflow 规则、stage gate、checkpoint 规则（直接写在 agent 正文中）
- `agents/implementer.md` — 包含 implement checkpoint 规则
- `agents/validator.md` — 包含 validation 规则与反模式

**设计决策**：规则直接写在对应 agent 的正文中，而不是独立的 `rules/` 目录。原因是 subagent 不一定能自动加载外部 rules 文件，而 agent 正文是**确定**会被加载的。

### 4.3 Agent Layer（角色层）

利用 OpenCode 的 Plan/Build primary agents 和 subagent 机制：

（详见第 5 节 SubAgent 规划）

### 4.4 Execution Layer（执行层）

让 workflow 可运行的机制：

- feature 目录初始化（由 Skill 或手动触发）
- 阶段进入条件检查（stage gate）
- implement 风险评估与 checkpoint
- validation gate
- 规模路由（small/medium/large）

---

## 5. SubAgent 与 Skill 规划

### 5.1 主 Agent：Orchestrator

Orchestrator 是整个 workflow 的**主控 agent**，不直接产出 artifact，而是负责：

- **流程调度**：根据当前阶段决定调用哪个 subagent
- **规模路由**：根据 intake 中的规模判定选择 small/medium/large 路径
- **Stage Gate 检查**：每次阶段切换前调用 stage-gate skill 检查前置条件
- **Checkpoint 决策**：implement 前调用 risk-assess，根据结果决定继续或等待用户确认
- **用户交互**：阶段间的确认、澄清、方向调整都由 Orchestrator 与用户对话
- **异常处理**：subagent 报错、validate fail、scope 偏离时，由 Orchestrator 决定下一步

**OpenCode 模式**：Plan（默认），必要时切换到 Build
**权限**：read + 调用 subagent + 调用 skill，不直接写代码

**工作流程**：

```
用户需求 → Orchestrator
  ├─ 调用 feature-init skill（初始化目录）
  ├─ [Intake 阶段] 迭代循环（Triage Agent）
  │   ├─ 调用 subagent → 写入 00-intake.md + 返回摘要和待澄清问题
  │   ├─ 展示给用户 → 用户回答问题 + 审核 artifact
  │   └─ 循环直到：无待澄清问题 且 用户定稿
  ├─ 判断规模路由
  │   ├─ small: 跳到 Implement
  │   └─ medium/large: 继续
  ├─ [Spec 阶段] 迭代循环（Spec Agent）→ 01-spec.md → 用户定稿
  ├─ [Plan 阶段] 迭代循环（Planner Agent）→ 02-plan.md → 用户定稿
  ├─ [Tasks 阶段] 迭代循环（Task Agent）→ 03-tasks.md → 用户定稿
  ├─ 调用 risk-assess skill → checkpoint 决策
  ├─ [Implement 阶段] Implementer Agent 执行 → 用户审核
  ├─ [Validate 阶段] Validator Agent 验证 → 用户审核
  ├─ validate pass? → 完成 / fail → 回到 implement 或更早阶段
  └─ (large) 生成 06-report.md
```

Orchestrator 是唯一面向用户的 agent，subagent 不直接与用户交互。

**关键约束：subagent 不能与用户交互**

OpenCode runtime 中，subagent 无法直接向用户提问或等待用户确认。当 subagent 需要用户输入时，它会退出并返回结果给 primary agent。因此，所有用户交互必须由 Orchestrator 代理完成。

**解决方案：迭代循环模式**

每个产出 artifact 的 subagent 每次调用都会：写入/更新 artifact + 返回待澄清问题。Orchestrator 负责在 subagent 和用户之间迭代传递，直到双方都没有问题。

迭代循环流程：

1. **首次调用**：Orchestrator 传入需求/上下文 → subagent 写入 artifact + 返回摘要和待澄清问题
2. **Orchestrator 展示**：将 artifact 摘要 + 待澄清问题展示给用户，请用户回答问题并审核 artifact
3. **迭代调用**：Orchestrator 将用户回答 + 修改意见传给 subagent → subagent 更新 artifact + 返回新的待澄清问题（如有）
4. **重复步骤 2-3**，直到 subagent 无疑问 且 用户确认定稿
5. **定稿后进入下一阶段**，只要用户提出任何修改意见就必须调用 subagent 返工

---

### 5.2 SubAgent 总览

| SubAgent | 作用 | OpenCode 模式 | 权限 | 输出 |
|----------|------|--------------|------|------|
| **Triage Agent** | 分析需求，判断规模，生成 intake | Plan | read-only | `00-intake.md` |
| **Spec Agent** | 根据 intake 编写规格定义 | Plan | read-only + write artifacts | `01-spec.md` |
| **Planner Agent** | 根据 spec 制定技术方案 | Plan | read-only + write artifacts | `02-plan.md` |
| **Task Agent** | 将 plan 拆解为可执行任务 | Plan | read-only + write artifacts | `03-tasks.md` |
| **Implementer Agent** | 执行 tasks 中的代码改动 | Build | read + write + bash (受限) | `04-implementation-log.md` + 代码变更 |
| **Validator Agent** | 验证实现与 spec 一致性 | Plan/Build | read + bash (只读命令) | `05-validation.md` |

---

### 5.3 SubAgent 详细说明

#### Triage Agent（迭代型 subagent）

**职责**：
- 接收用户的原始需求描述
- 结构化为 intake 文档
- 评估任务规模（small / medium / large）
- 识别待澄清项
- 推荐后续流程路径

**调用模式**：迭代循环。每次调用都写入/更新 `00-intake.md`，返回摘要 + 待澄清问题。循环直到无疑问且用户定稿。

**权限**：read, glob, grep, edit（无 bash）

**调用时机**：workflow 启动时，用户提出新需求

---

#### Spec Agent（迭代型 subagent）

**职责**：
- 阅读 intake，探索现有代码上下文
- 定义问题、用户场景、期望行为
- 编写可测试的验收标准
- 识别边界情况
- 确保 spec human-reviewable（不超过 2-3 页）

**调用模式**：迭代循环。每次调用都写入/更新 `01-spec.md`，返回摘要 + 待澄清问题。循环直到无疑问且用户定稿。

**权限**：read, glob, grep, edit（无 bash）

**调用时机**：intake 定稿后（medium/large 任务）

---

#### Planner Agent（迭代型 subagent）

**职责**：
- 阅读 spec，探索代码架构
- 制定技术方案（怎么做、为什么这样做）
- 识别涉及的模块和文件
- 评估风险
- 规划实现顺序

**调用模式**：迭代循环。每次调用都写入/更新 `02-plan.md`，返回摘要 + 待澄清问题 + 技术取舍。循环直到无疑问且用户定稿。

**权限**：read, glob, grep, bash（只读命令）, edit

**调用时机**：spec 定稿后

---

#### Task Agent（迭代型 subagent）

**职责**：
- 阅读 plan，将其拆解为离散的小任务（INVEST 原则）
- 定义依赖关系
- 标记高风险任务
- 编写实现提示

**调用模式**：迭代循环。每次调用都写入/更新 `03-tasks.md`，返回摘要 + 待澄清问题 + 粒度建议。循环直到无疑问且用户定稿。

**权限**：read, glob, grep, edit（无 bash）

**调用时机**：plan 定稿后

---

#### Implementer Agent（执行型 subagent）

**职责**：
- 按照 tasks 逐个执行代码改动
- 持续更新 implementation-log
- 不超出 tasks 定义的变更范围
- 遇到 plan 外问题时记录偏离而不是自行决定
- 变更超出预期时**停止并退出**，让 Orchestrator 重新评估

**调用模式**：执行型，非迭代。执行完毕后返回完成摘要给 Orchestrator（完成任务、改动文件、偏离、阻塞项）。

**权限**：read, glob, grep, bash（受限，禁止破坏性命令）, edit

**调用时机**：tasks 定稿后，checkpoint 通过后

---

#### Validator Agent（执行型 subagent）

**职责**：
- 逐条核验 spec 中的验收标准
- 执行测试命令和验证脚本
- 检查 spec-实现一致性（有无功能漂移）
- 检查是否有计划外变更
- 给出明确的 pass/fail 结论
- fail 时指出具体问题和建议

**交互模式**：无澄清模式。验证完毕后返回 pass/fail 结果 + 问题列表给 Orchestrator。

**权限**：read, glob, grep, bash（只读验证命令）, edit（只写 `05-validation.md`）

**调用时机**：implement 完成后

---

### 5.4 Skill 总览

| Skill | 作用 | 触发方式 | 说明 |
|-------|------|---------|------|
| **sdd-workflow** | **主流程 Skill**，启动并驱动整个 workflow | 用户手动触发（如 `/sdd`） | 激活 Orchestrator，开始 workflow 主循环 |
| **feature-init** | 初始化 feature 目录结构 | Orchestrator 调用 / 手动 | 创建 `features/xxx/` 目录，从 templates 复制模板 |
| **risk-assess** | 评估实施风险等级 | implement 前由 Orchestrator 触发 | 分析 tasks 涉及的文件范围、改动类型，输出 low/medium/high |
| **stage-gate** | 检查是否满足进入下一阶段的条件 | 阶段切换时由 Orchestrator 触发 | 检查前置 artifact 是否存在且完整 |
| **spec-check** | 验证 spec 完整性 | spec 编写后 | 检查验收标准是否可测试、是否有遗漏 |
| **diff-review** | 审查代码变更与 spec 一致性 | validate 阶段 | 对比 git diff 与 spec/tasks 的一致性 |

---

### 5.5 Skill 详细说明

#### sdd-workflow（主流程 Skill）

**作用**：整个 workflow 的用户入口，激活 Orchestrator 并驱动主循环。

**触发方式**：用户通过 `/sdd` 或类似命令启动。

**行为**：
1. 接收用户的需求描述（或提示用户输入）
2. 激活 Orchestrator Agent
3. Orchestrator 开始主循环：初始化 → intake → 路由 → 按阶段推进 → 完成
4. 每个阶段间由 Orchestrator 与用户确认后再继续

**为什么需要这个 Skill**：
- 没有它，用户需要手动调用每个 subagent，workflow 就散了
- 它是"一键启动 SDD 流程"的入口
- 它把 Orchestrator 的调度逻辑封装成一个可触发的能力

---

#### feature-init

**作用**：初始化一个新 feature 的工作目录。

**行为**：
1. 在 `features/` 下创建 `{id}-{name}/` 目录
2. 从 `.opencode/templates/` 复制所有模板文件
3. 替换模板中的 `{{feature-name}}` 占位符

**触发**：用户说"开始一个新 feature"或 workflow 启动时。

---

#### risk-assess

**作用**：在 implement 前评估风险等级。

**评估维度**：
- 涉及文件数量和范围
- 是否修改核心模块 / 公共接口
- 是否涉及数据库 / 配置变更
- 任务复杂度

**输出**：`low` / `medium` / `high` + 理由

**后续行为**：
- `low`：自动继续
- `medium`：生成 checkpoint 摘要，等待用户确认
- `high`：强制停止，要求用户 review plan 和 tasks

---

#### stage-gate

**作用**：检查是否满足进入下一阶段的条件。

**检查项**：
- 前置 artifact 文件是否存在
- 关键字段是否已填写（非空模板）
- 用户是否已确认

---

#### spec-check

**作用**：审查 spec 文档的质量。

**检查项**：
- 验收标准是否具体、可测试
- 是否有边界情况说明
- 范围是否合理
- 是否过长（超过建议长度则警告）

---

#### diff-review

**作用**：在 validate 阶段，对比代码变更与 spec/tasks。

**行为**：
- 读取 `git diff` 输出
- 对照 tasks 中的变更范围
- 检查是否有超出 tasks 定义的改动
- 检查是否有 tasks 定义但未完成的改动

---

## 6. Agent 与 OpenCode 能力的映射

| OpenCode 能力 | Workflow 中的使用方式 |
|--------------|---------------------|
| Primary / Subagent 机制 | Orchestrator 为 primary agent，其余 6 个为 subagent |
| Subagent 不能与用户交互 | 所有用户交互通过 Orchestrator 代理（迭代循环模式） |
| Permission 控制 | 按角色分配 read/glob/grep/bash/edit 权限（YAML record 格式） |
| Project-level agents config | `.opencode/agents/*.md` 定义每个 agent（prompt + 规则直接写在正文中） |
| Steps 限制 | 每个 subagent 设置 50 步上限（Implementer 200 步） |
| Bash integration | Implementer（受限执行）、Validator（只读命令）、Planner（只读命令） |
| 各 subagent 内置 read/glob/grep | 每个 subagent 自行探索代码，无需独立 Explorer agent |

---

## 7. AGENTS / Rules / Skills / Artifacts 分工

这是 v1 最需要避免混乱的地方。

| 层 | 作用域 | 职责 | 不应包含 |
|----|--------|------|---------|
| **AGENTS.md** | project | 项目级 contract：目标、角色、目录约定、基本规则 | 不是超长 prompt |
| **Agents** | project | 每个 agent 的职责、规则、约束（prompt + rules 合一） | 不互相重复 |
| **Skills** | project | 可复用的任务能力入口 | 不是全项目治理文件 |
| **Artifacts** | feature | 当前 feature 的事实状态，agent 的共享工作面 | 不包含项目级规则 |
| **Templates** | project | artifact 的结构参考，agent 按此生成产物 | 不是已填内容 |

> **设计决策**：阶段 prompt 和 workflow 规则直接写在对应 agent 定义文件的正文中（`.opencode/agents/*.md`），不再单独维护 `prompts/` 或 `rules/` 目录。原因是 subagent 的正文是确定会被加载的，而外部文件不一定。

---

## 8. 关键规则

### 8.1 artifact-first
任何阶段讨论，最终都应落到 feature 产物，不留在会话里。

### 8.2 stage-order
默认顺序：`intake → spec → plan → tasks → implement → validate`

### 8.3 stage-gate
- 没有 intake 不进 spec
- 没有 spec 不进 plan（small 任务可跳过）
- 没有 plan 不进 tasks
- 没有 tasks 不进 implement
- validate fail 不可视为完成

### 8.4 implement-checkpoint
- 进入 implement 前进行风险评估
- `low`：可继续
- `medium`：checkpoint，等用户确认
- `high`：强制停止 + 人工 review

### 8.5 validation-gate
validate 必须回答：
- 做了什么检查
- 与 spec 是否一致
- 是否存在偏移
- 是否允许通过

### 8.6 scope-bound
implement 不得超出 tasks 定义的变更范围。计划外改动必须记录在 implementation-log 中。

---

## 9. 建议目录结构

```
.opencode/
├── docs/                           # 研究与提案文档
├── templates/                      # artifact 模板（agent 参照生成）
│   ├── 00-intake.md
│   ├── 01-spec.md
│   ├── 02-plan.md
│   ├── 03-tasks.md
│   ├── 04-implementation-log.md
│   ├── 05-validation.md
│   └── 06-report.md
├── agents/                         # agent 定义（prompt + 规则直接写在正文中）
│   ├── orchestrator.md             # 主控 agent
│   ├── triage.md
│   ├── spec-writer.md
│   ├── planner.md
│   ├── task-decomposer.md
│   ├── implementer.md
│   ├── reviewer.md
│   └── validator.md
├── skills/                         # 可复用能力（显式调用）
│   ├── sdd-workflow/               # 主流程入口 skill
│   ├── feature-init/
│   ├── risk-assess/
│   ├── stage-gate/
│   ├── spec-check/
│   └── diff-review/
└── AGENTS.md                       # 项目级 contract
```

Feature 产物目录：

```
features/{{feature-id}}-{{feature-name}}/
├── 00-intake.md
├── 01-spec.md
├── 02-plan.md
├── 03-tasks.md
├── 04-implementation-log.md
├── 05-validation.md
└── 06-report.md                    # 可选
```

---

## 10. 成功标准

v1 是否成功，看这 6 件事：

1. **需求到任务的链路是否清晰** — intake/spec/plan/tasks 是否稳定可用
2. **implement 是否真正受控** — checkpoint 是否生效，高风险时是否能停下来
3. **validation 是否真能作为 gate** — 不是写一句"通过"就算完
4. **agent 分工是否清楚** — 每个 subagent 边界清晰，不互相踩踏
5. **小任务是否能走快** — small 路径不被重流程拖累
6. **项目说明是否足够让新 agent 进入上下文** — 不靠口头解释才能上手

---

## 11. 下一步

### 第一步：确认本提案
- 阶段定义是否合理
- subagent 分工是否清楚
- skill 规划是否足够/过多
- 模板是否需要调整

### 第二步：搭 project-level 骨架
- `AGENTS.md`
- `agents/`（subagent 定义，prompt + 规则直接写在正文中）
- `skills/`

### 第三步：最小 demo 验证
选一个小 feature，走一遍 intake → tasks → implement → validate，验证 workflow 是否跑得通。

---

## 12. 依据与判断边界

本提案基于三类输入：

### OpenCode runtime 能力
- Plan / Build 双模式、subagent 机制、permission 控制、project-level config
- 来源：OpenCode 官方文档

### SDD 外部实践
- SpecKit 阶段化、OpenSpec 审计思想、Intent-Driven 最佳实践
- 来源：Jimmy Song SDD 系列、Microsoft、GitHub、Thoughtworks、Fowler

### 当前仓库经验
- lightweight shell-based pipeline 的验证经验
- implement checkpoint 的现实价值
- 文档/模板过重的风险教训

本提案是面向我们自己的 OpenCode workflow v1 设计，不是外部标准答案。

---

## 参考来源

- OpenCode docs: <https://opencode.ai/docs/>
- OpenCode agents: <https://opencode.ai/docs/agents/>
- GitHub Spec Kit: <https://github.com/github/spec-kit>
- Jimmy Song SDD: <https://jimmysong.io/zh/book/ai-handbook/sdd/methodology/>
- Intent-Driven: <https://intent-driven.dev/knowledge/best-practices/>
- Martin Fowler SDD: <https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html>
