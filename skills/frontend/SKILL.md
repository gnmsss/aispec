---
name: frontend
description: 前端工程规范。编写或修改前端代码(Vue3 后台管理、uni-app 公众号 H5 / 微信小程序)、审查前端代码变更、或初始化前端项目时使用。
---

# 前端规范

## 编码引导

1. 任何前端编码任务,先读 `rules/frontend/baseline.md`(技术栈锁定 + 硬约束 + 红线,必载)。
2. 按应用端读 `rules/frontend/practices.md` 对应章节:后台管理 / 公众号 H5 / 微信小程序;涉及环境配置、性能、监控、测试再读对应章节。
3. 跨域联动:
   - API 契约、联调、错误码映射 → `rules/collaboration.md`
   - 视觉、交互、无障碍、响应式 → `rules/design.md`
   - XSS、敏感数据、依赖安全 → `rules/security.md`
4. 输出代码前自查 baseline.md「红线」清单;新增依赖前核对技术栈锁定。

## 代码审查清单

违反 MUST 项标记为阻断:

1. 技术栈:无禁止依赖(Taro、双同类库);uni-app 端未用 Axios 作页面请求
2. 类型:strict 全开,无无注释 any,导出 API 有显式类型
3. 分层:请求走 services 层;组件未直写请求;平台分支走 platform/ 适配层
4. 状态:全局 store 无页面私有状态;副作用有清理
5. 安全:无硬编码密钥;无未消毒 v-html;日志埋点无敏感明文
6. 性能:大列表有分页/虚拟滚动;组件按需引入;v-for 有稳定 key
7. 小程序:无 SVG 图标;新页面声明主/分包归属;主包 ≤ 2MB
8. 调试残留:无 console.log/debugger;生产构建配置了 console 清理
9. 错误处理:接口异常有兜底与可见反馈;错误码走统一映射
10. 提交:Conventional Commits;lint + typecheck + test 通过
11. 契约:接口变更与后端契约同步(rules/collaboration.md)

## 项目脚手架

初始化新前端项目时:

1. 确认应用端(未指定必须先询问):后台管理(Vue3 + Vite)/ 移动端(uni-app,确认目标端 H5、小程序或双端)。
2. 按 `rules/frontend/baseline.md` 技术栈锁定安装核心依赖,禁止超出清单自行加库。
3. 按 `rules/frontend/practices.md`「通用项目结构」建目录与路径别名;配置 ESLint + Prettier + husky + lint-staged + commitlint。
4. 落地骨架:请求层封装(拦截器 + 错误码映射 + 超时)、Pinia store 结构、platform 适配层、`.env` 系列与类型声明、全局错误捕获与上报。
5. 后台管理额外:权限点体系、路由守卫、布局框架;小程序额外:分包配置、UnoCSS safelist、包体积检查脚本。
