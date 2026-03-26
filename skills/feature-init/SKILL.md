---
name: feature-init
description: 初始化 feature 工作目录。在 features/ 下创建新目录，从 .opencode/templates/ 复制所有模板文件并替换占位符。
user-invocable: true
argument-hint: "[feature-id] [feature-name]"
allowed-tools: Bash, Read, Write
---

# Feature Init

初始化一个新 feature 的工作目录。

## 使用方式

```
/feature-init 002 user-search
```

## 行为

1. 在 `features/` 下创建 `$0-$1/` 目录
2. 从 `.opencode/templates/` 复制所有模板文件（00-06）
3. 替换模板中的 `{{feature-name}}` 为 `$1`

## 执行

```bash
FEATURE_DIR="features/$0-$1"
mkdir -p "$FEATURE_DIR"

for template in .opencode/templates/*.md; do
  filename=$(basename "$template")
  sed "s/{{feature-name}}/$1/g" "$template" > "$FEATURE_DIR/$filename"
done

echo "Feature directory initialized: $FEATURE_DIR"
ls -la "$FEATURE_DIR"
```

Feature ID: $0
Feature Name: $1
