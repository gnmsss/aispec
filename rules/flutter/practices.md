# Flutter 场景实践(按需加载)

硬约束与红线见 `baseline.md`。默认状态管理 Riverpod(Bloc 项目按等价模式执行)。

## 项目结构(feature-first)

```text
lib/
├── main.dart                    # 入口:runApp + 初始化编排
├── app/                         # App Widget、路由(go_router)、主题、全局 Provider
├── core/                        # 跨功能技术组件:network(dio 封装)、storage、logger、error、extensions
├── features/<feature>/
│   ├── presentation/            # pages、widgets、providers(或 bloc/)
│   ├── domain/                  # entities、repositories(接口)、usecases(可选)
│   └── data/                    # models(json_serializable/freezed)、datasources、repositories(实现)
└── shared/                      # 跨 feature 共享 widget、常量、工具
test/                            # 与 lib/ 同构组织
```

1. feature 间禁止直接引用对方内部文件,共享内容下沉 `shared/`/`core/`。
2. 路由统一 `go_router`:路由表集中定义,深链与守卫(登录态)统一处理。

## 网络与数据

1. HTTP 统一 `dio` 封装于 `core/network`:拦截器(鉴权注入、刷新令牌、日志脱敏、错误映射)、超时、重试;业务经 Repository 调用,禁止页面直接用 dio。
2. JSON 模型用 `freezed` + `json_serializable` 生成,禁止手写 `fromJson` 易错样板;API 错误映射为统一 `Failure` 类型(sealed class)。
3. 本地存储:结构化数据用 `drift`(SQLite)或 `isar`/`hive`,统一选型;键值对用 `shared_preferences`;敏感数据用 `flutter_secure_storage`。
4. 缓存策略:Repository 层实现(内存 + 本地),明确失效规则;离线优先场景本地兜底 + 恢复同步。

## 状态管理(Riverpod)

1. 用代码生成风格(`@riverpod` 注解 + riverpod_generator);异步数据用 `AsyncNotifier`/`FutureProvider`,UI 层用 `AsyncValue.when` 处理 loading/error/data 三态。
2. Provider 粒度按功能拆分,禁止巨型全局 state;`ref.watch` 精确到字段(`select`)减少重建。
3. 副作用(导航、toast)经 `ref.listen` 触发,不在 build 中执行。

## 错误处理

1. 全局兜底:`runZonedGuarded` + `FlutterError.onError` + `PlatformDispatcher.onError` 统一捕获上报。
2. 错误分级:业务错误(用户可见提示)/系统错误(上报 + 通用提示);统一错误提示组件,禁止散写 SnackBar 文案逻辑。
3. 崩溃上报接入 Sentry/Firebase Crashlytics,带版本与设备信息,脱敏。

## UI 与适配

1. 主题集中:`ThemeData` + 扩展 token(颜色、字号、间距),浅/深两套;禁止页面硬编码颜色字号。
2. 适配:布局用相对约束(Expanded/Flexible/LayoutBuilder),断点适配平板/折叠屏;文本尊重系统字体缩放并验证极端值。
3. 国际化用 `flutter_localizations` + `intl`(arb 文件),文案禁止硬编码在 Widget。
4. 视觉与交互遵循 `rules/design.md`(无障碍:Semantics、对比度、触控目标 ≥ 48dp)。

## 性能

1. 指标目标:帧率稳定 60fps(高刷设备跟随),启动至首屏 ≤ 2s,包体积按平台预算管控。
2. 图片:网络图用 `cached_network_image`;指定 `cacheWidth/cacheHeight`;大图列表降采样。
3. 构建优化:`const` Widget、`RepaintBoundary` 隔离高频重绘、避免深层 `Opacity`/`ClipPath`。
4. 性能排查用 DevTools(Performance/Memory);性能敏感变更附 profile 模式验证。

## 配置与环境

1. 多环境用 `--dart-define-from-file=env/{dev,prod}.json`;禁止环境常量硬编码;flavor 按平台配置(Android productFlavors / iOS xcconfig)。
2. 图标与启动图用 `flutter_launcher_icons`/`flutter_native_splash` 生成。

## 平台集成

1. 平台通道:MethodChannel 封装在 `core/platform`,接口抽象,禁止 Widget 直接调用;错误码跨端统一。
2. 插件优先官方/社区高分包;权限申请统一封装(`permission_handler`),先解释后申请。

## 测试与发布

1. 单元测试:Domain 用例与 Repository(mock DataSource);Provider 测试用 `ProviderContainer`;Widget 测试覆盖关键页面三态(loading/error/data)。
2. 集成测试(`integration_test`)覆盖核心链路;golden test 用于关键组件视觉回归(可选)。
3. CI:`dart analyze --fatal-infos` + `dart format --set-exit-if-changed` + `flutter test` + 双平台构建。
4. 发布:Android AAB + 签名(keystore 走 CI Secret);iOS 走 TestFlight;版本号 `x.y.z+build`;灰度发布(分阶段放量)+ 崩溃率监控回滚阈值。
