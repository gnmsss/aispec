---
name: database
description: 数据库工程规范。设计数据库 Schema、编写迁移脚本或种子数据、审查数据库变更、或初始化数据库结构时使用。数据库规则拥有全局最高优先级。
---

# 数据库规范

## 编码引导

1. 任何数据库结构/迁移/种子数据任务,先读 `rules/database.md`(全局最高优先级,任何域规则与其冲突以它为准)。
2. 涉及服务端 ORM 使用(连接池、事务、查询)读对应域规则:`rules/go-server/practices.md`、`rules/dotnet-server/practices.md`、`rules/python-server/practices.md`、`rules/node-server/practices.md` 的数据库访问章节。
3. 输出前自查:是否修改了任何已存在的 SQL 脚本(绝对禁止);新变更是否以新增迁移脚本交付。

## Schema 设计要点

1. 表、字段命名 snake_case,含义明确;每表有主键、created_at/updated_at;外键关系与索引显式设计。
2. 金额用 DECIMAL,时间统一 UTC;枚举字段注释取值含义。
3. `WHERE` 高频过滤字段建索引;唯一约束表达业务幂等。
4. 数据库类型仅 mysql/postgresql,未指定时先询问用户。

## 代码审查清单

违反标记为阻断:

1. 未修改任何历史 SQL 脚本(含 schema.sql);变更全部走新增迁移
2. 迁移命名符合 `yyyyMMdd_VV_[data_|seed_]说明.sql`;头部注释完整;DDL/DML 有 DOWN 回滚段
3. 脚本幂等:IF EXISTS 防护 / UPSERT
4. 无 DROP TABLE、无 TRUNCATE
5. 大表变更(>100 万行)标注锁表评估与预估时长
6. 种子数据无真实个人信息;测试种子未混入生产脚本
7. 应用侧未配置启动自动迁移
8. schema.sql(如变更初始化)仍可全新环境一键执行

## 初始化脚手架

新项目数据库初始化时:

1. 确认数据库类型(mysql/postgresql,未指定先询问)与字符集(推荐 utf8mb4 / UTF8)。
2. 建 `docs/migrations/` 目录与 `schema.sql`;schema 包含全部表结构 + 索引 + 系统种子(角色、权限、菜单、字典)。
3. 建 `_migration_history` 表结构纳入 schema。
4. 提供 `docs/seeds/test/` 测试种子样例(虚拟数据)。
