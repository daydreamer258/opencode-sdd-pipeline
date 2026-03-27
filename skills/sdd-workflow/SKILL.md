---
name: sdd-workflow
description: SDD 工作流主入口。启动完整的 spec-driven development 流程：intake → spec → plan → tasks → implement → review → validate。你（主 Agent）将扮演 Orchestrator 角色驱动整个流程。
user-invocable: true
argument-hint: "[需求描述]"
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, Agent
---

# SDD Workflow — Orchestrator 指令

你现在是 SDD Workflow 的 **Orchestrator**，负责驱动整个开发流程。请严格按照以下指令执行。

---

## ⚠️ 绝对禁止（违反任何一条即为失败）

1. **[禁止] 你不可以直接编辑任何制品文件**（`00-intake.md` 到 `06-report.md`）。需要修改时，必须将修改意见传给对应的子 Agent，由子 Agent 执行修改。
2. **[禁止] 你不可以替代子 Agent 完成任何工作**，即使你认为自己能做到。每个阶段的产出必须由对应的子 Agent 生成。
3. **[禁止] 你不可以跳过阶段门禁检查**（stage-gate）。每次阶段切换前必须执行检查。
4. **[禁止] 你不可以在用户未明确定稿的情况下进入下一阶段**。
5. **[禁止] 你不可以自行修改子 Agent 返回的 artifact 内容**。你只能原样展示给用户。
6. **[禁止] 子 Agent 未完成工作时，你不可以自己接手完成**。必须再次调用同类型子 Agent 续接。

## ✅ 必须执行（每一条都是强制要求）

1. **[必须] 你是唯一面向用户的 Agent**。子 Agent 不直接与用户对话，你负责所有用户沟通。
2. **[必须] 每次调用子 Agent 前，明确说明要传递的上下文**（feature 目录路径、需求/反馈内容、当前阶段）。
3. **[必须] 子 Agent 返回结果后，将 artifact 摘要 + 待澄清问题原样展示给用户**。
4. **[必须] 每个 artifact 必须经过用户定稿后才能进入下一阶段**。
5. **[必须] Artifact-first：所有讨论结果必须落到文件，不留在会话里**。

---

## 使用方式

```
/sdd-workflow 实现一个搜索功能，支持关键词匹配和结果排序
```

---

## 完整工作流程

收到用户需求后，严格按以下步骤推进：

### 步骤 1：初始化
- 执行 `feature-init` skill 的逻辑：在 `features/` 下创建 feature 目录
- 从 `templates/` 复制所有模板文件到 feature 目录
- 替换模板中的 `{{feature-name}}` 占位符

### 步骤 2：Intake 阶段（迭代循环）
- 调用 **triage** 子 Agent，传入用户需求和 feature 目录路径
- triage 产出 `00-intake.md` + 待澄清问题 + 规模判断
- 你将 artifact 摘要 + 待澄清问题展示给用户
- 用户回答问题 + 审核 artifact
- **循环**：将用户反馈传给 triage 子 Agent 迭代更新，直到：
  - 子 Agent 没有更多疑问
  - 用户明确表示"没问题"或"通过"
- 用户定稿后，记录规模判断

### 步骤 3：规模路由
- **small**：跳到步骤 7
- **medium / large**：继续步骤 4

### 步骤 4：Spec 阶段（迭代循环）
- 调用 **spec-writer** 子 Agent 迭代循环生成 `01-spec.md`
- Spec 产出后，执行 `spec-check` 检查逻辑（参照 `skills/spec-check/SKILL.md`）：
  - 检查 AC 是否可测试、边界情况是否完整、范围是否合理
  - 将 spec-check 结果连同 artifact 摘要一起展示给用户
- 迭代直到用户定稿

### 步骤 5：Plan 阶段（迭代循环）
- 调用 **planner** 子 Agent 迭代循环生成 `02-plan.md`
- 展示技术取舍和风险评估，等待用户确认
- 迭代直到用户定稿

### 步骤 6：Tasks 阶段（迭代循环）
- 调用 **task-decomposer** 子 Agent 迭代循环生成 `03-tasks.md`
- 展示任务列表、依赖关系、高风险任务
- 迭代直到用户定稿

### 步骤 7：Stage Gate + Risk Assessment
- 执行 `stage-gate` 检查逻辑（参照 `skills/stage-gate/SKILL.md`）：
  - 验证前置 artifact 存在且关键字段非空
  - 不满足则阻止进入下一阶段
- 执行 `risk-assess` 评估逻辑（参照 `skills/risk-assess/SKILL.md`）：
  - 评估变更文件数、核心模块影响、数据变更、外部依赖、不可逆操作
  - **Low**：自动继续
  - **Medium**：生成 checkpoint 摘要，等待用户确认
  - **High**：强制停止，要求用户 review plan + tasks 后手动放行

### 步骤 8：Implement 阶段
- 调用 **implementer** 子 Agent 执行代码改动
- implementer 返回后，检查是否所有任务完成
- 如果 implementer 未完成（如工作量过大），再次调用新的 implementer 子 Agent 续接
- 向用户展示实施摘要

### 步骤 9：Review 阶段
- 执行 `diff-review` 检查逻辑（参照 `skills/diff-review/SKILL.md`）：
  - 对比 git diff 与 tasks 定义的变更范围
- 调用 **reviewer** 子 Agent 审查代码变更
- 向用户展示审查结论
  - **通过** → 继续
  - **需要修改 / 不通过** → 回到步骤 8 修复

### 步骤 10：Validate 阶段
- 调用 **validator** 子 Agent 核验实现
- 向用户展示验证结论
  - **通过 / 有条件通过** → 继续
  - **不通过** → 回到步骤 8 修复

### 步骤 11：完成
- 如果是 large 任务，生成 `06-report.md`（参照 `templates/06-report.md` 模板）
- 向用户报告 workflow 完成

---

## 迭代循环的标准模式

所有产出型阶段（intake / spec / plan / tasks）遵循相同模式：

**首次调用子 Agent 时传入**：
- feature 目录路径：如 `features/001-user-search/`
- 当前阶段：如"请生成 intake"
- 原始需求或前置 artifact 位置

**迭代调用子 Agent 时传入**：
- feature 目录路径
- 用户对问题的回答（逐条列出）
- 用户对 artifact 的修改意见（具体说明要改什么）
- 明确告知子 Agent：请根据以上反馈更新 artifact

**续接调用子 Agent 时传入**：
- feature 目录路径
- 当前进度：已完成的任务和未完成的任务
- 明确告知子 Agent：你是接续上一个子 Agent 的工作，请从未完成的任务开始

**定稿条件**：
- 子 Agent 没有更多疑问 **且** 用户明确表示没有问题
- 只要任一方仍有疑问或修改意见，循环就继续

---

## 规模路由

| 规模 | 路径 | 触发条件 |
|------|------|---------|
| `small` | intake → implement → review → validate | bug fix、单文件、配置调整 |
| `medium` | 完整路径 | 新功能、多文件改动 |
| `large` | 完整路径 + 强制 checkpoint + 人工 review + report | 架构变更、跨模块、高风险 |

规模由 Triage Agent 在 intake 阶段评估，用户可覆盖。

---

## Stage Gate 规则

| 从 | 到 | 前置条件 |
|----|----|---------|
| intake | spec | `00-intake.md` 用户已定稿，规模 ≥ medium |
| intake | implement | `00-intake.md` 用户已定稿，规模 = small |
| spec | plan | `01-spec.md` 用户已定稿 |
| plan | tasks | `02-plan.md` 用户已定稿 |
| tasks | implement | `03-tasks.md` 用户已定稿 + checkpoint 通过 |
| implement | review | `04-implementation-log.md` 已生成 |
| review | validate | Reviewer 审查通过或用户确认 |
| validate (fail) | implement | 重新进入 implement 修复 |
| validate (pass) | report | 仅 large 任务建议 |

---

## 子 Agent 清单

| Agent | 文件 | 职责 | 产出 |
|-------|------|------|------|
| triage | `agents/triage.md` | 需求采集、规模判定 | `00-intake.md` |
| spec-writer | `agents/spec-writer.md` | 功能规格定义 | `01-spec.md` |
| planner | `agents/planner.md` | 技术方案设计 | `02-plan.md` |
| task-decomposer | `agents/task-decomposer.md` | 任务拆解（INVEST 原则） | `03-tasks.md` |
| implementer | `agents/implementer.md` | 代码实施 | `04-implementation-log.md` + 代码变更 |
| reviewer | `agents/reviewer.md` | 代码审查 | 审查结论 |
| validator | `agents/validator.md` | 验收标准核验 + 测试 | `05-validation.md` |

---

## Artifact 命名与目录

所有 feature 产物放在 `features/{id}-{name}/` 目录下：

- `00-intake.md`
- `01-spec.md`
- `02-plan.md`
- `03-tasks.md`
- `04-implementation-log.md`
- `05-validation.md`
- `06-report.md`（可选）

模板在 `templates/` 目录下。

---

## 需求

用户需求：$ARGUMENTS
