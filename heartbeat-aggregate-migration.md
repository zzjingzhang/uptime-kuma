# Heartbeat Aggregate Table Migration (stat_minutely / stat_hourly / stat_daily)

## 1. 执行顺序 (Execution Order in Database.patch())

`Database.patch()` 方法中的执行顺序如下：

### 1.1 第一步：旧 SQLite 补丁 (patchSqlite())
- **位置**: [database.js#L503-L507](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L503-L507)
- **仅适用于**: SQLite 数据库
- **作用**: 执行旧版本的 SQL 补丁文件（位于 `db/old_migrations/`）
- **逻辑**: 按版本号顺序执行 `patch1.sql` 到 `patch{n}.sql` 文件

### 1.2 第二步：Knex 迁移 (knex.migrate.latest())
- **位置**: [database.js#L519-L521](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L519-L521)
- **作用**: 创建 `stat_minutely`、`stat_hourly`、`stat_daily` 表结构
- **关键迁移文件**:
  - `2023-08-16-0000-create-uptime.js - 创建 `stat_minutely` 和 `stat_daily`
  - `2023-12-22-0000-hourly-uptime.js - 创建 `stat_hourly`
  - `2023-12-21-0000-stat-ping-min-max.js - 添加 ping_min/ping_max 字段
  - `2024-01-22-0000-stats-extras.js - 添加 extras 字段

### 1.3 第三步：数据迁移 (migrateAggregateTable())
- **位置**: [database.js#L528](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L528)
- **作用**: 将旧 `heartbeat` 表中的历史数据聚合写入到 `stat_*` 表

```javascript
// 简化的执行流程：
patch() {
    if (sqlite) await patchSqlite();      // Step 1
    await knex.migrate.latest();            // Step 2 - 创建 stat_* 表结构
    await migrateAggregateTable();              // Step 3 - 数据迁移
}
```

---

## 2. 数据迁移流程：按 monitor/date 读取 heartbeat

### 2.1 读取策略

在 [migrateAggregateTable()](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L854-L981) 中的读取逻辑：

**第一步：获取所有 monitor:
```sql
SELECT DISTINCT monitor_id FROM heartbeat ORDER BY monitor_id ASC
```

**第二步：对每个 monitor，获取所有日期：
```sql
SELECT DISTINCT DATE(time) AS date
FROM heartbeat
WHERE monitor_id = ?
ORDER BY date ASC
```

**第三步：对每个 (monitor, date)，获取该天的所有 heartbeat：
```sql
SELECT status, ping, time
FROM heartbeat
WHERE monitor_id = ? AND DATE(time) = ?
ORDER BY time ASC
```

### 2.2 调用 UptimeCalculator.update()

对于每个 (monitor, date) 组合：

1. 为每个 monitor-date 每天创建一个新的 `UptimeCalculator` 实例
2. 设置 `migrationMode = true`
3. 遍历所有 heartbeat，逐条调用 `calculator.update(status, ping, time)

**关键代码** [database.js#L930-L956](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L930-L956):

```javascript
for (const [i, monitor] of monitors.entries()) {
    for (const [dateIndex, date] of dates.entries()) {
        let calculator = new UptimeCalculator();
        calculator.monitorID = monitor.monitor_id;
        calculator.setMigrationMode(true);

        let heartbeats = await R.getAll(...);

        for (let heartbeat of heartbeats) {
            await calculator.update(
                heartbeat.status,
                parseFloat(heartbeat.ping),
                dayjs(heartbeat.time)
            );
        }
    }
}
```

**设计意图**：
- 按天分组处理可以限制内存使用
- 每天使用独立的计算器实例，避免跨天数据污染
- 按时间升序处理保证统计正确

---

## 3. migrationMode 跳过部分 minutely/hourly 写入的原因

### 3.1 核心逻辑位于 [UptimeCalculator.update()](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/uptime-calculator.js#L317-L371)

### 3.2 写入条件判断：

**stat_daily - 始终写入（无条件）：
```javascript
// 无条件写入 daily 数据
let dailyStatBean = await this.getDailyStatBean(dailyKey);
dailyStatBean.up = dailyData.up;
// ... 其他字段
await R.store(dailyStatBean);
```

**stat_hourly - 有条件写入：
```javascript
if (!this.migrationMode || date.isAfter(currentDate.subtract(this.statHourlyKeepDay, "day"))) {
    // 写入 hourly 数据
}
```

**stat_minutely - 有条件写入：
```javascript
if (!this.migrationMode || date.isAfter(currentDate.subtract(this.statMinutelyKeepHour, "hour"))) {
    // 写入 minutely 数据
}
```

### 3.3 配置参数：
- `statHourlyKeepDay = 30` - 保留 30 天的 hourly 数据
- `statMinutelyKeepHour = 24` - 保留 24 小时的 minutely 数据

### 3.4 为什么要跳过？

**原因**：
1. **性能优化：迁移时处理的是历史数据，对于超过保留期的旧数据：
   - 超过 30 天的历史数据，只需要 daily 聚合，不需要 hourly 细粒度
   - 超过 24 小时的历史数据，只需要 hourly 聚合，不需要 minutely 细粒度
2. **存储空间**：减少不必要的数据库写入量，提升迁移速度
3. **数据保留策略与正常运行时一致：正常运行时也会定期清理超过保留期的 minutely/hourly 数据

**总结**：migrationMode 下，对于超过保留期的旧数据，只保留最粗粒度的 daily 统计，跳过细粒度的 hourly/minutely 统计。

---

## 4. 迁移后清理非 important heartbeat 的原因

### 4.1 清理逻辑位于 [clearHeartbeatData()](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L988-L1015)

### 4.2 清理 SQL：
```sql
DELETE FROM heartbeat
WHERE monitor_id = ?
  AND important = 0
  AND time < (NOW() - 24小时)
  AND id NOT IN (SELECT id FROM heartbeat
                WHERE monitor_id = ?
                ORDER BY time DESC
                LIMIT 100)
```

### 4.3 `important` 字段的含义：
- `important = 1`：状态转换的 heartbeat（如 UP→DOWN、DOWN→UP）
- `important = 0`：常规心跳，状态未变化

### 4.4 为什么清理？

1. **数据已聚合**：所有心跳数据已经聚合到 `stat_*` 表中，用于 uptime 计算不再依赖原始 heartbeat
2. **节省空间**：删除大量非重要 heartbeat 可以显著减小数据库体积
3. **保留必要数据**：
   - 保留 `important = 1` 的状态转换记录（用于事件日志）
   - 保留最近 24 小时的所有 heartbeat
   - 每个 monitor 至少保留最近 100 条 heartbeat
4. **正常运行时也会定期执行**：这是日常数据清理的常规操作

---

## 5. 迁移中断后再次启动的处理

### 5.1 迁移状态机位于 [migrateAggregateTable()](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/database.js#L866-L876)

### 5.2 状态值：

| 状态 | 含义 |
|------|------|
| 空 | 未开始 |
| `migrating` | 迁移中 |
| `migrated` | 已完成 |

### 5.3 中断后的行为：

```javascript
let migrateState = await Settings.get("migrateAggregateTableState");

if (migrateState === "migrated") {
    // 已完成，直接跳过
    return;
} else if (migrateState === "migrating") {
    // 迁移中（可能是中断了）
    log.warn("db", "Aggregate table migration is already in progress, or it was interrupted");
    throw new Error("Aggregate table migration is already in progress");
}
```

### 5.4 为什么抛出错误而不是自动重试？

**原因**：
- 迁移没有使用事务（代码注释说明：UptimeCalculator 原本不是为事务设计的）
- 中断后 stat_* 表中可能存在部分数据
- 无法确定哪些数据已迁移，哪些未迁移
- 为了避免数据重复或不一致，不允许自动继续

### 5.5 恢复方法：

运行重置脚本 [reset-migrate-aggregate-table-state.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/extra/reset-migrate-aggregate-table-state.js)

```bash
npm run reset-migrate-aggregate-table-state
```

脚本执行的操作：
1. `DELETE FROM stat_minutely`
2. `DELETE FROM stat_hourly`
3. `DELETE FROM stat_daily`
4. `Settings.set("migrateAggregateTableState", "")`

重置后重新启动 Uptime Kuma，迁移会从头开始。

### 5.6 迁移前的安全检查：

在开始迁移前，还会检查 stat_* 表是否为空：
```javascript
for (let table of ["stat_minutely", "stat_hourly", "stat_daily"]) {
    let countResult = await R.getRow(`SELECT COUNT(*) AS count FROM ${table}`);
    if (count > 0) {
        log.warn("db", `Aggregate table ${table} is not empty, migration will not be started`);
        return;
    }
}
```

如果 stat_* 表已有数据（如 2.0.0-dev 用户），迁移会被跳过。

---

## 总结

### 迁移流程图

```
启动
  ↓
patchSqlite() (SQLite 旧补丁
  ↓
knex.migrate.latest() (创建 stat_* 表结构)
  ↓
migrateAggregateTable()
  │
  ├─ 检查 migrateAggregateTableState
  │   ├─ "migrated" → 跳过
  │   └─ "migrating" → 抛错误，需手动重置
  │
  ├─ 检查 stat_* 表是否为空 → 非空则跳过
  │
  └─ 设置状态为 "migrating"
  │
  └─ 按 monitor → 按 date → 逐条 heartbeat → UptimeCalculator.update()
  │       │
  │       └─ migrationMode=true
  │       └─ daily: 全部写入
  │       └─ hourly: 仅最近 30 天写入
  │       └─ minutely: 仅最近 24 小时写入
  │
  └─ clearHeartbeatData(true) (清理非重要 heartbeat
  │
  └─ 设置状态为 "migrated"
  ↓
完成
```

