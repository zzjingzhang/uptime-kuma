# Uptime Kuma 数据库架构深度分析

## 目录

1. [Setup-Database 流程分析](#1-setup-database-流程分析)
2. [Database.connect() 与 db-config 读写流程](#2-databaseconnect-与-db-config-读写流程)
3. [SQLite Template Copy 与 PRAGMA 机制](#3-sqlite-template-copy-与-pragma-机制)
4. [MariaDB createTables() 流程](#4-mariadb-createtables-流程)
5. [Knex MySQL2 Column Compiler Patch](#5-knex-mysql2-column-compiler-patch)
6. [Embedded MariaDB 生命周期管理](#6-embedded-mariadb-生命周期管理)
7. [为什么新字段必须放进 knex_migrations 而不是 knex_init_db.js](#7-为什么新字段必须放进-knex_migrations-而不是-knex_init_dbjs)
8. [原生 SQL 迁移对 SQLite/MariaDB 双支持的风险分析](#8-原生-sql-迁移对-sqlitemariadb-双支持的风险分析)

---

## 1. Setup-Database 流程分析

### 核心文件
- [setup-database.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/setup-database.js)

### 流程概述

`SetupDatabase` 类是一个独立的 Express 应用，用于在首次安装或配置无效时引导用户完成数据库配置。

#### 初始化优先级逻辑 (构造函数)

```
优先级顺序：环境变量 > db-config.json > kuma.db (1.x 升级) > 显示设置页面
```

**关键判断逻辑** [L66-L112](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/setup-database.js#L66-L112)：

1. **尝试读取 db-config.json**
   - 成功：`needSetup = false`
   - 失败：进入下一步

2. **检查 kuma.db 是否存在 (1.x 用户升级)**
   - 存在：自动生成 SQLite 配置的 `db-config.json`，`needSetup = false`
   - 不存在：`needSetup = true`

3. **检查环境变量 UPTIME_KUMA_DB_TYPE**
   - 存在：覆盖配置并写入 `db-config.json`，`needSetup = false`
   - 支持通过 `_FILE` 后缀从 Docker secrets 文件读取敏感信息

#### 设置页面 API 端点

| 端点 | 方法 | 功能 |
|------|------|------|
| `/setup-database-info` | GET | 返回设置状态、embedded MariaDB 启用状态 |
| `/setup-database` | POST | 接收并验证数据库配置，测试连接，写入配置 |

#### 配置验证流程 [L170-L293](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/setup-database.js#L170-L293)

1. **支持的数据库类型**：`sqlite`、`mariadb`、`embedded-mariadb` (需环境变量启用)
2. **MariaDB 外部连接验证**：
   - Socket 模式：优先使用 `UPTIME_KUMA_DB_SOCKET`
   - TCP 模式：需要 hostname、port、dbName、username、password
   - 执行 `SELECT 1` 测试连接
3. **写入配置**：调用 `Database.writeDBConfig()`
4. **关闭设置服务器，启动主服务器**

---

## 2. Database.connect() 与 db-config 读写流程

### 核心文件
- [database.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js)

### db-config 读写

#### readDBConfig() [L205-L219](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L205-L219)

```javascript
static readDBConfig() {
    let dbConfigString = fs.readFileSync(path.join(Database.dataDir, "db-config.json")).toString("utf-8");
    dbConfig = JSON.parse(dbConfigString);
    // 验证 type 字段
    return dbConfig;
}
```

**验证规则**：
- 必须是 object 类型
- `type` 字段必须是 string

#### writeDBConfig() [L226-L228](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L226-L228)

```javascript
static writeDBConfig(dbConfig) {
    fs.writeFileSync(path.join(Database.dataDir, "db-config.json"), JSON.stringify(dbConfig, null, 4));
}
```

### Database.connect() 主流程 [L237-L446](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L237-L446)

#### 步骤 1: Knex MySQL2 Column Compiler Patch (见第 5 节)
#### 步骤 2: 读取配置，降级处理
- 读取失败时默认使用 SQLite

#### 步骤 3: 连接池配置
- **MariaDB**：默认 10 连接，最大 100
- **SQLite**：默认单连接（避免 SQLITE_BUSY），可通过 `UPTIME_KUMA_SQLITE_SINGLE_CONNECTION=false` 启用多连接

#### 步骤 4: 按数据库类型建立连接

| 数据库类型 | 处理方式 |
|-----------|----------|
| SQLite | 复制模板数据库 → 配置 Knex → 初始化 PRAGMA |
| MariaDB (外部) | 先创建数据库 → 配置 Knex 连接 |
| MariaDB (嵌入式) | 启动 EmbeddedMariaDB → 通过 socket 连接 |

#### 步骤 5: 初始化 RedBeanNode ORM
```javascript
R.setup(knexInstance);
R.freeze(true);
R.autoloadModels("./server/model");
```

#### 步骤 6: 数据库特定初始化
- SQLite：输出 journal_mode、cache_size、版本信息
- MariaDB：调用 `initMariaDB()` 检查表是否存在，不存在则 `createTables()`

---

## 3. SQLite Template Copy 与 PRAGMA 机制

### Template Copy 机制 [L292-L296](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L292-L296)

```javascript
if (dbConfig.type === "sqlite") {
    if (!fs.existsSync(Database.sqlitePath)) {
        log.info("server", "Copying Database");
        fs.copyFileSync(Database.templatePath, Database.sqlitePath);
    }
    // ...
}
```

**设计意图**：
- 模板路径：`./db/kuma.db`（预初始化的空数据库）
- 目标路径：`./data/kuma.db`（运行时数据库）
- 避免从零创建表结构，确保与 MariaDB 的 `createTables()` 产生一致的 schema

### PRAGMA 初始化 [L454-L479](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L454-L479)

`initSQLite()` 在每个连接创建后执行（`pool.afterCreate` 钩子）：

| PRAGMA | 值 | 说明 |
|--------|-----|------|
| `journal_mode` | WAL / MEMORY | 测试模式用 MEMORY，生产用 WAL（Write-Ahead Logging） |
| `foreign_keys` | ON | 启用外键约束 |
| `cache_size` | -12000 | 页缓存大小（负数表示 KB，约 12MB） |
| `auto_vacuum` | INCREMENTAL | 增量自动清理 |
| `busy_timeout` | 5000 | 锁等待超时 5 秒，避免 SQLITE_BUSY |
| `synchronous` | NORMAL | 同步模式，平衡性能与安全性 |

**注意**：SQLite 外键在 Knex 迁移时会临时关闭，解决 Knex 的已知问题。

---

## 4. MariaDB createTables() 流程

### 核心文件
- [knex_init_db.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/db/knex_init_db.js)

### 触发时机 [database.js L485-L495](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L485-L495)

```javascript
static async initMariaDB() {
    let hasTable = await R.hasTable("docker_host");
    if (!hasTable) {
        const { createTables } = require("../db/knex_init_db");
        await createTables();
    }
}
```

### createTables() 结构

#### 警告提示
```javascript
/**
 * ⚠️⚠️⚠️⚠️⚠️⚠️ DO NOT ADD ANYTHING HERE!
 * IF YOU NEED TO ADD FIELDS, ADD IT TO ./db/knex_migrations
 * See ./db/knex_migrations/README.md for more information
 */
```

#### 创建的表（按顺序）
1. `docker_host` - Docker 主机配置
2. `group` - 监控分组
3. `proxy` - 代理配置
4. `user` - 用户表
5. `monitor` - 监控项（最大的表，60+ 字段）
6. `heartbeat` - 心跳记录
7. `incident` - 事件记录
8. `maintenance` - 维护计划
9. `status_page` - 状态页
10. 关联表：`maintenance_status_page`, `maintenance_timeslot`, `monitor_group`, `monitor_maintenance`, `monitor_notification`, `monitor_tag`
11. `notification` - 通知配置
12. `tag` - 标签
13. `monitor_tls_info` - TLS 信息
14. `notification_sent_history` - 通知发送历史
15. `setting` - 系统设置
16. `status_page_cname` - 状态页 CNAME

#### 历史迁移补丁的回溯
`createTables()` 不仅包含基础表结构，还包含了历史补丁的回溯（约 L406-L604），确保新安装的 MariaDB 与经历过完整迁移链的 SQLite 数据库 schema 一致。

---

## 5. Knex MySQL2 Column Compiler Patch

### 核心文件
- [mysql2-columncompiler.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/utils/knex/lib/dialects/mysql2/schema/mysql2-columncompiler.js)
- [database.js L238-L244](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L238-L244)

### Patch 注入方式

```javascript
const { getDialectByNameOrAlias } = require("knex/lib/dialects");
const mysql2 = getDialectByNameOrAlias("mysql2");
mysql2.prototype.columnCompiler = function () {
    return new KumaColumnCompiler(this, ...arguments);
};
```

### Patch 内容：defaultTo 方法覆写

```javascript
class KumaColumnCompiler extends ColumnCompilerMySQL {
    defaultTo(value) {
        if (this.type === "text" && typeof value === "string") {
            return `default (${formatDefault(value, this.type, this.client)})`;
        }
        return super.defaultTo.apply(this, arguments);
    }
}
```

### 问题背景

**原生 Knex 问题**：
- MySQL/MariaDB 的 TEXT 类型默认值需要特殊语法
- Knex 默认生成：`default 'value'`
- 但 MySQL 8.0+/MariaDB 10.2+ 要求：`default ('value')`（表达式形式）

**Patch 效果**：
```sql
-- 修复前（Knex 默认，会报错）
ALTER TABLE monitor ADD description TEXT default 'test';

-- 修复后（Patch 后）
ALTER TABLE monitor ADD description TEXT default ('test');
```

---

## 6. Embedded MariaDB 生命周期管理

### 核心文件
- [embedded-mariadb.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/embedded-mariadb.js)

### 单例模式

```javascript
static getInstance() {
    if (!EmbeddedMariaDB.instance) {
        EmbeddedMariaDB.instance = new EmbeddedMariaDB();
    }
    return EmbeddedMariaDB.instance;
}
```

### 启动流程 [L57-L80](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/embedded-mariadb.js#L57-L80)

```
start()
  ↓
initDB() - 初始化数据目录
  ↓
    ├─ 不存在时执行 mariadb-install-db
    ├─ 设置目录权限 (uid:gid 1000:1000, mode 755)
    └─ 创建 run 目录
  ↓
startChildProcess() - 启动 mariadbd 子进程
  ↓
轮询等待 "ready for connections" 日志
```

### 子进程管理 [L86-L132](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/embedded-mariadb.js#L86-L132)

**启动参数**：
```javascript
childProcess.spawn("mariadbd", [
    "--user=node",
    "--datadir=/app/data/mariadb",
    "--socket=/app/data/run/mariadb.sock",
    "--pid-file=/app/data/run/mysqld.pid",
]);
```

**事件处理**：
- `close`：非 0 退出码自动重启
- `error`：记录错误（如 ENOENT 表示 mariadbd 未找到）
- stdout/stderr：监听 "ready for connections" 触发数据库初始化

### 数据库后初始化 [L203-L214](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/embedded-mariadb.js#L203-L214)

```javascript
async initDBAfterStarted() {
    const connection = mysql.createConnection({
        socketPath: this.socketPath,
        user: this.username,
    });
    await connection.execute("CREATE DATABASE IF NOT EXISTS `kuma`");
    this.started = true;
}
```

### 约束条件
- 仅限 Docker 容器内使用
- 运行用户必须是 `node` 或 `root`
- 通过 Unix socket 连接，不暴露 TCP 端口

---

## 7. 为什么新字段必须放进 knex_migrations 而不是 knex_init_db.js

### 设计哲学

**knex_init_db.js 是"基准线"，knex_migrations 是"增量流"**

### 核心理由

#### 1. **现有安装的升级路径**
- **场景**：已安装运行的用户（SQLite 或 MariaDB）
- **问题**：`createTables()` 只在全新安装且无表时执行
- **结论**：只有通过 `knex_migrations` 才能确保所有用户（新老）都能获得 schema 更新

#### 2. **SQLite 与 MariaDB 的双轨初始化**

| 数据库类型 | 初始化方式 | 表结构来源 |
|-----------|-----------|-----------|
| SQLite | 复制 kuma.db 模板 | 预建模板数据库 |
| MariaDB | 调用 createTables() | 从头创建表 |

**问题**：如果只在 `knex_init_db.js` 添加字段：
- ✅ 新 MariaDB 用户：能获得新字段
- ❌ 新 SQLite 用户：**不能**（模板未更新）
- ❌ 所有老用户：**不能**

**正确方式**：添加到 `knex_migrations`：
- ✅ 所有用户：通过迁移获得新字段
- ✅ 代码只写一次，Knex 自动适配两种数据库

#### 3. **迁移的可追溯性与幂等性**
- Knex 维护 `knex_migrations` 表记录已执行的迁移
- 每个迁移有唯一时间戳，按顺序执行
- 支持回滚（`down` 函数）
- 支持多开发人员并行工作

#### 4. **代码注释的明确警告**
在 [knex_init_db.js L5-L9](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/db/knex_init_db.js#L5-L9)：
```javascript
/**
 * ⚠️⚠️⚠️⚠️⚠️⚠️ DO NOT ADD ANYTHING HERE!
 * IF YOU NEED TO ADD FIELDS, ADD IT TO ./db/knex_migrations
 * See ./db/knex_migrations/README.md for more information
 */
```

### 历史证据：patchList 的废弃

在 [database.js L71-L116](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L71-L116) 可以看到 `patchList`（已标记 `@deprecated`）：
- 早期：SQLite 使用原生 SQL patch 文件
- 后期：统一使用 knex migrations 支持双数据库
- 最后一个转换的 patch：`patch-monitor-tls-info-add-fk.sql`

---

## 8. 原生 SQL 迁移对 SQLite/MariaDB 双支持的风险分析

### 官方规范

在 [knex_migrations/README.md L9](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/db/knex_migrations/README.md#L9) 明确要求：
> "Avoid native SQL syntax, use knex methods, because Uptime Kuma supports SQLite and MariaDB."

### 风险清单

#### 1. **语法不兼容风险**

| 特性 | SQLite | MariaDB |
|------|--------|---------|
| 字符串拼接 | `\|\|` | `CONCAT()` |
| 日期函数 | `DATETIME('now')` | `UTC_TIMESTAMP()` |
| 自增列 | `AUTOINCREMENT` | `AUTO_INCREMENT` |
| 索引类型 | 支持部分索引 | 不支持部分索引 |
| ALTER TABLE | 功能受限 | 功能完整 |
| 删表约束 | 需单独 DROP | 支持 CASCADE |

**示例：错误的原生 SQL**
```javascript
// ❌ MariaDB 会失败
await knex.raw("UPDATE monitor SET created_date = DATETIME('now')");

// ✅ 使用 Knex 或 Database.sqlHourOffset()
await knex("monitor").update({ created_date: knex.fn.now() });
```

#### 2. **数据类型映射差异**

| Knex 类型 | SQLite | MariaDB |
|----------|--------|---------|
| `boolean` | INTEGER (0/1) | TINYINT(1) |
| `datetime` | TEXT | DATETIME |
| `text` | TEXT | TEXT |
| `increments` | INTEGER PRIMARY KEY AUTOINCREMENT | INT UNSIGNED AUTO_INCREMENT |

#### 3. **事务与锁行为差异**
- SQLite：库级锁，DDL 可能阻塞整个数据库
- MariaDB：行级锁，支持在线 DDL

#### 4. **索引能力差异**
参见 [2025-12-22-0121-optimize-important-indexes.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/db/knex_migrations/2025-12-22-0121-optimize-important-indexes.js) 的处理方式：

```javascript
exports.up = async function (knex) {
    const isSQLite = knex.client.dialect === "sqlite3";
    if (isSQLite) {
        // SQLite：可以创建部分索引
        await knex.schema.alterTable("heartbeat", function (table) {
            table.index(["monitor_id", "time"], "monitor_important_time_index", {
                predicate: knex.whereRaw("important = 1"),
            });
        });
    }
    // MariaDB：不支持，跳过或使用其他方案
};
```

### 缓解策略

#### 策略 1：优先使用 Knex Query Builder
```javascript
// ✅ 推荐：Knex 自动适配
await knex.schema.alterTable("monitor", table => {
    table.string("new_field", 255).nullable();
});
```

#### 策略 2：使用方言检测分支
```javascript
const isSQLite = knex.client.dialect === "sqlite3";
if (isSQLite) {
    await knex.raw("SQLite 专用 SQL");
} else {
    await knex.raw("MariaDB 专用 SQL");
}
```

#### 策略 3：使用抽象辅助函数
如 [database.js L835-L841](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L835-L841) 的 `sqlHourOffset()`：
```javascript
static sqlHourOffset() {
    if (Database.dbConfig.type === "sqlite") {
        return "DATETIME('now', ? || ' hours')";
    } else {
        return "DATE_ADD(UTC_TIMESTAMP(), INTERVAL ? HOUR)";
    }
}
```

### 违规后果

1. **开发环境测试通过，生产环境失败**
   - 开发者用 SQLite 测试，用户用 MariaDB 报错

2. **静默数据损坏**
   - 语法通过但语义不同（如日期处理差异）

3. **迁移执行中断**
   - 部分成功部分失败，数据库处于不一致状态

4. **用户数据丢失风险**
   - 失败的迁移可能导致升级流程终止，用户无法更新

---

## 总结

Uptime Kuma 的数据库架构设计体现了对多数据库支持的深度考量：

1. **三层设计**：Setup 引导 → Knex 抽象层 → 数据库特定实现
2. **迁移优先**：所有 schema 变更通过 knex_migrations，确保新旧用户、新旧数据库的一致性
3. **Patch 机制**：通过 Column Compiler Patch 解决 Knex 与 MariaDB 的兼容性问题
4. **嵌入式支持**：Embedded MariaDB 提供 Docker 一键部署体验
5. **SQL 约束**：严格避免原生 SQL，通过 Knex 抽象保证跨数据库兼容

理解这些设计原则对于贡献代码、排查问题、以及确保数据安全都至关重要。
