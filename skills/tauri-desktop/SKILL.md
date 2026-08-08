---
name: tauri-desktop
description: Tauri 桌面应用工程规范。编写或修改 Tauri v2 应用代码(Rust Command/Service、前端 IPC 调用、自动更新、系统集成)、审查 Tauri 代码变更、或初始化 Tauri 桌面项目时使用。
---

# Tauri 桌面规范

## 编码引导

1. 任何 Tauri 编码任务,先读 `rules/tauri-desktop/baseline.md`(硬约束 + 红线,必载)。
2. 涉及以下场景再读 `rules/tauri-desktop/practices.md` 对应章节:项目结构、数据访问、错误处理、状态与事件、窗口与系统集成、自动更新、配置、日志、性能、测试发布。
3. 跨域联动:
   - 前端部分(组件、状态、样式)→ `rules/frontend/baseline.md`
   - 密钥、敏感数据 → `rules/security.md`
   - 远程 API 契约 → `rules/collaboration.md`
4. 输出代码前自查 baseline.md「红线」清单。

## 代码审查清单

违反 MUST 项标记为阻断:

1. 分层:Command 无业务逻辑;Service 不依赖 Tauri API;前端组件不直接 invoke
2. Rust:无 unwrap()、无 unsafe、无 static mut;Result 无忽略;错误实现 Serialize 带错误码
3. IPC:长耗时操作异步 + Event 进度;无循环逐条 invoke;大数据分页/流式
4. 安全:Capability 最小权限;CSP 无 unsafe-eval/inline;敏感数据在密钥链;HTTPS + 证书校验
5. 数据:前端未直接操作数据库/外部 API;数据库文件在 app_data_dir;迁移脚本入库
6. 文件:PathBuf 构造路径,无用户输入拼接;异步 I/O;临时文件自动清理
7. 更新:走 tauri-plugin-updater + 签名验证;私钥未入库
8. 前端:TypeScript strict;无 console.log 残留;错误提示统一组件
9. 构建:clippy -D warnings 通过;生产构建无 DevTools;发布有代码签名
10. 测试:services 有单元测试;缺陷修复有回归测试

## 项目脚手架

初始化新 Tauri 项目时:

1. 确认(未指定必须先询问):前端框架(React/Vue/Svelte)、是否需要本地数据库、是否需要自动更新。
2. `pnpm create tauri-app`(Tauri v2)+ TypeScript;`rust-toolchain.toml` 锁定 stable;Cargo.toml 配置 Clippy lints(unwrap_used deny)。
3. 按 `rules/tauri-desktop/practices.md`「项目结构」组织 `src-tauri/src/{commands,services,repositories,models,errors,state}` 与前端 `src/api/`。
4. 落地骨架:capabilities 最小权限声明、严格 CSP、统一错误类型(thiserror + Serialize)、tracing 日志(文件轮转)、前端 invoke 封装层。
5. 需要更新时配置 tauri-plugin-updater + 签名密钥(CI Secret);CI 配三平台构建矩阵。
