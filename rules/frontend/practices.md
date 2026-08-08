# 前端场景实践(按需加载)

硬约束与红线见 `baseline.md`。按应用端读取对应章节:后台管理 / 公众号 H5 / 微信小程序。

## 通用项目结构

```text
<project-root>/
├── src/
│   ├── pages/              # 页面(uni-app)或 views/(admin)
│   ├── components/         # 通用组件(PascalCase)
│   ├── services/           # 接口层:请求封装 + API 模块
│   ├── stores/             # Pinia stores
│   ├── platform/           # 端差异适配层(h5/、mp-weixin/)
│   ├── utils/ 
│   └── styles/             # token、全局样式
├── tests/
├── scripts/
└── package.json / tsconfig.json / eslint 配置
```

1. 别名统一:`@pages/*`、`@components/*`、`@services/*`、`@stores/*`、`@utils/*`。
2. 跨端可共享:eslint-config、tsconfig、工具库、设计 token、通用 SDK 封装;禁止共享:页面、路由、端特有交互与 API 调用。
3. 多 Tab 同页场景优先 `tab-host` 宿主页模式(宿主管理 Tab 状态,子视图组件挂载);仅需独立分享链接/路由权限时才拆独立页面。

## 后台管理(admin-console)

1. 权限点在 `permission/` 统一定义并遵循命名规范,页面禁止散写权限字符串。
2. 列表查询参数统一模型(分页、排序、筛选字段名全局一致);大列表分页或虚拟滚动。
3. 高风险操作二次确认 + 审计信息;接口异常可见化(错误提示 + 重试入口)。
4. 复杂表单/表格抽象为业务组件(如 Schema 驱动的 ProTable 模式);富文本统一 Tiptap 封装组件。
5. Axios 实例统一拦截器:注入鉴权、刷新令牌、错误码映射、超时与重试策略。

## 公众号 H5(wechat-h5)

1. 微信授权流程统一封装在 `platform/h5/wechat`,页面禁止直连 JSSDK;分享、支付、拉起能力必须有失败回退与错误提示。
2. 必须处理三类核心异常:授权拒绝、签名过期、弱网超时。
3. 活动页放 `scenes/`,活动结束可整体归档下线。
4. 首屏资源体积预算,超预算需评审;关键路径骨架屏/降级占位;覆盖 iOS/Android + 主流微信版本兼容验证。

## 微信小程序(miniprogram)

1. 主包/分包目录明确,新增页面必须声明归属;低频页面禁止放主包;主包 ≤ 2MB。
2. 平台 API 统一 `platform/mp-weixin` 封装;高频更新场景控制 setData 粒度。
3. 分享、支付、订阅消息统一封装并有失败回退;UnoCSS 动态类名纳入 safelist/构建产物检查,防止样式丢失。
4. 发版前:包体积检查、资源格式检查(无 SVG)、敏感内容检查、审核驳回项清单过检;灰度发布 + 紧急回滚策略。

## 环境配置

1. `.env` 系列:`.env` 公共 + `.env.development`/`.env.production` 环境差异;敏感值不入库,`.env.local` 进 `.gitignore`。
2. 客户端可见变量仅限 `VITE_` 前缀;`import.meta.env` 类型声明补全(`env.d.ts`)。
3. 构建时配置(API 域名、CDN 前缀)与运行时配置(feature flag)分离;flag 统一管理有默认值。

## 性能

1. Web 端指标基线:LCP ≤ 2.5s、INP ≤ 200ms、CLS ≤ 0.1(移动端 4G 条件);小程序:首屏渲染 ≤ 1.5s、setData 单次 ≤ 64KB。
2. Bundle 管控:路由级代码分割 + 组件按需引入(Element Plus/uview-plus 按需);构建产物体积纳入 CI 检查,超预算阻断;分析用 `rollup-plugin-visualizer`。
3. 资源:图片压缩 + WebP/AVIF 优先(小程序用位图),CDN 分发,懒加载;字体子集化。
4. 渲染:长列表虚拟滚动;`v-for` 必须稳定 key;避免深层响应式大对象(`shallowRef`/`markRaw` 优化)。
5. 网络:接口合并与并行请求;重复请求去重;弱网下有 loading 与超时反馈。
6. 内存:组件卸载清理定时器/监听器/订阅;避免闭包持有大对象。

## 错误监控

1. 全局捕获:`app.config.errorHandler` + `window.onerror` + `unhandledrejection`(uni-app 用对应生命周期);错误分类:脚本错误/接口错误/资源错误/自定义业务错误。
2. 上报接入统一监控平台(Sentry 或等效),带 release 版本与 sourcemap(仅上传平台不发布公网);上报脱敏。
3. 接口错误监控:失败率、慢接口纳入指标;告警:错误率突增、白屏、核心接口失败。

## Git 工作流

1. 分支:`feature/*`、`fix/*`、`hotfix/*`;提交消息 Conventional Commits(commitlint + husky 校验)。
2. 受保护分支禁止直推;合并需 PR + CI 通过;版本号 SemVer,发版打 tag。

## 测试

1. 单元测试:Vitest;工具函数与核心 composable 必测;组件测试用 `@vue/test-utils` + testing-library,断言行为而非实现细节。
2. Mock:接口 mock 统一(msw 或项目约定),禁止在测试里写死生产地址;测试文件与源码同构组织(`__tests__/` 或 `*.spec.ts` 就近)。
3. 关键链路(登录、下单、支付)补 E2E(Playwright),纳入 CI;遵循 `rules/testing.md`。
