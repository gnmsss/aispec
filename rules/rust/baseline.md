# Rust 基线(必载)

适用:所有 Rust 代码(服务端、CLI、工具库;Tauri 后端在遵守本文件基础上叠加 `rules/tauri-desktop/baseline.md`)。
冲突优先级:`rules/database.md` > `rules/security.md` > `rules/collaboration.md` > 本域规则。

## 技术基线(MUST)

1. Rust stable 最新版,`rust-toolchain.toml` 入库锁定;edition 以 `Cargo.toml` 为准(新项目用最新 edition);升级单独提交。
2. 库/长期维护项目声明 MSRV(`rust-version` 字段),提升 MSRV 视为破坏性变更。
3. `Cargo.lock` 入库(应用与 workspace;纯库可选);依赖引入评审:维护状态(>12 个月未更新禁止)、许可证(商用禁 GPL/AGPL)、下载量与替代方案。
4. CI 门禁:`cargo fmt --check`、`cargo clippy --all-targets -- -D warnings`、`cargo test`、`cargo audit`(高危 CVSS ≥ 7.0 阻断);推荐 `cargo-deny` 统管许可证/漏洞/重复依赖。
5. Clippy 配置:`[lints.clippy] all + pedantic = warn`,`unwrap_used = "deny"`,`expect_used = "warn"`(测试代码可豁免)。

## 错误处理(MUST)

1. `Result` + `?` 贯穿,错误必须处理或显式传播;禁止吞错;`let _ =` 忽略 Result 必须注释原因。
2. 分工约定:**库与领域层用 `thiserror`** 定义结构化错误枚举(按域分类,携带上下文);**应用边界(main、handler 顶层)可用 `anyhow`** 聚合;库的公共 API 禁止暴露 `anyhow::Error`。
3. 生产代码禁止 `unwrap()`;`expect()` 仅限「逻辑上不可能失败」场景且消息说明为何不可能;禁止用 `panic!`/`assert!` 做业务错误处理(仅限不可恢复的程序缺陷)。
4. 错误逐层加上下文(`.context()` / `#[source]`),保留根因链;对外(API 响应、CLI stderr)输出可控消息,不泄露内部细节。

## unsafe 治理(MUST)

1. 默认 `#![forbid(unsafe_code)]`(crate 级);确需 unsafe 必须:评审批准 + 每个 unsafe 块上方 `// SAFETY:` 注释说明不变量为何成立。
2. FFI/P-Invoke 等必须 unsafe 的场景,把 unsafe 封装在最小安全抽象层内,对外暴露安全 API。

## 所有权与 API 设计(MUST)

1. 函数参数借用优先:接受 `&str`/`&[T]`/`impl AsRef<T>`,不无脑要 `String`/`Vec<T>` 所有权;`clone()` 若为绕开借用检查必须注释说明。
2. 领域语义用 newtype 表达(`UserId(u64)` 而非裸 `u64`);类型转换实现 `From`/`TryFrom`,禁止散写手工转换函数。
3. 公共类型按需派生 `Debug`/`Clone`/`PartialEq`;公共 API 有文档注释(`///`),`cargo doc` 无警告。
4. 禁止 `static mut` 与可变全局单例;共享状态显式传递或依赖注入;确需全局用 `OnceLock`/`LazyLock` 且不可变。

## 并发与异步(MUST)

1. 异步运行时统一 tokio(项目内唯一);异步上下文禁止阻塞调用(同步 I/O、重计算、`std::thread::sleep`),阻塞操作走 `spawn_blocking` 或专用线程。
2. 共享可变状态最小化:优先消息传递(channel);确需共享用 `Arc<RwLock/Mutex>`,锁粒度小、跨 `.await` 持锁禁止(用 tokio 锁或重构)。
3. spawn 的任务必须有归属:持有 JoinHandle 或交给任务管理器,错误可回收;禁止 fire-and-forget 丢弃错误。
4. channel 选型明确(mpsc/oneshot/broadcast/watch 按语义),有界优先,禁止无界队列积压。

## 日志(MUST)

1. 统一 `tracing`(结构化字段,`#[instrument]` 按需);禁止 `println!`/`dbg!` 做日志或调试残留提交(CLI 的用户输出除外,见 practices)。
2. 禁止输出密钥、令牌等敏感信息;错误日志带上下文与根因链。

## 红线(MUST NOT)

1. 禁止生产 `unwrap()`、未注释的 `expect()`、业务流程 `panic!`。
2. 禁止未评审/未注释 SAFETY 的 `unsafe`;禁止 `static mut`。
3. 禁止异步上下文阻塞、跨 await 持锁、无界 channel。
4. 禁止硬编码密钥/地址/凭据;配置与密钥管理遵循 `rules/security.md`。
5. 禁止绕过 clippy(`#[allow]` 必须附原因注释)。
