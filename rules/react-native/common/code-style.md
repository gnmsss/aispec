# rules/react-native/common/code-style.md

## 文档目标
1. 定义 React Native + TypeScript 项目的编码风格标准，保持团队一致性。

---

## 命名规范（MUST）

| 类型 | 命名风格 | 示例 |
|------|---------|------|
| 组件（函数式） | `PascalCase` | `UserProfile`, `OrderList` |
| Hook | `camelCase`，`use` 前缀 | `useAuth`, `useOrderList` |
| 函数 / 变量 / 参数 | `camelCase` | `userName`, `fetchOrders()` |
| 常量（模块级） | `UPPER_SNAKE_CASE` | `MAX_RETRY_COUNT`, `API_BASE_URL` |
| 枚举 | `PascalCase`（枚举名+值） | `OrderStatus.Pending` |
| 类型 / 接口 | `PascalCase` | `UserProfile`, `AuthState` |
| 类型文件 | `camelCase` + `.types.ts` | `auth.types.ts`, `order.types.ts` |
| 组件文件 | `PascalCase` + `.tsx` | `UserProfile.tsx`, `OrderCard.tsx` |
| 工具函数文件 | `camelCase` + `.ts` | `dateUtils.ts`, `formatCurrency.ts` |
| 测试文件 | 与源文件同名 + `.test.ts(x)` | `UserProfile.test.tsx` |
| 样式文件 | 与组件同名 + `.styles.ts` | `UserProfile.styles.ts` |

---

## 组件文件组织（MUST）

1. 每个组件文件只导出一个公开组件（允许文件内私有辅助组件）。
2. 文件名必须与默认导出组件名一致：`UserProfile` → `UserProfile.tsx`。
3. 组件目录结构推荐：

```
components/
  UserProfile/
    index.ts          # re-export
    UserProfile.tsx   # 组件实现
    UserProfile.styles.ts  # 样式
    UserProfile.test.tsx   # 测试
    UserProfile.types.ts   # 类型定义（可选）
```

4. 禁止在组件文件中混写业务逻辑，复杂逻辑必须抽取到自定义 Hook 或 Service 层。
5. 组件内部结构顺序（SHOULD）：类型定义 → Props 解构 → Hook 调用 → 事件处理函数 → JSX 返回。

---

## TypeScript 类型注解（MUST）

1. 所有组件 Props 必须使用 `interface` 或 `type` 显式定义，禁止使用 `any`。
2. 禁止使用 `as any` 或 `as unknown as T` 绕过类型检查（确需时必须注释原因）。
3. API 响应数据必须定义对应的 TypeScript 类型，禁止使用 `Record<string, any>`。
4. 函数返回值类型必须显式声明（允许 React 组件返回值推断）。
5. 优先使用 `interface` 定义对象结构，使用 `type` 定义联合类型 / 交叉类型 / 工具类型。
6. 泛型参数必须使用语义化命名：`TData`、`TError` 而非 `T`、`U`。

```typescript
// 正确
interface UserProfileProps {
  userId: string;
  onEdit: (user: User) => void;
}

// 错误
const UserProfile = (props: any) => { ... };
```

---

## Import 排序（MUST）

1. 导入顺序必须按以下分组，组间用空行分隔：
   1. React / React Native 核心库。
   2. 第三方库。
   3. 项目内部模块（使用路径别名 `@/`）。
   4. 相对路径导入（当前模块内部）。
   5. 类型导入（`import type`）。

```typescript
import React, { useCallback, useMemo } from 'react';
import { View, Text, StyleSheet } from 'react-native';

import { useNavigation } from '@react-navigation/native';
import { useQuery } from '@tanstack/react-query';

import { useAuth } from '@/hooks/useAuth';
import { userService } from '@/services/userService';

import { UserAvatar } from './UserAvatar';
import { styles } from './UserProfile.styles';

import type { UserProfileProps } from './UserProfile.types';
```

2. 推荐使用 `eslint-plugin-import` + `eslint-plugin-simple-import-sort` 自动排序。
3. 禁止使用 `import *` 通配符导入（确需时限于第三方库类型导出）。

---

## 格式化（MUST）

1. 使用 **Prettier** 统一格式化，行宽 100 字符。
2. 禁止提交未格式化的代码。
3. 推荐在 IDE 中启用保存时自动格式化。
4. JSX 属性超过 3 个时必须换行书写。

---

## 文档注释（MUST）

1. 所有公开组件、Hook、Service 函数必须添加 JSDoc 注释。
2. 注释语言统一使用中文；与外部开源库交互的接口适配文件允许使用英文。
3. 复杂业务逻辑必须在函数头部添加说明注释。
4. Props 中非直观字段必须添加 JSDoc 行内注释。
5. 私有工具函数在逻辑复杂时添加注释，简单实现不要求注释。

```typescript
/**
 * 用户认证 Hook。
 *
 * 提供登录、登出、Token 刷新等认证相关功能。
 * 内部通过 AuthService 访问后端 API。
 */
export function useAuth(): AuthContext {
  // ...
}
```

---

## 禁止事项

1. 禁止在生产代码中使用 `console.log` / `console.warn` / `console.error`（使用结构化日志库）。
2. 禁止使用 `any` 类型绕过类型检查（确需时必须注释原因）。
3. 禁止使用 `// @ts-ignore` 注释跳过类型检查（使用 `@ts-expect-error` 并附原因）。
4. 禁止使用 `eslint-disable` 注释绕过 lint 规则（确需时必须附原因并经评审）。
5. 禁止使用 `var` 声明变量，必须使用 `const` / `let`。
