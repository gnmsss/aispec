# Tauri 桌面场景实践(按需加载)

硬约束与红线见 `baseline.md`。

## 项目结构

```text
MyTauriApp/
├── src-tauri/                    # Rust 后端
│   ├── Cargo.toml / tauri.conf.json
│   ├── capabilities/main.json    # 权限声明(最小化)
│   ├── migrations/               # SQLite 迁移脚本(001_init.sql …)
│   ├── config.default.toml       # 默认运行时配置
│   └── src/
│       ├── main.rs / lib.rs      # 入口、Builder 配置
│       ├── commands/             # Tauri Command(仅校验 + 转发)
│       ├── services/             # 业务逻辑(不依赖 Tauri API)
│       ├── repositories/         # 数据访问
│       ├── models/  errors/  state/
└── src/                          # Web 前端
    ├── api/                      # invoke 封装层(唯一 IPC 入口)
    ├── pages/  components/  stores/  utils/
```

## 数据访问

1. 本地数据库用 SQLite + `sqlx::SqlitePool`(或 `rusqlite` 统一选型);数据库文件放 `app_data_dir`;Schema 变更走 `migrations/` 迁移脚本入库。
2. 远程 API 由 Rust 侧 `reqwest` 统一发起:超时(默认 30s)+ 重试策略;Base URL 走配置;Token 从密钥链读取。
3. 文件操作用 `tokio::fs` 异步;大文件流式处理;临时文件用 `tempfile` 保证异常清理。
4. 离线支持:无网络可启动并用本地数据;请求失败降级本地缓存并提示离线;恢复后按需同步。

## 错误处理

1. Rust 侧统一错误枚举(`thiserror`),按域分类并实现 `Serialize`,IPC 边界转为 `{ code, message }` 结构。
2. 前端 api 层统一拦截错误:业务错误映射用户可读提示(toast/dialog 统一组件),系统错误上报日志;禁止裸 `alert`。
3. Panic 处理:注册 panic hook 记录崩溃日志到日志目录。

## 状态与事件

1. Rust 应用状态 `app.manage` 注入;高频读低频写用 `RwLock`,简单互斥用 `Mutex`;跨线程异步任务用 `tokio::spawn` + channel。
2. 后端 → 前端推送用 Event(`app.emit`);前端监听统一在 api 层注册与清理,组件卸载取消订阅。
3. 前端状态:React 用 Zustand、Vue 用 Pinia;来自 Rust 的数据经 Store 缓存共享。

## 窗口与系统集成

1. 多窗口:每窗口独立 Capability;窗口创建/关闭逻辑集中管理,禁止散落页面。
2. 系统托盘、全局快捷键、开机自启用官方插件(`tauri-plugin-*`),注册集中在 `lib.rs` Builder。
3. 深链/单实例:用 `tauri-plugin-single-instance` + `tauri-plugin-deep-link`。

## 自动更新

1. 必须用 `tauri-plugin-updater`:启动时静默检测 + 用户手动检测入口;下载安装有进度反馈(Event 推送);安装后提示重启。
2. 更新清单(latest.json)部署在 HTTPS 端点;签名密钥安全保管(CI Secret),公钥打进应用。
3. 灰度策略:按渠道(stable/beta)分发更新清单;版本号 SemVer;更新失败可回退,不破坏用户数据。

## 配置

1. 默认配置 `config.default.toml` 打包;用户配置写入 `app_config_dir`,启动时合并(用户 > 默认);配置变更热生效或明确提示重启。
2. 敏感配置(Token)只进密钥链;日志级别、代理等运行时可调。

## 日志与可观测性

1. Rust 侧用 `tracing` + `tracing-appender`(文件轮转,保留期限可配);前端错误经 IPC 汇入统一日志。
2. 日志写入 `app_log_dir`;禁止记录敏感数据;提供「打开日志目录」入口便于用户反馈。
3. 崩溃与错误可选接入远端上报(用户同意后),带应用版本与系统信息。

## 性能

1. 启动优化:首屏窗口尽快可见(先显示再异步加载数据);重初始化放异步任务;避免启动时同步扫描大目录。
2. IPC 传输大数据用分页/流式,禁止单次传输超大 JSON;二进制数据用 `tauri::ipc::Response` 或临时文件句柄。
3. 前端打包体积管控(按需引入);长列表虚拟滚动。

## 测试与发布

1. services 层单元测试(不依赖 Tauri API,`cargo test`);repository 用内存 SQLite 测试;前端 api 层 mock invoke(`@tauri-apps/api/mocks`)。
2. CI:`cargo clippy -D warnings` + `cargo test` + 前端 lint/typecheck/test + `tauri build` 三平台矩阵。
3. 发布:三平台安装包签名;版本号与 `tauri.conf.json` 一致;发布说明含变更与回滚指引。
