---
name: rust
description: Rust 工程规范。编写或修改 Rust 代码(axum 服务端、CLI 工具、库、workspace)、审查 Rust 代码变更、或初始化 Rust 项目时使用。Tauri 后端在此基础上叠加 tauri-desktop 规范。
---

# Rust 规范

## 编码引导

1. 任何 Rust 编码任务,先读 `rules/rust/baseline.md`(语言基线 + 红线,必载)。
2. 按项目形态读 `rules/rust/practices.md` 对应章节:workspace 结构、服务端(axum)、CLI(clap)、配置、测试、性能、发布。
3. Tauri 后端任务:本 skill 语言基线 + `rules/tauri-desktop/baseline.md`(Command 分层、IPC、Capability 等 Tauri 特有约束)。
4. 跨域联动:
   - API 契约(服务端)→ `rules/collaboration.md`
   - 数据库 schema / sqlx 迁移 → `rules/database.md`(优先级最高)
   - 密钥、敏感数据 → `rules/security.md`
5. 输出代码前自查 baseline.md「红线」清单。

## 代码审查清单

违反 MUST 项标记为阻断:

1. 错误:无生产 unwrap;expect 有「不可能失败」说明;库不暴露 anyhow;错误链有上下文;无业务 panic
2. unsafe:默认 forbid;存在处有评审与 // SAFETY 注释;封装在最小安全抽象内
3. 所有权:参数借用优先;绕借用检查的 clone 有注释;领域语义用 newtype;无 static mut
4. 异步:无异步上下文阻塞;无跨 await 持锁;spawn 任务错误可回收;channel 有界
5. 工具链:fmt/clippy(-D warnings)通过;#[allow] 有原因注释;cargo audit 无高危
6. 服务端:handler → service → repository 分层;统一错误响应;sqlx 编译期校验;事务在 service
7. CLI:数据走 stdout、日志走 stderr;退出码语义化;错误消息可操作
8. 依赖:Cargo.lock 已更新;新 crate 经评审(维护状态/许可证)
9. 日志:tracing 结构化;无 println!/dbg! 残留;无敏感信息
10. 测试:业务逻辑有测试;错误路径有断言;缺陷修复有回归测试

## 项目脚手架

初始化新 Rust 项目时:

1. 确认(未指定必须先询问):项目形态(服务端/CLI/库/workspace)、服务端则确认数据库类型(mysql/postgresql)。
2. `cargo new` / workspace 初始化;`rust-toolchain.toml` 锁 stable;Cargo.toml 配 `[lints.clippy]`(unwrap_used deny)与 MSRV;配 `deny.toml`。
3. 按 `rules/rust/practices.md`「Workspace 与项目结构」组织 crates/bins;CI 配 fmt + clippy + test + audit。
4. 服务端骨架:axum + 统一错误类型(IntoResponse)+ request_id/Trace 中间件 + 配置加载校验 + 优雅停机 + healthz/readyz + sqlx 迁移目录。
5. CLI 骨架:clap derive 子命令结构 + 退出码约定 + tracing(stderr)+ 配置优先级链。
