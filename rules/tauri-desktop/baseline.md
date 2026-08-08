# Tauri 桌面基线(必载)

适用:Tauri v2 跨平台桌面应用(Rust 后端 + Web 前端,Windows/macOS/Linux)。
冲突优先级:`rules/security.md` > `rules/collaboration.md` > 本域规则。

## 技术基线(MUST)

1. Tauri v2(禁止新建 v1 项目);Rust stable 最新版,`rust-toolchain.toml` 入库锁定;前端框架不限但必须 TypeScript(`strict: true`)。
2. 包管理:Rust 用 Cargo,前端用 pnpm;提交前 `cargo check` 与前端构建无错误。
3. Clippy 强制:`clippy::all` + `pedantic` warn,`unwrap_used = "deny"`;CI 执行 `cargo clippy -- -D warnings`、`cargo fmt --check`、`cargo audit`(高危漏洞阻断)。
4. 前端遵循 `rules/frontend/baseline.md` 的 TypeScript、命名与调试代码约束。

## 分层架构(MUST)

1. Rust 侧:`commands`(参数校验 + 调用转发,禁止业务逻辑)→ `services`(业务逻辑,禁止依赖 Tauri API,保持可测试)→ `repositories`(持久化,禁止依赖 service);统一错误类型在 `errors/`。
2. 前端侧:组件禁止直接调用 `invoke()`,必须经 `api/` 层封装;IPC 结果存入前端 Store,禁止多组件重复调用 IPC。
3. 持久化数据单一数据源在 Rust 侧;前端 Store 只做视图状态镜像。

## IPC(MUST)

1. Command 参数 `snake_case`(Tauri 自动转 camelCase);返回值必须 `Result<T, E>` 且 `E: Serialize`,错误带结构化错误码。
2. 应用状态用 `app.manage(state)` + `tauri::State` 访问,内部可变字段用 `Mutex`/`RwLock`;禁止 `static mut` / `lazy_static!` 可变全局。
3. 长耗时操作异步执行并用 Event 推送进度,禁止 Command 内长时间阻塞;禁止前端循环逐条 `invoke()`(合并批量)。

## 安全(MUST)

1. Capability 最小权限:`src-tauri/capabilities/` 显式声明,每个窗口独立配置;文件系统访问限定 Scope;禁止 `core:default` 不细化。
2. CSP 严格配置:禁止 `unsafe-eval`、`unsafe-inline`;外部资源域名白名单;禁止 `dangerousRemoteDomainIpcAccess`。
3. 密钥/Token 存系统密钥链(Keychain/Credential Manager),禁止 `localStorage`、`tauri-plugin-store` 或明文文件存敏感数据。
4. 所有 HTTP 走 HTTPS,禁止禁用证书校验;外部 API 由 Rust 侧(`reqwest`)代理,禁止前端直接调用。
5. 文件路径用 `PathBuf` 构造并做路径遍历防护,禁止拼接用户输入;禁止读写应用安装目录,用户数据放 `app_data_dir`。
6. 生产构建禁用 DevTools;发布必须代码签名(Authenticode / Apple Developer ID);更新包必须走签名验证。

## Rust 代码(MUST)

1. 生产代码禁止 `unwrap()`(用 `?` 或 `expect("明确原因")`);禁止 `unsafe`(评审批准并注释除外)。
2. 禁止忽略 `Result`(`let _ =` 需注释);禁止同步文件 I/O 阻塞异步运行时(用 `tokio::fs`)。
3. 禁止硬编码 API 地址、密钥、凭据;配置走 `config.default.toml` + 用户配置目录。

## 红线(MUST NOT)

1. 禁止前端直接操作数据库(必须经 Rust IPC);禁止前端直连外部 HTTP API。
2. 禁止自行实现「下载 zip → 解压覆盖」更新逻辑,必须用 `tauri-plugin-updater`;禁止跳过更新签名验证;禁止签名私钥入库。
3. 禁止要求用户手动下载安装包完成更新。
4. 禁止 `alert()` 展示错误;禁止生产残留 `console.log`。
5. 禁止 `shell:allow-open` 打开任意 URL(限定协议与域名);Shell 执行必须 Sidecar 或显式受限权限。
6. 禁止模块循环依赖。
