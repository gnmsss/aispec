# Rust 场景实践(按需加载)

语言基线与红线见 `baseline.md`。本文件覆盖:workspace 结构、服务端(axum)、CLI、配置、测试、性能、发布;Tauri 后端结构见 `rules/tauri-desktop/practices.md`。

## Workspace 与项目结构

```text
my-project/
├── Cargo.toml                # [workspace] + workspace.dependencies 统一版本
├── rust-toolchain.toml
├── deny.toml                 # cargo-deny 配置
├── crates/
│   ├── domain/               # 领域模型与错误(纯逻辑,无 I/O 依赖)
│   ├── infra/                # 数据访问、外部服务客户端
│   └── common/               # 跨 crate 工具(无业务语义)
└── bins/
    ├── server/               # 服务端可执行(main 仅组装)
    └── cli/                  # CLI 可执行
```

1. 多 crate 用 workspace 管理,版本经 `workspace.dependencies` 统一;crate 依赖单向,禁止循环。
2. `main.rs` 只做配置加载、组件构建、启动编排;业务逻辑在库 crate 中(可测试)。
3. feature flag 设计:默认 feature 最小化;可选依赖用 `optional = true` + feature 门控;feature 组合可编译(CI 抽查 `--no-default-features`)。

## 服务端(axum)

1. 分层同其他服务端域:`handler(路由/提取器/响应映射) → service(用例/事务) → repository(sqlx/sea-orm)`;handler 禁止直接写 SQL。
2. 状态注入:`State<Arc<AppState>>` 集中持有组件(连接池、客户端),构造于启动期;禁止全局单例。
3. 统一响应与错误:实现 `IntoResponse` 的应用错误类型,业务错误 → 错误码,系统错误记日志返回固定文案;响应结构与错误码遵循 `rules/collaboration.md`。
4. 中间件走 tower:`request_id`、超时、限流、`TraceLayer`(tracing 集成);CORS 白名单走配置。
5. sqlx:编译期校验(`query!` 宏或 offline 模式);连接池参数走配置;事务在 service 层;迁移用 `sqlx migrate`(遵循 `rules/database.md`,禁止启动自动迁移生产库)。
6. 优雅停机:监听 SIGTERM,`axum::serve(...).with_graceful_shutdown()` 排空在途请求;健康检查 `/healthz`、`/readyz`。
7. 可观测性:`tracing` + OpenTelemetry(W3C Trace Context 透传),指标 Prometheus 格式;遵循 `rules/operations.md`。

## CLI(clap)

1. 参数解析统一 clap(derive 风格);子命令结构清晰,`--help` 文案完整。
2. 输出纪律:**数据输出走 stdout,日志与诊断走 stderr**;支持 `--json` 时输出稳定 schema 供脚本消费;人读输出可用色彩但尊重 `NO_COLOR`。
3. 退出码语义化:0 成功,非 0 按错误类别区分并文档化;错误消息可操作(说明原因与建议)。
4. 长操作显示进度(indicatif),支持 Ctrl-C 优雅中断并清理临时文件。
5. 配置优先级:命令行参数 > 环境变量 > 配置文件 > 默认值;遵循 XDG 目录规范存放配置。

## 配置

1. 统一配置库(figment/config-rs 选一):强类型结构体 + serde,启动即校验失败退出。
2. 密钥经环境变量或密钥服务注入,禁止入库;`.env` 仅本地开发且 gitignore。

## 测试

1. 单元测试同文件 `#[cfg(test)] mod tests`;集成测试放 `tests/`;文档示例保证 `cargo test --doc` 通过。
2. service 层 mock 依赖(trait + mockall 或手写 fake);repository 集成测试用 testcontainers 起真实数据库。
3. 表驱动风格覆盖正常/边界/异常;错误路径必须有断言(`assert!(matches!(err, MyError::NotFound))`)。
4. 不变量复杂的逻辑(解析器、编解码)可加属性测试(proptest);并发代码用 `loom`(如必要)。

## 性能

1. 热路径避免不必要分配:预分配容量、`&str` 切片替代 String 拼接、`Cow` 按需;大数据流式处理。
2. 性能敏感模块写 criterion 基准,热路径变更 PR 附前后对比;排查用 `cargo flamegraph`/`tokio-console`(异步)。
3. release 构建按需开启 `lto = "thin"`、`codegen-units = 1`(权衡编译时长)。

## 发布

1. 版本 SemVer;库发布前 `cargo publish --dry-run` + CHANGELOG;破坏性变更升主版本。
2. 二进制分发:交叉编译目标明确(`cargo-zigbuild`/cross),CI 产出多平台产物并附校验和;推荐 `cargo-dist` 自动化。
3. 容器部署:多阶段构建(builder + `debian-slim`/distroless),非 root 运行。
