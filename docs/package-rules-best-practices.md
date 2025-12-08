# Renovate PackageRules 配置最佳实践

本文档总结了 Renovate PackageRules 的配置规则和最佳实践，基于 Renovate 官方文档（2025年版本）。

## 目录

- [核心概念](#核心概念)
- [规则评估顺序](#规则评估顺序)
- [最佳实践](#最佳实践)
- [常见问题](#常见问题)
- [配置示例](#配置示例)

## 核心概念

### 什么是 PackageRules

PackageRules 允许你对特定的依赖项应用不同的 Renovate 设置。通过 PackageRules，你可以实现：
- 为特定包启用自动合并
- 为不同类型的更新设置不同的标签
- 设置自定义的更新计划
- 分组管理相关依赖的更新

### 规则结构要求

**每个 PackageRule 必须包含：**
1. **至少一个匹配条件**（以 `match*` 开头的属性）
2. **至少一个操作属性**（如 `enabled`、`automerge`、`labels` 等）

```json5
// ❌ 错误：只有匹配条件，没有操作属性
{"matchDatasources": ["golang-version"], "rangeStrategy": "bump"}

// ✅ 正确：既有匹配条件，又有操作属性
{"matchDatasources": ["golang-version"], "rangeStrategy": "bump", "enabled": true, "automerge": true}
```

### 常用匹配条件

| 匹配器 | 用途 | 示例 |
|--------|------|------|
| `matchPackageNames` | 精确匹配包名 | `["golang", "k8s.io/client-go"]` |
| `matchPackagePatterns` | 正则匹配包名 | `["^@types/.*", "k8s.io/**"]` |
| `matchDatasources` | 匹配数据源 | `["docker", "go", "npm"]` |
| `matchUpdateTypes` | 匹配更新类型 | `["major", "minor", "patch"]` |
| `matchManagers` | 匹配包管理器 | `["gomod", "npm"]` |

### 常用操作属性

| 属性 | 用途 | 示例 |
|------|------|------|
| `enabled` | 启用/禁用更新 | `true` / `false` |
| `automerge` | 自动合并 PR | `true` / `false` |
| `groupName` | 分组更新 | `"golang packages"` |
| `addLabels` | 添加标签 | `["security"]` |

## 规则评估顺序

### 核心原则

1. **顺序处理**：Renovate 按照配置数组中的顺序依次评估规则
2. **后者优先**：当多个规则匹配同一个包时，**后面的规则会覆盖前面规则的设置**
3. **AND 逻辑**：单个规则内的所有匹配条件使用 AND 逻辑，必须全部满足才会触发

### ⚠️ 重要：Renovate 的多阶段处理机制

Renovate 采用**多阶段处理流程**，这对规则配置有重要影响：

```text
1. Extract（提取阶段）
   ↓ 发现并提取依赖项
   ↓ packageRules 在此阶段首次应用（基于 matchDatasources、matchPackageNames 等）

2. Lookup（查找阶段）
   ↓ 查找可用的新版本
   ↓ 确定 updateType (major/minor/patch/digest)
   ↓ packageRules 在此阶段再次应用（现在 matchUpdateTypes 可以生效）

3. Apply（应用阶段）
   ↓ 决定是否创建 PR、是否自动合并等
```

**关键点：`matchUpdateTypes` 只在 Lookup 阶段之后才能生效！**

这意味着：

- 如果在 Extract 阶段依赖被 `enabled: false` 禁用，它将不会进入 Lookup 阶段
- 后续使用 `matchUpdateTypes` 的规则将无法重新启用该依赖
- `updateType` 信息在 Extract 阶段**不存在**，只在 Lookup 阶段才会被初始化

### 实际示例：为什么需要分开配置

#### ❌ 错误配置

```json5
{
  "packageRules": [
    {"matchPackagePatterns": [".*"], "enabled": false},
    {"matchDatasources": ["golang-version"], "matchUpdateTypes": ["minor", "patch"], "enabled": true}  // ❌ 不会生效
  ]
}
```

**原因：**第一条规则在 Extract 阶段禁用了 golang-version，永远不会到达 Lookup 阶段初始化 updateType。

#### ✅ 正确配置

```json5
{
  "packageRules": [
    {"matchPackagePatterns": [".*"], "enabled": false},
    {"matchDatasources": ["golang-version"], "enabled": true},  // Extract 阶段重新启用
    {"matchDatasources": ["golang-version"], "matchUpdateTypes": ["minor", "patch"], "enabled": true, "automerge": true}  // Apply 阶段过滤
  ]
}
```

### 推荐的规则顺序

```
1. 广泛的排除规则（默认禁用所有）
2. 启用特定数据源（Extract 阶段生效）
3. 禁用特定更新类型（Apply 阶段生效）
4. 特定场景的白名单规则（Apply 阶段生效）
5. 分组配置
```

### 一般规则示例

```json5
{
  "packageRules": [
    {"matchPackagePatterns": [".*"], "enabled": false},  // 默认禁用
    {"matchDatasources": ["docker", "go"], "enabled": true},  // Extract 阶段启用
    {"matchDatasources": ["docker"], "matchUpdateTypes": ["patch"], "enabled": true, "automerge": true}  // Apply 阶段过滤
  ]
}
```

**结果：**Docker patch 更新启用且自动合并，Docker major/minor 和 Go 更新仅启用不自动合并。

## 最佳实践

### 1. 使用 `config:best-practices` 预设

Renovate 提供了 `config:best-practices` 预设，包含了最新的最佳实践：

```json5
{
  "extends": [
    "config:best-practices",  // 推荐使用
    // "config:recommended"   // 旧版推荐
  ]
}
```

**`config:best-practices` 包含：**
- `config:recommended`
- `docker:pinDigests` - 固定 Docker 镜像 digest
- `helpers:pinGitHubActionDigests` - 固定 GitHub Actions digest
- `:configMigration` - 自动迁移配置
- `:pinDevDependencies` - 固定开发依赖
- `abandonments:recommended` - 检测废弃的包

**推荐等待时间：**
- Patch 更新：3-7 天
- Minor 更新：7-14 天
- 第三方依赖：14 天（官方推荐）

### 2. 明确禁用而非依赖默认行为

遵循"**显式优于隐式**"的原则：

```json5
// ❌ 不推荐
{"matchUpdateTypes": ["major"]}  // 依赖默认行为

// ✅ 推荐
{"matchUpdateTypes": ["major"], "enabled": false}  // 显式声明
```

### 3. 使用 `addLabels` 而非 `labels`

- `labels` 会**替换**之前的标签
- `addLabels` 会**追加**标签

```json5
{
  "packageRules": [
    {
      "matchPackagePatterns": [".*"],
      "labels": ["dependencies"]  // 设置基础标签
    },
    {
      "matchUpdateTypes": ["major"],
      "addLabels": ["breaking-change"]  // 追加标签，不覆盖 "dependencies"
    }
  ]
}
```

### 4. 排除规则放在前面

使用负向匹配或排除模式时，**将排除规则放在配置数组的前面**：

```json5
{
  "packageRules": [
    // ✅ 排除规则放前面
    {
      "matchPackagePatterns": ["^@types/.*"],
      "enabled": false
    },

    // ✅ 特定的覆盖规则放后面
    {
      "matchPackageNames": ["@types/node"],
      "enabled": true  // 覆盖上面的排除规则
    }
  ]
}
```

### 5. 合理分组相关依赖

将相关的依赖分组，减少 PR 数量：

```json5
{
  "packageRules": [
    // Kubernetes 生态系统分组
    {
      "matchPackagePatterns": [
        "^k8s.io/.*",
        "^sigs.k8s.io/.*"
      ],
      "groupName": "kubernetes packages",
      "groupSlug": "k8s"
    },

    // Golang 官方包分组
    {
      "matchPackagePatterns": ["^golang.org/x/.*"],
      "groupName": "golang official packages",
      "groupSlug": "go-official"
    }
  ]
}
```

### 6. 为不同类型的更新设置不同策略

```json5
{
  "packageRules": [
    // Major 更新：需要手动审批
    {
      "matchUpdateTypes": ["major"],
      "enabled": true,
      "dependencyDashboardApproval": true
    },

    // Minor 更新：自动创建 PR，但不自动合并
    {
      "matchUpdateTypes": ["minor"],
      "enabled": true
    },

    // Patch 更新：自动合并
    {
      "matchUpdateTypes": ["patch"],
      "enabled": true,
      "automerge": true
    }
  ]
}
```

### 7. 避免过度使用通配符

```json5
// ❌ 不推荐
{"matchPackageNames": ["*"], "enabled": false}  // 可能不按预期工作

// ✅ 推荐
{"matchPackagePatterns": [".*"], "enabled": false}  // 使用正则表达式
{"enabled": false}  // 或省略匹配器（默认匹配所有）
```

## 常见问题

### 问题 1：为什么使用 `matchUpdateTypes` 的规则没有生效？

这是最常见的配置错误。由于 `updateType` 只在 Lookup 阶段初始化，如果依赖在 Extract 阶段被禁用，就永远不会到达 Lookup 阶段。

**解决方案：**需要添加中间规则先启用数据源（详见[多阶段处理机制](#️-重要renovate-的多阶段处理机制)）

```json5
{
  "packageRules": [
    {"matchPackagePatterns": [".*"], "enabled": false},
    {"matchDatasources": ["docker"], "enabled": true},  // Extract 阶段启用
    {"matchDatasources": ["docker"], "matchUpdateTypes": ["patch"], "enabled": true, "automerge": true}  // Apply 阶段过滤
  ]
}
```

### 问题 2：为什么我的规则没有生效？

**可能原因：**
1. **缺少操作属性**：只有匹配条件，没有 `enabled`、`automerge` 等操作属性
2. **规则顺序错误**：后面的规则覆盖了前面的设置
3. **匹配条件不正确**：使用了错误的匹配器或模式
4. **`matchUpdateTypes` 时机问题**：参见问题 1

**解决方案：**
```json5
// ❌ 错误
{"matchDatasources": ["golang-version"], "rangeStrategy": "bump"}

// ✅ 正确
{"matchDatasources": ["golang-version"], "rangeStrategy": "bump", "enabled": true}
```

### 问题 3：规则之间冲突怎么办？

**理解覆盖机制：**
- 后面的规则会覆盖前面规则的**相同属性**
- 不同的属性会**合并**

```json5
{
  "packageRules": [
    {
      "matchDatasources": ["docker"],
      "enabled": true,
      "labels": ["docker"]
    },
    {
      "matchDatasources": ["docker"],
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "addLabels": ["automerge"]  // 使用 addLabels 追加标签
    }
  ]
}
```

对于 Docker patch 更新，最终配置为：
- `enabled: true` （从第一条规则继承）
- `labels: ["docker"]` （从第一条规则继承）
- `automerge: true` （第二条规则新增）
- 标签变为：`["docker", "automerge"]`

### 问题 4：如何禁用 Digest 更新？

Digest 更新通常由 Renovate 自动触发，可能导致不必要的更新：

```json5
{
  "packageRules": [
    // 禁用 Go 依赖的 digest 更新
    {
      "matchDatasources": ["go"],
      "matchUpdateTypes": ["digest"],
      "enabled": false
    }
  ]
}
```

### 问题 5：如何仅允许安全更新？

使用 Renovate 的安全预设：

```json5
{
  "packageRules": [
    {
      "matchBaseBranches": ["release-*", "main"],
      "extends": ["security:only-security-updates"],
      "enabled": true,
      "automerge": true
    }
  ]
}
```

## 配置示例

### 示例 1：保守的生产环境配置

```json5
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:best-practices"],
  "packageRules": [
    // 默认禁用所有更新
    {
      "matchPackagePatterns": [".*"],
      "enabled": false
    },

    // 仅允许安全更新
    {
      "extends": ["security:only-security-updates"],
      "enabled": true,
      "automerge": true
    },

    // 对于开发依赖，允许 patch 更新
    {
      "matchDepTypes": ["devDependencies"],
      "matchUpdateTypes": ["patch"],
      "enabled": true,
      "automerge": true
    },

    // Major 更新需要手动审批
    {
      "matchUpdateTypes": ["major"],
      "enabled": true,
      "dependencyDashboardApproval": true,
      "labels": ["major-update", "needs-review"]
    }
  ]
}
```

### 示例 2：积极的开发环境配置

```json5
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:best-practices"],
  "packageRules": [
    // 启用所有数据源
    {
      "matchDatasources": ["npm", "docker", "go"],
      "enabled": true
    },

    // Patch 和 Minor 自动合并
    {
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true
    },

    // Major 更新创建 PR 但不自动合并
    {
      "matchUpdateTypes": ["major"],
      "automerge": false
    },

    // 分组所有 patch 更新
    {
      "matchUpdateTypes": ["patch"],
      "groupName": "all patch updates",
      "groupSlug": "patch-updates"
    }
  ]
}
```

### 示例 3：多环境配置

基于分支的多环境配置，详见 [examples/multi-environment-config.json5](examples/multi-environment-config.json5)

## 参考资源

- [Renovate Package Rules 官方文档](https://docs.renovatebot.com/configuration-options/#packagerules)
- [Renovate Package Rules Guide](https://docs.mend.io/wsk/renovate-package-rules-guide)
- [Common Practices for Renovate Configuration](https://docs.mend.io/wsk/common-practices-for-renovate-configuration)
- [Renovate Best Practices](https://docs.renovatebot.com/upgrade-best-practices/)
- [Renovate Presets](https://docs.renovatebot.com/presets-default/)

## 更新历史

- 2025-12-08: 初始版本，基于 Renovate 2025 年最新文档
- 2025-12-08: 添加 Renovate 多阶段处理机制的详细说明（Extract → Lookup → Apply）
- 2025-12-08: 优化文档结构，精简代码示例，优化表格内容，将示例 3 移至单独文件
