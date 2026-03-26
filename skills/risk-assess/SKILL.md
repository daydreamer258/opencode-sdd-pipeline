---
name: risk-assess
description: 评估 implement 阶段的风险等级。分析 tasks 涉及的文件范围、改动类型，输出 low/medium/high 风险等级和理由。在 implement 前由 Orchestrator 调用。
user-invocable: false
allowed-tools: Read, Glob, Grep
---

# Risk Assessment

评估即将进入 implement 阶段的风险等级。

## 输入

- 当前 feature 目录下的 `03-tasks.md`
- 当前 feature 目录下的 `02-plan.md`（风险评估部分）

## 评估维度

| 维度 | Low | Medium | High |
|------|-----|--------|------|
| 变更文件数 | 1-3 个 | 4-10 个 | 10+ 个 |
| 核心模块 | 不涉及 | 间接涉及 | 直接修改 |
| 公共接口 | 不变 | 新增 | 修改/删除 |
| 数据变更 | 无 | 配置变更 | Schema/迁移 |
| 外部依赖 | 不变 | 新增 | 升级/删除 |
| 不可逆操作 | 无 | 有但可恢复 | 不可恢复 |

## 判定规则

- 所有维度均为 Low → **Low**
- 任一维度为 High → **High**
- 其他情况 → **Medium**

## 输出格式

```
风险等级: low / medium / high
理由:
- 维度1: 评估结果
- 维度2: 评估结果
建议: proceed / checkpoint / stop-and-review
```
