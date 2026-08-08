# Flutter 基线(必载)

适用:Flutter 跨平台应用(移动为主,可含桌面/Web 目标)。
冲突优先级:`rules/security.md` > `rules/collaboration.md` > 本域规则。

## 技术基线(MUST)

1. Flutter 最新 stable channel(FVM 管理,`.fvmrc` 入库);Dart 3.x + Null Safety;禁止 beta/dev/master channel 用于生产。
2. `pubspec.yaml` 明确 `environment` SDK 约束;依赖版本明确约束(`^x.y.z`),禁止 `any`;`pubspec.lock` 入库。
3. 第三方包引入需评审:pub.dev 评分、维护状态(>12 个月未更新禁止)、许可证(MIT/BSD/Apache 优先);定期 `dart pub outdated`。
4. `analysis_options.yaml` 必须启用严格模式(`flutter_lints` 或 `very_good_analysis` + strict-casts/inference/raw-types);CI 执行 `dart analyze --fatal-infos` 阻断;提交前 `dart format` 统一格式。
5. 多 Package 项目用 Melos 管理。

## 分层架构(MUST)

1. 三层单向:`Presentation(Widget/状态) → Domain(实体、用例、Repository 接口,纯 Dart) → Data(Repository 实现、DataSource)`。
2. Domain 层不依赖 Flutter 框架;外部依赖(网络、数据库、平台服务)经接口隔离在 Data 层。
3. Widget 禁止直接调用 HTTP Client/Database/Repository;数据流单向:`Event → 状态管理 → State → UI`。
4. 状态管理全项目统一一种(推荐 Riverpod;Bloc 亦可),禁止混用;`setState` 仅限 Widget 内部局部 UI 状态。
5. 状态类必须不可变(freezed 或手动 copyWith);状态变更可追踪(DevTools)。
6. DI 统一(Riverpod provider 体系或 get_it),外部依赖全部注册容器,测试可替换 Mock;禁止业务代码直接实例化基础设施组件。

## 代码质量(MUST)

1. 禁止 `print()` 输出日志(统一日志组件);禁止 `dynamic` 绕过类型检查(确需注释原因);禁止滥用 `!` 强制解包。
2. 禁止空 `catch`;禁止 `// ignore:` 绕过 lint(确需附原因并评审)。
3. 公开 API 有文档注释;文件与目录 `lowercase_with_underscores`,类型 `UpperCamelCase`。

## 性能(MUST)

1. 禁止在 `build()` 中执行网络请求/数据库操作/创建 Controller;`itemBuilder` 内禁止创建 Controller/Stream。
2. 长列表必须 `ListView.builder`/`SliverList`,禁止 `ListView(children: [...])`。
3. 主 Isolate 禁止 >16ms 同步计算(用 `compute`/`Isolate.run`);禁止全局持有 `BuildContext`。
4. `const` 构造函数能用尽用;局部刷新(`Consumer`/`BlocBuilder` 范围最小化)。

## 安全(MUST)

1. 禁止硬编码密钥/Token/密码(用 `--dart-define-from-file` 或原生安全存储注入);禁止生产禁用 SSL 验证。
2. Token/密码存 `flutter_secure_storage`(Keychain/Keystore),禁止 `SharedPreferences`。
3. 日志与崩溃报告禁止输出敏感信息;keystore/.p12/私钥禁止入库。

## 红线(MUST NOT)

1. 禁止 Widget 层直接数据访问;禁止跨 Widget 共享状态用 setState;禁止混用状态管理方案。
2. 禁止回调链超 2 层传递复杂状态。
3. 禁止无约束依赖版本与未评审新包。
4. 禁止异步操作后不检查 `mounted`/生命周期就使用 `BuildContext`。
5. 禁止修改历史数据库迁移;本地库 Schema 变更走版本化迁移。
