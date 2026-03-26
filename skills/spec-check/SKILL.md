---
name: spec-check
description: 审查 spec 文档的质量。检查验收标准是否可测试、边界情况是否完整、范围是否合理。在 spec 编写后调用。
user-invocable: false
allowed-tools: Read
---

# Spec Quality Check

审查 spec 文档质量，确保满足 SDD 标准。

## 输入

- 当前 feature 目录下的 `01-spec.md`

## 检查项

### 1. 验收标准（AC）质量
- [ ] 每条 AC 是否具体（不是"应该更好"这类模糊表述）
- [ ] 每条 AC 是否可测试/可验证
- [ ] AC 是否覆盖了核心行为

### 2. 边界情况
- [ ] 是否列出了边界情况
- [ ] 边界情况是否有期望处理方式

### 3. 范围
- [ ] 范围是否合理（不过大）
- [ ] 非目标是否明确
- [ ] 是否有范围蔓延的迹象

### 4. 篇幅
- [ ] 是否 human-reviewable（建议不超过 2-3 页）
- [ ] 是否有冗余内容

## 输出格式

```
Spec 质量检查: [feature-name]
总评: good / needs-improvement / poor
问题:
- [具体问题及改进建议]
```
