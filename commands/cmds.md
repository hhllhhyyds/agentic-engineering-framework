列出框架所有可用 command 和 skill，输出格式清晰的表格让用户一目了然。

## Commands（用户入口）

| Command | 加载的 Skill | 用途 |
|---------|------------|------|
| `/requirements-clarification` | `workflow-requirements-clarification` | 需求澄清，生成 spec.md |
| `/system-design` | `workflow-system-design` | 系统设计，填写 spec 设计章节 |
| `/code-generation` | `workflow-code-generation` | 代码实现（含 TDD 模式） |
| `/test-generation` | `workflow-test-generation` | 测试生成（单元/集成/性能） |
| `/code-review` | `workflow-code-review` | 多 Agent 代码审查 |
| `/troubleshooting` | `troubleshooting` | 编译错误、运行异常、现网告警排查 |
| `/performance-optimization` | `bp-performance-optimization` | 性能分析与优化 |
| `/reflect` | `self-refinement` | 回顾对话，沉淀错误经验 |
| `/skill-authoring` | `bp-skill-authoring` | 编写新 Skill 的指导 |
| `/cmds` | — | 显示本列表 |

## Skills（按需加载的知识和流程）

### Workflow（端到端流程）
- `workflow-requirements-clarification` — 需求澄清
- `workflow-system-design` — 系统设计
- `workflow-code-generation` — 代码实现
- `workflow-test-generation` — 测试生成
- `workflow-code-review` — 多 Agent 代码审查

### Best Practices（通用工程知识）
- `bp-coding-best-practices` — 编码最佳实践
- `bp-architecture-design` — 架构设计原则
- `bp-component-design` — 组件设计
- `bp-distributed-systems` — 分布式系统
- `bp-performance-optimization` — 性能优化
- `bp-skill-authoring` — Skill 编写指导

### Standards（项目编码规范）
- `std-cpp` — C++ 编码规范
- `std-go` — Go 编码规范
- `std-rust` — Rust 编码规范

### 工具型
- `troubleshooting` — 问题排查
- `self-refinement` — 经验沉淀反馈闭环
