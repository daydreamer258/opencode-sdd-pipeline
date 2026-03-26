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

## 流程

收到用户需求后，按以下步骤推进：

1. **初始化**：调用 `feature-init` skill，在 `features/` 下创建 feature 目录
2. **Intake**：调用 Triage Agent 迭代循环生成 `00-intake.md`，评估任务规模
3. **用户定稿** intake 后，根据规模路由：
   - `small`：跳到步骤 8
   - `medium` / `large`：继续
4. **Spec**：调用 Spec Writer Agent 迭代循环生成 `01-spec.md`，然后调用 `spec-check` skill 检查质量
5. **Plan**：调用 Planner Agent 迭代循环生成 `02-plan.md`
6. **Tasks**：调用 Task Decomposer Agent 迭代循环生成 `03-tasks.md`
7. **Checkpoint**：调用 `stage-gate` skill 检查前置条件，调用 `risk-assess` skill 评估风险，medium/high 风险等待用户确认
8. **Implement**：调用 Implementer Agent 执行代码改动
9. **Review**：调用 Reviewer Agent 审查代码变更（Reviewer 内部调用 `diff-review` skill）
10. **Validate**：调用 Validator Agent 核验实现
11. **完成**（large 任务生成 `06-report.md`）

每个阶段完成后向用户展示摘要，等待确认后再继续。

## 关键规则

- 所有 agent 定义在 `.opencode/agents/` 目录下（规则直接写在 agent 正文中）
- 所有模板在 `.opencode/templates/` 目录下
- 所有 skill 在流程中显式调用

## 需求

用户需求：$ARGUMENTS
