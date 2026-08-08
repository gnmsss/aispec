# 数据库规范(全局最高优先级)

适用:所有涉及数据库结构、迁移脚本、种子数据的变更。与任何域规则冲突时,以本文件为准。

## 核心规则(MUST)

1. `schema.sql` 是全量初始化脚本:包含所有表结构、索引、菜单权限等初始化数据,必须保证可在生产环境直接执行完成初始化。
2. 所有后续结构或数据变更必须新增迁移脚本,目录 `docs/migrations/`,命名 `yyyyMMdd_版本号_变更说明.sql`(如 `20260808_01_add_channel_menu.sql`)。
3. **严禁修改任何已存在的 SQL 脚本**(含 `schema.sql` 与历史迁移);严禁调整已执行脚本的排序;变更一律通过新增脚本交付。
4. 已执行迁移记录在 `_migration_history` 表;迁移按文件名排序执行。

## 迁移脚本(MUST)

1. 分类与前缀:DDL `yyyyMMdd_VV_`、DML `yyyyMMdd_VV_data_`、种子 `yyyyMMdd_VV_seed_`。
2. 脚本头部注释:Migration 名、Author、Description、关联需求;DDL/DML 必须提供 DOWN 回滚段。
3. 脚本必须幂等:DDL 用 `IF NOT EXISTS`/`IF EXISTS` 防护;种子数据用 UPSERT。
4. 禁止 `DROP TABLE`(用重命名保留)与 `TRUNCATE`(用条件 DELETE)。
5. 大表(>100 万行)变更必须评估锁表影响并标注预估执行时间,需审批;MySQL 索引创建用 `ALGORITHM=INPLACE, LOCK=NONE`(如支持)。

## 种子数据(MUST)

1. 系统/业务种子(角色、权限、菜单、字典、默认配置)入 `schema.sql`,后续经 `seed_` 迁移补充;测试种子放 `docs/seeds/test/`,禁止在生产执行。
2. 种子数据幂等;禁止包含真实个人信息,测试数据用虚拟手机号/邮箱/证件号。

## 变更发布流程(MUST)

编写脚本(含 UP/DOWN)→ 本地验证 → Code Review → test 自动执行 → staging 验证 → 大表审批 → prod 发布窗口执行 → 数据完整性 + 应用兼容性验证 → 记录 `_migration_history`。

## 备份与脱敏(MUST)

1. prod:全量每日 + 增量每小时,全量保留 30 天,异地存储且加密;staging 每周全量保留 7 天。
2. 生产数据导入非生产环境、日志、导出报表必须脱敏:手机号 `138****1234`、身份证中间 10 位打星、银行卡仅留后 4 位、邮箱用户名打星、姓名留姓、地址留省市。
3. 每季度在 staging 做一次恢复演练,记录 RTO/RPO;禁止在 prod 演练。

## 与应用层的衔接(MUST)

1. 应用禁止启动时自动执行迁移(AutoMigrate/`alembic upgrade`/`migrate deploy`),迁移独立执行。
2. ORM 迁移工具(EF Core Migrations/Alembic/Prisma Migrate/Room)生成的迁移同样遵守「禁止修改历史」与回滚要求。
3. 金额禁止 float/double,统一 Decimal/定点;时间统一 UTC 存储。
