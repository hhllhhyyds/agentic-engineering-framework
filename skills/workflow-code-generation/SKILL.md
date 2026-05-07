---
name: workflow-code-generation
description: 代码文件修改的统一入口。当用户请求任何代码变更（新功能、优化、Bug 修复、重构）时必须首先调用此 skill。仅适用于代码文件（如 .cc/.cpp/.h/.go/.py 等），修改 .md 等非代码文件时不需要调用。它会评估复杂度、检查 spec.md、生成 tasks.md、并逐个任务执行。
---

# 代码生成

> 所有代码修改的统一入口。**先加载规范，再写代码。**

## 工作流程

### 步骤 1：评估复杂度与路径路由

> ⚠️ **防御性检查**：如果 AI 无法明确回答"要改什么模块/文件"、"要实现什么行为"、"怎样算完成"这三个问题中的任何一个，则**立即停止**，调用 `workflow-requirements-clarification`。本 skill 不负责需求澄清。

**默认走标准流程**（步骤 2 → 3 → 4 → 5）。Fast-Path 仅当改动为**单文件、局部改动**（能在路由阶段确定完整文件列表，且只涉及 1 个文件的局部修改）时才可进入。

> **无法确定 → 标准流程。**

#### Fast-Path 执行

1. 加载编码规范（同步骤 4）
2. 实现改动
3. 执行过程中发现实际需要改多个文件 → **立即退出**，回到步骤 2 进入标准流程
4. 询问用户是否需要 code review
   - **需要** → 加载 `workflow-code-review` skill（指定 `skip_reviewers: [magical-prompt-reviewer]`）→ 修复循环
   - **不需要** → 输出改动说明
5. **结束**，不进入后续步骤

#### 标准流程入口

查找 `docs/design-docs/<module>/<feature>/spec.md`：
- 存在且完整 → **仔细通读全文**（不要只读关注的章节），确认理解
- 不存在/不完整 → **调用 `workflow-requirements-clarification`**（禁止自行澄清）

**强制**：`spec.md` 与 `tasks.md` 同时存在时，编码前必须完整读取两者。

#### TDD 模式判定

TDD（Test-Driven Development）模式即"先写失败测试，再实现通过"——适用于核心逻辑、复杂业务规则等需要精确规格验证的任务。

**启用条件**（任一满足即启用）：
- 用户明确要求 TDD
- 任务涉及核心数据结构、算法逻辑、业务规则，且 spec 中的验收标准可用测试表达
- 任务风险高（如影响核心路径、数据正确性）

**不启用条件**（任一满足即不启用）：
- Fast-Path 任务（单文件局部改动）
- 纯重构（不改变行为，仅改变结构）
- 用户明确要求不启用

判定结果在步骤 5 中控制执行流程。

### 步骤 2：从 tasks.md 选定当前任务

从 `tasks.md` 找第一个未完成任务，记录其编号。然后读取 `dev_plan/<N>-<name>.md` 获取该任务的完整上下文（目标、涉及文件、依赖、context、验收标准）。

### 步骤 3：检查/创建 tasks.md + dev_plan/

- **已存在** → 读取 `tasks.md` 任务总览，再读当前任务的 `dev_plan/<N>-<name>.md` 获取上下文，进入步骤 4
- **不存在** → **先读取** [reference/task_planning_guide.md](reference/task_planning_guide.md)，然后严格按其流程创建 `tasks.md`（概览）和 `dev_plan/`（各任务详情）

> 🚨 **创建任务清单后必须停下来等用户确认。** 展示 `tasks.md` 中的任务总览和 `dev_plan/` 结构，然后**停止并等待用户回复**。禁止自动进入步骤 4。

### 步骤 4：加载编码规范（🚨 强制前置）

> **未加载规范就写代码 → 立即停止，先加载。**

#### 必须加载

| 规范 | 说明 |
|------|------|
| `bp-coding-best-practices` | 通用编码最佳实践 |
| `bp-performance-optimization` | 性能优化（所有代码都是性能敏感的） |

#### 按需加载

| 规范 | 何时加载 |
|------|----------|
| `std-cpp` | `.cc`/`.cpp`/`.h` 文件 |
| `std-go` | `.go` 文件 |
| `bp-distributed-systems` | 涉及网络通信、多节点协调、一致性、故障恢复 |

### 步骤 5：逐个任务实现

**核心规则：一个 Task → review 通过 → 报告 → 等用户批准 → 下一个 Task**

#### TDD 模式（启用时执行）

在 Phase 1 之前插入 Red 阶段：

**Phase 0a: 理解上下文** — 读取 spec 和当前 task 描述，确定接口的预期行为、输入/输出、边界条件、需 mock 的外部依赖。

**Phase 0b: 生成失败测试（Red）** — 基于 spec 和 task 描述编写测试（不基于实现——此时代码不存在）。测试必须可编译但断言失败，覆盖正常路径 + 边界条件 + 异常场景。

**Phase 0c: 暂停等确认** — 展示测试代码等待用户审查。确认 → 进入 Phase 1（Green）；要求修改 → 修改后重等；取消 TDD → 回退非 TDD 模式。

---

#### Phase 1: 实现（TDD = Green / 非 TDD = 直接实现）

- **TDD 模式**：实现最小代码让测试全部通过
- **非 TDD 模式**：直接按 spec 和 task 描述实现
- 更新 tasks.md 标记 In Progress

#### Phase 2: Code Review 与修复循环（🚨 强制）

**2a. 执行 Code Review**

加载 `workflow-code-review` skill，按其工作流执行（指定 `skip_reviewers: [magical-prompt-reviewer]`）。该 skill 会并行调用 reviewer subagent 执行审查——**禁止主 agent 自己做 review 代替 subagent**。

**2b. 逐条反思犯错原因**

对报告中**每条** keep 的 finding，反思犯错原因：

| 原因分类 | 含义 |
|---------|------|
| **Spec 理解偏差** | spec 写清楚了但理解错误 |
| **规范未遵守** | 编码规范有要求但未执行 |
| **执行遗漏** | 漏掉边界/细节 |
| **设计考虑不足** | 需更深层设计思考 |

**2c. 自动修复 → Re-review 循环**

修复 finding → 重新加载 `workflow-code-review` skill 执行 re-review → 重复 2b-2c，直到结论为 PASS。

中间轮次的报告和反思不需要展示给用户，内部处理即可。

**2d. 输出 Review 总结**

所有轮次结束（PASS）后，向用户输出一份总结，包含：
- 经历了几轮 review
- 每条发现的问题、修复方式和犯错原因
- 最终 PASS 的 review 报告

#### Phase 2.5: 文档同步

Review 通过后，执行文档同步确保代码与文档一致。读取 [reference/document_sync_guide.md](reference/document_sync_guide.md) 并按自下而上扫描算法执行：

1. **代码本身** — 更新新增/修改的公共接口 doc comment
2. **模块文档** — 检查并更新模块说明和设计文档
3. **跨模块影响** — 检查依赖、接口契约、配置项等波及面
4. **项目级影响** — 检查架构原则、技术栈文档是否需更新

完成后在 task 报告中包含「文档同步」结果表。

#### Phase 3: 汇报 → 停止等待

输出报告，更新 tasks.md 标记 Completed，**停止等待用户批准**。

#### Task 完成报告模板

```markdown
---
## ✅ Task N 完成

### 改动文件
- `path/to/file.cc`: [改动说明]

### 改动内容
[做了什么，为什么]

### Code Review 结果

**Review 轮次**: [第 K 轮通过]
[附 workflow-code-review 输出的报告]

### 文档同步

| 文档 | 操作 | 说明 |
|------|------|------|
| `path/to/doc.md` | 更新 | [变更摘要] |
| `path/to/another.md` | 无需更新 | [判断理由] |

---

询问用户下一步操作，提供以下选项：
- **继续/LGTM** → 下一个任务
- **修改** → 重新提交（用户附带修改要求）
- **回滚** → 撤销本次改动
```

---

## 用户跳过 spec 时

必须生成简化版 spec.md（标注 `Status: Quick Draft`）。**禁止无 spec 修改中等+代码。**

## 恢复中断的任务

读取 `tasks.md` → 找未完成任务 → **重新加载编码规范** → 继续执行。
