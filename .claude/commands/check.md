---
description: 检查 Renovate 配置文件是否符合最佳实践
allowed-tools: Read
argument-hint: [config-file-path]
---

## 任务说明

检查用户指定的配置文件 `$1`，基于 `docs/package-rules-best-practices.md` 文档验证是否符合 Renovate 最佳实践。

## 配置文件说明

根据 @README.md，本仓库包含以下配置文件：
- **core.json5**: 与策略无关的配置，如自定义数据源（customDatasources）和自定义管理器（customManagers）
- **team-devops.json5**: DevOps 团队的自动修复策略配置文件
- **base.json**: 遗留配置文件，完全继承 devops.json5，保留用于兼容性

## 执行步骤

1. 读取配置文件 `$1`
2. 参考 @docs/package-rules-best-practices.md 的规则
3. 逐项对照检查清单进行验证
4. **重点检查 packageRules 的逻辑正确性**
5. 生成结构化的检查报告

## 检查清单

### 1. 规则结构验证

- [ ] 每个 PackageRule 包含至少一个匹配条件（`match*` 属性）
- [ ] 每个 PackageRule 包含至少一个操作属性（`enabled`、`automerge`、`labels` 等）
- [ ] 没有只有匹配条件而无操作属性的规则

### 2. **PackageRules 逻辑正确性（重点）**

- [ ] **多阶段处理机制**：`matchUpdateTypes` 的使用时机正确
  - Extract 阶段：依赖被发现和提取，此时 `updateType` 尚不存在
  - Lookup 阶段：查找新版本，初始化 `updateType` (major/minor/patch/digest)
  - Apply 阶段：决定是否创建 PR、是否自动合并
- [ ] **规则顺序合理性**：
  - 如果使用 `matchUpdateTypes`，确保对应数据源已在前面的规则中启用
  - 正确的顺序：广泛的排除规则 → 启用特定数据源 → 基于 updateType 的过滤
- [ ] **规则覆盖逻辑**：后面的规则会覆盖前面的相同属性，验证是否符合预期
- [ ] **匹配条件的 AND 逻辑**：单个规则内的所有 `match*` 条件必须全部满足才会触发

### 3. 标签使用

- [ ] 优先使用 `addLabels` 而非 `labels`（避免覆盖）
- [ ] `labels` 和 `addLabels` 使用顺序正确

### 4. 规则顺序

- [ ] 排除规则在前面
- [ ] 特定覆盖规则在后面
- [ ] 无意外覆盖配置

### 5. 通配符使用

- [ ] `matchPackageNames: ["*"]` 用于"默认禁止所有"的排除规则时是**正确的**
- [ ] 其他场景建议使用 `matchPackagePatterns: [".*"]` 进行正则匹配

### 6. 显式配置

- [ ] 所有规则显式声明 `enabled: true/false`
- [ ] 不依赖默认行为

### 7. 预设使用

- [ ] 使用 `config:best-practices` 或 `config:recommended` 预设

## 输出格式要求

生成以下格式的报告：

### 1. 配置概览

```
文件：$1
配置类型：[core.json5 | team-devops.json5 | base.json]
PackageRules 数量：X 条
预设配置：是否使用 best-practices/recommended
customDatasources：有/无
customManagers：有/无
```

### 2. **PackageRules 逻辑分析（重点）**

对每条规则进行逻辑分析，重点关注：

#### 规则执行流程模拟

- 模拟 Renovate 的三个阶段（Extract → Lookup → Apply）
- 分析每条规则在哪个阶段生效
- 验证规则之间的覆盖关系是否符合预期

#### 典型逻辑错误检测

检测以下常见逻辑错误：

1. **`matchUpdateTypes` 时机错误**
   ```json5
   // ❌ 错误示例
   {"matchPackagePatterns": [".*"], "enabled": false},
   {"matchDatasources": ["golang-version"], "matchUpdateTypes": ["patch"], "enabled": true}
   // 问题：第二条规则永远不会生效，因为数据源在 Extract 阶段已被禁用
   ```

2. **规则覆盖冲突**
   ```json5
   // ⚠️ 警告示例
   {"matchDatasources": ["docker"], "enabled": true, "automerge": true},
   {"matchDatasources": ["docker"], "matchUpdateTypes": ["major"], "automerge": false}
   // 分析：major 更新的 automerge 会被覆盖为 false，其他更新类型仍为 true
   ```

3. **缺少中间启用规则**
   ```json5
   // ❌ 错误：缺少中间规则
   {"matchPackagePatterns": [".*"], "enabled": false},
   {"matchDatasources": ["docker"], "matchUpdateTypes": ["patch"], "enabled": true}

   // ✅ 正确：添加中间规则
   {"matchPackagePatterns": [".*"], "enabled": false},
   {"matchDatasources": ["docker"], "enabled": true},  // 中间规则
   {"matchDatasources": ["docker"], "matchUpdateTypes": ["patch"], "automerge": true}
   ```

### 3. 问题列表

按严重程度分类：

- ❌ **严重问题**：逻辑错误，规则不会按预期工作，必须修复
- ⚠️ **警告**：可能存在风险或不符合最佳实践，建议改进
- 💡 **建议**：可以优化的地方，提升配置质量

每个问题包含：

- 问题类型（逻辑错误/安全问题/代码风格）
- 问题描述
- 所在位置（规则索引，如 `packageRules[0]`）
- 相关代码片段
- 影响范围（哪些包会受影响）

### 4. 修复建议

针对每个问题提供：

- **问题根因分析**：解释为什么会有这个问题
- **修复方案**：提供具体的修改建议
- **修改后的代码示例**：完整的代码片段
- **验证方法**：如何验证修复是否正确

### 5. 总结评价

- **整体评分**：通过 ✅ / 需要改进 ⚠️ / 存在严重问题 ❌
- **逻辑正确性评价**：规则执行流程是否合理
- **优先修复项**：按优先级排序的待修复问题
- **后续优化建议**：进一步改进的方向

## 特别注意

### 对于 core.json5

- `customDatasources` 和 `customManagers` **不适用** packageRules 标准检查
- 重点检查自定义数据源和管理器的配置正确性
- 验证正则表达式和 transformTemplates 的逻辑

### 对于 team-devops.json5

- 重点检查自动修复策略的 packageRules 逻辑
- 验证安全性配置（`automerge` 等）
- 分析规则覆盖关系是否符合预期

### 对于 base.json

- 检查是否正确继承 devops.json5
- 验证兼容性问题

### 通用注意事项

- 如果配置文件不存在，友好提示用户
- 提供可操作的建议，包含完整的代码示例
- 关注实际业务逻辑和安全性
- 使用表格或流程图辅助说明复杂的逻辑关系
