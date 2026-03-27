---
name: sdd-workflow
description: SDD 工作流主入口。启动 Orchestrator 驱动完整的 spec-driven development 流程：intake → spec → plan → tasks → implement → review → validate。当用户想要按 SDD 流程开发新 feature 时使用。
user-invocable: true
argument-hint: "[需求描述]"
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, Agent
---

# SDD Workflow

启动 SDD（Spec-Driven Development）工作流。

## 使用方式

```
/sdd-workflow 实现一个搜索功能，支持关键词匹配和结果排序
```

## ⚠️ 强制规则（Orchestrator 必须遵守）

1. **每个阶段结束后，必须向用户展示结果并等待用户明确确认，才能进入下一阶段。** 用户未确认 = 不可继续。不可以自行判断"应该没问题"然后跳过确认。
2. **每个阶段的产出必须由对应的 SubAgent 生成。** Orchestrator 绝对不可以自己编写或修改制品文件（`00-intake.md` 到 `06-report.md`），也不可以替代 SubAgent 完成任何工作。这是最重要的规则，没有例外。
3. **SubAgent 超步数退出时，必须启动新的同类型 SubAgent 续接，不可以自己接手。**
4. **阶段切换前必须调用 `stage-gate` skill 检查前置条件。**

## 流程

收到用户需求后，按以下步骤推进：

1. **初始化**：调用 `feature-init` skill，在 `features/` 下创建 feature 目录
2. **Intake**：调用 Triage Agent 迭代循环生成 `00-intake.md`，评估任务规模
3. **→ 向用户展示 intake 摘要 + 待澄清问题，等待用户确认定稿** 后，根据规模路由：
   - `small`：跳到步骤 8
   - `medium` / `large`：继续
4. **Spec**：调用 Spec Writer Agent 迭代循环生成 `01-spec.md`，然后调用 `spec-check` skill 检查质量
5. **→ 向用户展示 spec 摘要 + spec-check 结果，等待用户确认定稿**
6. **Plan**：调用 Planner Agent 迭代循环生成 `02-plan.md`
7. **→ 向用户展示 plan 摘要 + 技术取舍，等待用户确认定稿**
8. **Tasks**：调用 Task Decomposer Agent 迭代循环生成 `03-tasks.md`
9. **→ 向用户展示 tasks 摘要，等待用户确认定稿**
10. **Checkpoint**：调用 `stage-gate` skill 检查前置条件，调用 `risk-assess` skill 评估风险，medium/high 风险等待用户确认
11. **Implement**：调用 Implementer Agent 执行代码改动
12. **→ 向用户展示实施摘要，等待用户确认**
13. **Review**：调用 Reviewer Agent 审查代码变更（Reviewer 内部调用 `diff-review` skill）
14. **→ 向用户展示审查结论，等待用户确认**
15. **Validate**：调用 Validator Agent 核验实现
16. **→ 向用户展示验证结论，等待用户确认**
17. **完成**（large 任务生成 `06-report.md`）

## 关键规则

- 所有 agent 定义在 `.opencode/agents/` 目录下（规则直接写在 agent 正文中）
- 所有模板在 `.opencode/templates/` 目录下
- 所有 skill 在流程中显式调用
- **每个 → 标记的步骤都是用户确认点，不可跳过**

## 需求

用户需求：$ARGUMENTS
