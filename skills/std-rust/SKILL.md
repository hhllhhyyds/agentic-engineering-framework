---
name: std-rust
description: 提供 Rust 开发规范。当编写或 review Rust 代码（.rs 文件）时使用。
---

# Rust 开发规范

> 基于 Rust 2024 edition + 项目既有约定。

## 核心规则速查

| 类别 | 必须遵守 |
|------|----------|
| **版本** | Rust 2024 edition，tokio 异步运行时 |
| **错误处理** | Library 用 `thiserror` 定义精确错误，app 用 `anyhow`；禁用 `unwrap()`/`expect()` |
| **文档注释** | 所有 `pub` 项必须有英文 `///` doc comment；设计文档用中文，代码注释用英文 |
| **命名** | 类型/枚举 `CamelCase`，函数/变量 `snake_case`，常量 `SCREAMING_SNAKE_CASE` |
| **模块** | 使用 `foo.rs` 而非 `foo/mod.rs`；最小化 `pub` 暴露面 |
| **可见性** | 默认 `pub(crate)`，仅跨 crate 时放宽为 `pub` |
| **异步** | trait 对象需 `Send + Sync`，统一使用 `#[async_trait]` |
| **测试** | 每个 `.rs` 文件底部有 `#[cfg(test)]` 单元测试；trait 实现需集成测试 |
| **Mock** | 使用 `mockall`，mock 放在 `#[cfg(test)]` 内 |
| **提交前** | `cargo test && cargo clippy -- -D warnings && cargo fmt --check` |
| **依赖** | 公共依赖在 `[workspace.dependencies]` 统一管理 |

## 完整规范

详见 [reference/full-standards.md](reference/full-standards.md)，包含：
- 命名规范、代码组织、错误处理
- 异步编程、文档约定、测试策略
- Commit 规范、Cargo 规范
