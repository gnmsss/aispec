# 前端基线(必载)

适用:后台管理(Vue3)、公众号 H5 与微信小程序(uni-app)三类前端项目。
冲突优先级:`rules/security.md` > `rules/collaboration.md` > 本域规则;应用端条款 > 通用条款。

## 技术栈锁定(MUST)

1. 后台管理:`Vue 3.5+ + TypeScript + Vite` + Element Plus + Tailwind CSS + Pinia(+persistedstate) + Vue Router + Axios;富文本 Tiptap,图表 ECharts。
2. 移动端(H5 + 小程序):`uni-app + Vue3 + TypeScript` + uview-plus + UnoCSS(Tailwind 风格语法) + Pinia(persistedstate 适配 `uni.setStorage`);请求统一封装 `uni.request`(禁止 Axios 作为页面请求入口);H5 微信能力 `weixin-js-sdk` 仅 H5 端启用;小程序图表 uCharts。
3. 禁止引入 Taro 生态(`@tarojs/*`);同一项目禁止并存两套同类核心库(双 UI 库、双状态管理、双请求入口)。
4. 同一应用需 H5 + 小程序时用一个 uni-app 项目多端编译;不同应用(员工端/C 端)必须拆分项目,跨项目禁止直接引用对方业务代码。

## TypeScript 与代码(MUST)

1. `strict: true`;禁止无边界 `any`,确需使用必须注释原因;导出函数/组件/composable 显式声明输入输出类型。
2. 命名:变量函数 `camelCase`、类型组件 `PascalCase`、常量 `UPPER_SNAKE_CASE`;目录 `kebab-case`、组件文件 `PascalCase`;禁止语义重复文件名(`util.ts` 与 `utils.ts` 并存)。
3. 单文件职责单一(行数红线见「组件化」节);统一路径别名(`@components/*`、`@services/*` 等),禁止长相对路径。
4. 业务模块间禁止循环依赖;多端差异统一经 `src/platform/*` 适配层,禁止散落页面。

## 组件化(MUST)

1. 组件分两级:**通用组件**(无业务,放 `src/components/`,纯 props/events 驱动,禁止 import store/service/api——这是「无业务」的可验证判定标准)与**业务组件**(含业务,放 `src/pages/<功能>/components/` 就近,可访问本功能的 store/service)。
2. 业务重复不是放弃组件化的理由:每个功能的列表、表单、卡片等仍必须拆为业务组件,只是留在功能目录内,不提前通用化。
3. 业务组件禁止被其他功能目录引用;第二个功能需要且业务语义一致时,经评审提升到共享层;语义不同宁可复制,禁止靠 if 分支把两种业务揉进一个组件。
4. **单文件 ≤ 300 行(全文件口径:SFC 的 template+script+style 合计 / 整个 TSX 文件),超线阻断**——超线视为组件化不彻底,必须继续拆分;CI 纳入行数检查。
5. 拆分必须按语义:拆出的组件有明确命名与 props/events 边界;禁止为凑行数机械拆成 `XxxPart1`/`XxxPart2`;表格列定义、表单 schema 等纯配置抽为独立文件。

## API 与状态(MUST)

1. 接口统一经 `services` 层访问,组件内禁止直接写请求实现;请求统一超时、鉴权、错误码映射与异常兜底。
2. 统一错误结构,禁止页面散写魔法错误码字符串。
3. 全局状态只放跨页面共享数据,页面私有状态禁止提升到全局;副作用(事件、定时器、订阅)必须可清理。
4. DTO 与视图模型分离,组件不直接操作原始接口数据。

## 注释与调试(MUST)

1. 组件、composable、复杂逻辑块需中文注释说明「做什么/为什么」;禁止无意义注释。
2. 生产构建自动移除 `console.*`(保留 `console.error`)与注释,经构建工具配置(`esbuild.drop`/Terser)完成;ESLint `no-console`(allow error)阻断提交。

## 安全(MUST)

1. 禁止前端硬编码密钥、token、私有地址;敏感配置只放服务端。
2. 日志与埋点禁止输出手机号、证件号等明文敏感信息。
3. 富文本渲染与用户输入回显必须过 XSS 过滤(DOMPurify 或平台等效);禁止 `v-html` 直插未消毒内容。
4. 环境变量仅 `VITE_`/框架规定前缀可进入客户端,禁止把服务端密钥放入该前缀。

## 质量门禁(MUST)

1. 提交前通过 `lint + typecheck + test`;PR 含变更说明、影响范围、回滚方案。
2. 依赖检查:`package.json` 不出现禁止依赖(Taro、双同类库);核心依赖符合技术栈锁定。
3. API 变更与联调遵循 `rules/collaboration.md`。

## 红线(MUST NOT)

1. 禁止组件内直接写请求实现、直接操作存储层。
2. 禁止大列表无边界全量渲染(必须分页或虚拟滚动)。
3. 禁止小程序使用 SVG 图标资源(仅位图);禁止小程序主包 > 2MB(超过必须分包)。
4. 禁止页面直接调用平台 API 分支(`uni.getSystemInfo` 按端 if),必须走 `platform/` 适配层。
5. 禁止提交 `console.log`/`debugger` 到主分支。
6. 禁止未评审引入新依赖库;禁止复制粘贴页面级实现替代抽象业务组件。
7. 禁止单文件超过 300 行;禁止无语义机械拆分凑行数。
