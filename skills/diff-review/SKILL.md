---
name: diff-review
description: 审查代码变更与 spec/tasks 的一致性。对比 git diff 与 tasks 定义的变更范围，检查是否有超范围或遗漏。在 validate 阶段调用。
user-invocable: false
allowed-tools: Read, Glob, Grep, Bash
---

# Diff Review

审查代码变更与 spec/tasks 的一致性。

## 输入

- 当前 feature 目录下的 `03-tasks.md`
- 当前 feature 目录下的 `04-implementation-log.md`
- `git diff` 输出

## 检查流程

1. 读取 `03-tasks.md` 中每个任务的变更范围
2. 运行 `git diff --stat` 查看实际变更文件列表
3. 对比两者：

### 一致性检查

| 检查项 | 说明 |
|--------|------|
| 超范围变更 | 是否有不在 tasks 中但实际改动的文件 |
| 遗漏变更 | 是否有 tasks 中定义但未改动的文件 |
| 变更量合理性 | 变更行数是否与任务复杂度匹配 |

### 内容检查

- 是否引入了不在 spec 中的新功能
- 是否遗漏了 spec 中的关键行为
- 是否有明显的代码质量问题

## 输出格式

```
Diff Review: [feature-name]
实际变更文件: [数量]
Tasks 定义文件: [数量]

一致性:
- 超范围变更: [有/无] — [详情]
- 遗漏变更: [有/无] — [详情]
- 计划外功能: [有/无] — [详情]

结论: consistent / has-deviations
```
