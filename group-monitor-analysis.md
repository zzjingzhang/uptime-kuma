# Group Monitor 深度分析

## 一、场景描述

假设有以下层级结构的 monitor 组：

```
祖父 Group (inactive, 关联 maintenance)
    └── 父 Group (inactive)
        └── 当前 Group (active)
            ├── 子 Monitor A (inactive)
            ├── 子 Monitor B (active, 从未产生 heartbeat)
            ├── 子 Monitor C (active, 上一跳 PENDING)
            └── 子 Monitor D (active, 上一跳 DOWN)
```

## 二、Group Check 的 heartbeat.status/msg 计算

### 核心逻辑

Group Monitor 的状态计算在 [server/monitor-types/group.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/group.js) 的 `check()` 方法中实现。

### 状态优先级规则

状态按最坏情况优先级排序：
- **DOWN** > **PENDING** > **UP**

### 具体分析步骤

1. **过滤 inactive 子 monitor** ([group.js:27-29](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/group.js#L27-L29))
   - 子 Monitor A (inactive) 被完全忽略，不参与状态计算

2. **遍历剩余 active 子 monitor**

   | 子 Monitor | 状态 | 处理逻辑 | 对 worstStatus 的影响 |
   |-----------|------|---------|---------------------|
   | Monitor B | 无 heartbeat | 标记为 PENDING ([group.js:35-40](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/group.js#L35-L40)) | UP → PENDING |
   | Monitor C | PENDING | 标记为 PENDING ([group.js:46-50](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/group.js#L46-L50)) | 保持 PENDING |
   | Monitor D | DOWN | 标记为 DOWN ([group.js:43-45](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/group.js#L43-L45)) | PENDING → DOWN |

3. **最终结果**
   - **heartbeat.status**: `DOWN`
   - **heartbeat.msg**: `Child monitors down: Monitor D; pending: Monitor B, Monitor C`

### 关键代码片段

```javascript
// group.js:22-52
let worstStatus = UP;
const downChildren = [];
const pendingChildren = [];

for (const child of children) {
    if (!child.active) {
        // Ignore inactive (=paused) children
        continue;
    }

    const label = child.name || `#${child.id}`;
    const lastBeat = await Monitor.getPreviousHeartbeat(child.id);

    if (!lastBeat) {
        if (worstStatus === UP) {
            worstStatus = PENDING;
        }
        pendingChildren.push(label);
        continue;
    }

    if (lastBeat.status === DOWN) {
        worstStatus = DOWN;
        downChildren.push(label);
    } else if (lastBeat.status === PENDING) {
        if (worstStatus !== DOWN) {
            worstStatus = PENDING;
        }
        pendingChildren.push(label);
    }
}
```

## 三、泛型 retry/notification 流程是否参与

### Group Monitor 的特殊处理

**答案：是，完全参与。**

Group Monitor 通过 **抛出 Error** 来利用通用的 retry 和 notification 流程：

```javascript
// group.js:72-73
// Throw to leverage the generic retry handling and notification flow
throw new Error(message);
```

### 完整流程

1. **Group check** 发现有子 monitor DOWN 或 PENDING
2. **抛出 Error** 进入 catch 块 ([monitor.js:959-1000](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L959-L1000))
3. **通用 retry 逻辑**生效：
   - 如果 `maxretries > 0 && retries < maxretries`，状态设为 **PENDING**
   - 否则继续为 **DOWN**
4. **notification 逻辑**生效：
   - 通过 `isImportantBeat()` 和 `isImportantForNotification()` 判断是否发送通知
   - 支持 `resendInterval` 重复通知

### 状态转换示例

假设 Group 配置 `maxretries = 2`：

| 重试次数 | 状态 | 说明 |
|---------|------|------|
| 第1次检测 | PENDING | retries = 1 < 2 |
| 第2次检测 | PENDING | retries = 2 < 2？不，等于，所以 DOWN |
| 第3次检测 | DOWN | retries = 3，触发通知 |

## 四、Monitor.isActive()/isParentActive() 如何影响启动

### isActive() 方法

[server/model/monitor.js:1347-1351](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1347-L1351)

```javascript
static async isActive(monitorID, active) {
    const parentActive = await Monitor.isParentActive(monitorID);
    return active === 1 && parentActive;
}
```

### isParentActive() 方法

[server/model/monitor.js:2065-2074](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L2065-L2074)

```javascript
static async isParentActive(monitorID) {
    const parent = await Monitor.getParent(monitorID);

    if (parent === null) {
        return true;
    }

    const parentActive = await Monitor.isParentActive(parent.id);
    return parent.active === 1 && parentActive;
}
```

### 递归逻辑说明

`isParentActive()` 是**递归向上**检查所有祖先：
1. 获取当前 monitor 的 parent
2. 如果没有 parent，返回 `true`（根节点默认为 active）
3. 递归检查 parent 的 parent 是否 active
4. 返回 `parent.active === 1 && parentActive`（与运算，全部 active 才为 true）

### 对启动的影响

在场景中，当前 Group 的启动状态：

```
祖父 Group (active=0) → false
    └── 父 Group (active=0) → 0 && false = false
        └── 当前 Group (active=1) → 1 && false = false
```

**结论：当前 Group 不会被启动**，因为 `isActive()` 返回 `false`。

### 启动时机

`isActive()` 在 [server/model/monitor.js:1873](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1873) 被调用，用于决定 monitor 是否加入运行列表。

## 五、Monitor.isUnderMaintenance() 如何递归到父级

### isUnderMaintenance() 方法

[server/model/monitor.js:1641-1663](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1641-L1663)

```javascript
static async isUnderMaintenance(monitorID) {
    // 1. 检查自身是否有关联的 maintenance 且处于 maintenance 状态
    const maintenanceIDList = await R.getCol(
        `SELECT maintenance_id FROM monitor_maintenance WHERE monitor_id = ?`,
        [monitorID]
    );

    for (const maintenanceID of maintenanceIDList) {
        const maintenance = await UptimeKumaServer.getInstance().getMaintenance(maintenanceID);
        if (maintenance && (await maintenance.isUnderMaintenance())) {
            return true;
        }
    }

    // 2. 递归检查父级
    const parent = await Monitor.getParent(monitorID);
    if (parent != null) {
        return await Monitor.isUnderMaintenance(parent.id);
    }

    return false;
}
```

### 递归逻辑说明

`isUnderMaintenance()` 采用**短路或**逻辑递归向上：
1. 先检查自身是否在 maintenance 中，如果是，立即返回 `true`
2. 否则获取 parent，递归调用 `isUnderMaintenance(parent.id)`
3. 如果没有 parent，返回 `false`

### 场景分析

在场景中，检查流程：

```
当前 Group → 无 maintenance → 递归检查父 Group
父 Group → 无 maintenance → 递归检查祖父 Group
祖父 Group → 有 maintenance 且生效 → 返回 true
```

**结论：当前 Group 处于 maintenance 状态**，即使它自己没有直接关联 maintenance。

### 对心跳的影响

在 beat 流程中，首先检查 maintenance：

```javascript
// monitor.js:466-468
if (await Monitor.isUnderMaintenance(this.id)) {
    bean.msg = "Monitor under maintenance";
    bean.status = MAINTENANCE;
}
```

如果返回 `true`，心跳状态直接设为 **MAINTENANCE**，跳过所有实际检查。

## 六、前端拖拽修改 parent 时如何避免形成环

### 两层防护机制

#### 1. 前端基础检查

[src/components/MonitorListItem.vue:263-266](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/components/MonitorListItem.vue#L263-L266)

```javascript
const draggedMonitorId = parseInt(draggedId);
if (isNaN(draggedMonitorId) || draggedMonitorId === this.monitor.id) {
    return;
}
```

**前端检查**：不能拖拽到自己下面。

#### 2. 后端严格检查（核心）

[server/server.js:811-817](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L811-L817)

```javascript
// Check if Parent is Descendant (would cause endless loop)
if (monitor.parent !== null) {
    const childIDs = await Monitor.getAllChildrenIDs(monitor.id);
    if (childIDs.includes(monitor.parent)) {
        throw new Error("Invalid Monitor Group");
    }
}
```

### getAllChildrenIDs() 递归获取所有后代

[server/model/monitor.js:1991-2006](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1991-L2006)

```javascript
static async getAllChildrenIDs(monitorID) {
    const childs = await Monitor.getChildren(monitorID);

    if (childs === null) {
        return [];
    }

    let childrenIDs = [];

    for (const child of childs) {
        childrenIDs.push(child.id);
        childrenIDs = childrenIDs.concat(await Monitor.getAllChildrenIDs(child.id));
    }

    return childrenIDs;
}
```

### 防环原理

**核心思想**：要把 A 拖到 B 下面，先获取 A 的**所有后代 ID**，如果 B 的 ID 在这个列表中，说明会形成环。

```
举个反例（会形成环的情况）：
    A (当前要修改 parent 的 monitor)
    └── B
        └── C

如果尝试把 A 的 parent 设为 B：
1. getAllChildrenIDs(A.id) → [B.id, C.id]
2. 检查 B.id 是否在 [B.id, C.id] 中 → 是！
3. 抛出 "Invalid Monitor Group" 错误
```

这样就防止了：A → B → A 的循环引用。

### 错误提示

如果检测到循环，后端抛出错误，前端通过 toast 显示：

```javascript
// MonitorListItem.vue:291-293
if (res && res.msg) {
    this.$root.toastError(res.msg);
}
```

## 七、总结

| 问题 | 结论 | 关键代码位置 |
|-----|------|-------------|
| Group heartbeat.status/msg | 取 active 子 monitor 的最坏状态，DOWN > PENDING > UP | [group.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/group.js) |
| 泛型 retry/notification | 完全参与，通过 throw Error 触发 | [group.js:72-73](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/group.js#L72-L73) |
| isActive()/isParentActive() | 递归向上与运算，任一祖先 inactive 则整个分支不启动 | [monitor.js:1347-1351](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1347-L1351) |
| isUnderMaintenance() | 递归向上或运算，任一祖先 maintenance 则整个分支 maintenance | [monitor.js:1641-1663](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1641-L1663) |
| 拖拽防环 | 后端检查：新 parent 是否在当前 monitor 的后代列表中 | [server.js:811-817](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L811-L817) |
