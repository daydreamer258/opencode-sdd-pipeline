---
name: stage-gate
description: 检查是否满足进入下一阶段的条件。验证前置 artifact 是否存在且关键字段非空。在阶段切换时由 Orchestrator 调用。
user-invocable: false
allowed-tools: Read, Glob
---

# Stage Gate Check

检查当前 feature 是否满足进入下一阶段的条件。

## 输入

- 当前 feature 目录路径
- 目标阶段名称

## 检查项

### 通用检查
1. 前置 artifact 文件是否存在
2. 文件是否非空（不是未填写的空模板）
3. 关键字段是否已填写

### 阶段特定检查

| 目标阶段 | 前置文件 | 关键字段 |
|---------|---------|---------|
| spec | `00-intake.md` | 问题定义、目标、规模判定 |
| plan | `01-spec.md` | 问题陈述、验收标准 |
| tasks | `02-plan.md` | 技术方案、涉及模块、风险评估 |
| implement | `03-tasks.md` | 任务列表（至少 1 个任务） |
| validate | `04-implementation-log.md` | 至少 1 个任务执行记录 |

## 输出格式

```
Stage Gate: [目标阶段]
状态: pass / fail
检查结果:
- [x] 前置文件存在
- [x] 关键字段非空
- [ ] 缺失: [具体说明]
```
