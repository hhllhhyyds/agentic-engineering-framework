# Rust 开发规范（完整版）

## 命名规范

| 项 | 规范 | 示例 |
|----|------|------|
| Types / Traits | `CamelCase` | `LlmClient`, `StoreError` |
| Functions / Methods | `snake_case` | `chat()`, `search_vectors()` |
| Variables | `snake_case` | `session_id`, `skill_name` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Modules | `snake_case` | `use crate::llm::client` |
| Enum variants | `CamelCase` | `Error::NotFound` |
| Type parameters | 短驼峰 | `<T>`, `<E>`, `<Ctx>` |
| Trait bound 缩写 | 约定俗成 | `AsRef`, `Into`, `Read`, `Write` |

模块文件使用 `foo.rs` 而非 `foo/mod.rs`。

## 代码组织

### 文件内顺序

```rust
// 1. 标准库
use std::collections::HashMap;

// 2. 外部依赖（空行分隔不同依赖组）
use anyhow::Result;
use serde::{Deserialize, Serialize};
use tokio::sync::mpsc;

// 3. 同 crate 内部
use crate::core::traits::LlmClient;
```

### 可见性

- 最小化 `pub` 暴露面。默认 `pub(crate)`，只在其他 crate 需要时放宽到 `pub`
- `pub` 类型和方法必须有 `///` doc comment（英文）
- 实现细节用 `pub(crate)` 或私有
- 辅助函数直接私有

### 模块结构惯例

```rust
// lib.rs — 公开 API 仅 re-export
mod client;
mod config;
mod error;

pub use client::LlmClient;
pub use config::Config;
pub use error::Error;

// client.rs — 实现细节
pub(crate) mod inner;  // 同 workspace 可见
```

## 错误处理

- **Library crate**（如 core、storage、llm）：用 `thiserror` 定义具体错误类型
- **Binary / App crate**（如 runtime、examples）：用 `anyhow` 聚合错误
- 核心 crate `core` 定义所有关键错误类型，其他 crate 引用或转换

```rust
// thiserror — 精确错误
#[derive(Debug, thiserror::Error)]
pub enum LlmError {
    #[error("API returned {status}: {message}")]
    Api { status: u16, message: String },

    #[error("rate limited, retry after {retry_after}s")]
    RateLimited { retry_after: u64 },

    #[error(transparent)]
    Io(#[from] std::io::Error),
}
```

- 库代码中 **禁用** `unwrap()`、`expect()`、`.ok()` 静默丢弃错误
- 仅允许场景：测试、`main()` 入口、确信不会失败的初始化（如 `Config::default()`）
- 错误传播用 `?` 操作符

## 异步编程

- 运行时统一使用 `tokio`
- 异步 trait 方法签名：

```rust
#[async_trait]
pub trait LlmClient: Send + Sync {
    async fn chat(&self, request: ChatRequest) -> Result<ChatResponse, LlmError>;
}
```

- 所有 trait 对象需要 `Send + Sync`（跨线程安全）
- 避免 `async_recursion`，优先重构为迭代模式

## 文档约定

- 文档语言：中文（设计文档、README、规范文件）
- 代码注释：**全部英文**（包括 `///` doc comment 和行内 `//` 注释）
- **所有 `pub` 类型、方法、trait、字段** 必须有 `///` doc comment
- 复杂 API 推荐在 doc comment 中包含使用示例：

```rust
/// Send a chat request to the LLM and return the response.
///
/// # Errors
///
/// Returns `LlmError::Api` if the API returns a non-2xx status.
/// Returns `LlmError::RateLimited` if the API rate-limits the request.
async fn chat(&self, request: ChatRequest) -> Result<ChatResponse, LlmError>;
```

## 测试

### 单元测试

每个 `.rs` 文件底部放置单元测试模块：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_default_config() {
        let config = Config::default();
        assert_eq!(config.timeout, 30);
    }
}
```

### 集成测试

- 在 crate 根目录的 `tests/` 目录下
- 文件命名：`test_<feature>.rs`
- 每个文件是一个独立的 crate

### Mock 策略

- 使用 `mockall` 为 trait 生成 mock
- Mock 放在 `tests/` 或 `#[cfg(test)]` 模块内，不进入生产代码
- 示例：

```rust
#[cfg(test)]
use mockall::mock;

mock! {
    pub LlmClient {
        async fn chat(&self, request: ChatRequest) -> Result<ChatResponse, LlmError>;
    }
}
```

### TDD 流程

遵循以下三阶段：

1. **Red**：先写会失败的测试，定义预期行为。测试可编译但断言失败
2. **Green**：实现最小代码让测试通过。不追求完美，追求可运行
3. **Refactor**：在有测试保护的前提下重构

### 运行检查

在提交前必须通过：

```bash
cargo test                    # 全部测试通过
cargo clippy -- -D warnings   # 无 lint 警告
cargo fmt --check             # 格式符合 rustfmt
```

## Commit 规范

格式：`<type>(<scope>): <description>`

| 类型 | 用途 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修复 |
| `docs` | 文档 |
| `test` | 测试（Red 阶段） |
| `refactor` | 重构 |
| `chore` | 构建/工具/CI |

示例：
```
feat(core): define MemoryLayer and Storage traits
test(llm): add integration tests for chat API
fix(storage): handle concurrent write with WAL mode
docs: update ARCHITECTURE.md with crate dependency graph
```

TDD 分阶段标记：
- Red（写测试）：`test(<scope>): add tests for #XX (<feature name>)`
- Green（实现）：`feat(<scope>): implement #XX (<feature name>)`

## Cargo 规范

### 依赖管理

- 公共依赖在 workspace 级别声明 `[workspace.dependencies]`
- 各 crate 通过 `crate_name.workspace = true` 引用
- 初版使用兼容范围（如 `tokio = "1"`），后期锁定具体版本

### Crate 清单

- 每个 crate 必须有 `description` 和 `version`
- Crate 名使用 kebab-case（`lithify-core` 而非 `lithify_core`）
- feature gate 保持最小集，不开启不需要的 feature

### 特性

- 测试用 `#[cfg(test)]` 而非 feature gate
- Mock 放在 `#[cfg(test)]` 内而非独立文件
- 可选依赖通过 feature 控制
