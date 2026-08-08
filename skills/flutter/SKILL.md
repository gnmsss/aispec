---
name: flutter
description: Flutter 工程规范。编写或修改 Flutter/Dart 应用代码(页面、状态管理、网络、本地存储、平台通道)、审查 Flutter 代码变更、或初始化 Flutter 项目时使用。
---

# Flutter 规范

## 编码引导

1. 任何 Flutter 编码任务,先读 `rules/flutter/baseline.md`(硬约束 + 红线,必载)。
2. 涉及以下场景再读 `rules/flutter/practices.md` 对应章节:项目结构、网络与数据、状态管理、错误处理、UI 与适配、性能、配置与环境、平台集成、测试发布。
3. 跨域联动:
   - API 契约、联调 → `rules/collaboration.md`
   - 视觉、交互、无障碍 → `rules/design.md`
   - 密钥、敏感数据 → `rules/security.md`
4. 输出代码前自查 baseline.md「红线」清单。

## 代码审查清单

违反 MUST 项标记为阻断:

1. 分层:Widget 未直接访问数据层;Domain 无 Flutter 依赖;数据流单向
2. 状态:全项目单一状态管理方案;状态类不可变;setState 仅限局部 UI;无超 2 层回调链
3. build():无网络/数据库操作、无 Controller 创建;itemBuilder 无 Stream/Controller
4. 列表:长列表用 builder 构造;图片有缓存尺寸
5. 异步:主 Isolate 无重计算;异步后使用 context 前检查 mounted
6. 类型:无未注释 dynamic;无滥用 !;无空 catch;无 ignore 绕过 lint
7. 安全:无硬编码密钥;Token 在 secure_storage;日志无敏感信息;证书校验未禁用
8. 主题:无硬编码颜色/字号/间距,走 token;文案走 arb 国际化
9. 依赖:版本有约束;新包经评审;lock 文件已更新
10. 测试:核心逻辑有单元测试;关键页面覆盖三态;缺陷修复有回归测试

## 项目脚手架

初始化新 Flutter 项目时:

1. 确认(未指定必须先询问):目标平台(iOS/Android/桌面)、状态管理(Riverpod/Bloc)、是否需要本地数据库。
2. `flutter create`(FVM 固定 stable 版本);配置 `analysis_options.yaml` 严格模式、`.fvmrc`、环境 dart-define 文件。
3. 按 `rules/flutter/practices.md`「项目结构」建 `lib/{app,core,features,shared}` feature-first 结构。
4. 落地骨架:dio 封装(拦截器 + 错误映射)、go_router 路由表、主题 token、全局错误捕获与崩溃上报、freezed 模型基建、日志组件。
5. 首个 feature 按 `presentation/domain/data` 打样,配单元 + Widget 测试。
