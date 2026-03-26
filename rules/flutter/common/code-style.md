# rules/flutter/common/code-style.md

## 文档目标
1. 定义 Dart 语言与 Flutter 框架的编码风格标准，保持团队一致性。

---

## 命名规范（MUST）

| 类型 | 命名风格 | 示例 |
|------|---------|------|
| 类 / 枚举 / 扩展 / typedef | `UpperCamelCase` | `UserProfile`, `AuthState` |
| 变量 / 函数 / 参数 | `lowerCamelCase` | `userName`, `fetchOrders()` |
| 常量（顶层 / static） | `lowerCamelCase` | `defaultTimeout`, `maxRetries` |
| 文件名 | `lowercase_with_underscores` | `user_profile.dart`, `auth_bloc.dart` |
| 库名 / 包名 | `lowercase_with_underscores` | `package:my_app/core/utils.dart` |
| 私有成员 | 前缀 `_` | `_isLoading`, `_handleTap()` |
| 枚举值 | `lowerCamelCase` | `OrderStatus.pending` |

---

## 文件组织（MUST）

1. 每个文件只包含一个公开类（允许配套的私有辅助类）。
2. 文件名必须与主要导出类名对应：`UserProfile` → `user_profile.dart`。
3. 导入顺序（按空行分组）：
   1. `dart:` 标准库。
   2. `package:flutter/` Flutter 框架。
   3. `package:` 第三方包。
   4. 项目内部包（相对路径或 package 路径）。
4. 禁止使用相对路径导入跨 `lib/` 目录的文件，必须使用 `package:` 路径。

```dart
// 正确的导入顺序
import 'dart:async';
import 'dart:convert';

import 'package:flutter/material.dart';

import 'package:dio/dio.dart';
import 'package:riverpod/riverpod.dart';

import 'package:my_app/core/extensions/string_ext.dart';
import 'package:my_app/features/auth/auth_state.dart';
```

---

## 格式化（MUST）

1. 使用 `dart format` 统一格式化，行宽默认 80 字符。
2. 禁止提交未格式化的代码。
3. 推荐在 IDE 中启用保存时自动格式化。
4. 使用尾随逗号（trailing comma）改善多参数格式化效果。

---

## 文档注释（MUST）

1. 所有公开 API（类、方法、属性）必须添加 `///` 文档注释。
2. 注释语言统一使用中文；与外部开源库交互的接口适配文件允许使用英文。
3. 文档注释第一行为简要描述（一句话），空行后可补充详细说明。
4. 使用 `///` 而非 `/** */` 风格。
5. 私有成员在逻辑复杂时添加注释，简单实现不要求注释。

```dart
/// 用户认证服务。
///
/// 提供登录、登出、Token 刷新等认证相关功能。
/// 内部通过 [AuthRepository] 访问后端 API。
class AuthService {
  /// 使用手机号和验证码登录。
  ///
  /// 登录成功后自动保存 Token 到安全存储。
  /// 抛出 [AuthException] 当认证失败时。
  Future<User> loginWithPhone(String phone, String code) async { ... }
}
```

---

## 类型使用（MUST）

1. 所有公开 API 必须显式声明类型，禁止依赖类型推断。
2. 局部变量允许使用 `var` / `final`，但复杂类型推荐显式声明。
3. 优先使用 `final` 声明不可变变量，减少可变状态。
4. 集合类型必须声明泛型参数：`List<String>` 而非 `List`。
5. 禁止使用 `dynamic`，确需时必须注释说明原因。

---

## 空安全（MUST）

1. 所有代码必须在 Sound Null Safety 下编译。
2. 禁止滥用 `!`（强制解包），必须用 `?.`、`??`、`if-null` 等安全方式处理。
3. late 变量仅用于确定会被初始化的场景（如依赖注入、`initState`），禁止用于延迟 Null 检查。

---

## 禁止事项

1. 禁止在生产代码中使用 `print()` 输出日志（使用结构化日志库）。
2. 禁止使用 `dynamic` 类型绕过类型检查（确需时必须注释原因）。
3. 禁止使用 `as` 进行不安全类型转换，优先使用 `is` 检查。
4. 禁止 `// ignore:` 注释绕过 lint 规则（确需时必须附原因并经评审）。
