# Uptime Kuma Monitor Interval 更新数据流分析

## 问题场景

用户在前端编辑一个 HTTP monitor，把 interval 从 60 秒改成 20 秒：
- 列表项立即更新显示新的 interval
- 但详情页的 heartbeat bar 在短时间内仍显示旧数据

---

## 一、完整数据流分析

### 1. 前端保存 (EditMonitor.vue)

**位置**: [EditMonitor.vue](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/pages/EditMonitor.vue#L4176)

```javascript
this.$root.getSocket().emit("editMonitor", this.monitor, (res) => {
    this.processing = false;
    this.$root.toastRes(res);
    this.init();
});
```

**立即更新的数据**:
- 前端本地表单数据
- 成功回调后重新初始化表单

### 2. Socket 事件 & 后端持久化 (server.js)

**位置**: [server.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L800-L970)

```javascript
socket.on("editMonitor", async (monitor, callback) => {
    // ... 权限验证 ...
    
    // 更新数据库字段，包括 interval
    bean.interval = monitor.interval;
    bean.retryInterval = monitor.retryInterval;
    
    await R.store(bean);  // 持久化到数据库
    
    if (await Monitor.isActive(bean.id, bean.active)) {
        await restartMonitor(socket.userID, bean.id);  // 关键：重启 monitor
    }
    
    await server.sendUpdateMonitorIntoList(socket, bean.id);  // 发送更新到列表
    
    callback({ ok: true, ... });
});
```

**立即更新的数据**:
- 数据库中的 monitor 配置（包括 interval）
- 触发 `sendUpdateMonitorIntoList` 更新前端列表

### 3. restartMonitor / startMonitor 逻辑

**位置**: [server.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1878-L1903)

```javascript
async function startMonitor(userID, monitorID) {
    // ... 更新 active 状态 ...
    
    if (monitor.id in server.monitorList) {
        await server.monitorList[monitor.id].stop();  // 先停止旧的 monitor
    }
    
    server.monitorList[monitor.id] = monitor;
    await monitor.start(io);  // 启动新的 monitor 实例
}

async function restartMonitor(userID, monitorID) {
    return await startMonitor(userID, monitorID);
}
```

**关键行为**:
- 调用 `monitor.stop()` 清除旧的 `heartbeatInterval`
- 创建新的 monitor 实例并调用 `start()`
- **注意**: 旧的 heartbeat 历史数据不会被清除

### 4. Monitor.start() / safeBeat 逻辑

**位置**: [monitor.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L410-L1158)

```javascript
async start(io) {
    const beat = async () => {
        let beatInterval = this.interval;  // 使用新的 interval 值
        
        // ... 执行实际的 HTTP 请求 ...
        
        // 发送单个 heartbeat 到前端
        io.to(this.user_id).emit("heartbeat", bean.toJSON());
        Monitor.sendStats(io, this.id, this.user_id);
        
        // 设置下一次心跳
        if (!this.isStop) {
            let intervalRemainingMs = Math.max(1, 
                beatInterval * 1000 - dayjs().diff(dayjs.utc(bean.time)));
            this.heartbeatInterval = setTimeout(safeBeat, intervalRemainingMs);
        }
    };
    
    const safeBeat = async () => {
        try {
            await beat();
        } catch (e) {
            if (!this.isStop) {
                this.heartbeatInterval = setTimeout(safeBeat, this.interval * 1000);
            }
        }
    };
    
    safeBeat();  // 立即执行第一次心跳
}
```

**立即更新的数据**:
- 新的 monitor 实例使用新的 interval 值
- 立即执行第一次心跳检查

**需要等待下一次 heartbeat 的数据**:
- heartbeat bar 显示的历史数据（旧的 60 秒间隔的心跳）
- 新的 20 秒间隔的心跳需要逐步累积

### 5. sendHeartbeatList / Monitor.sendStats

**位置**: [client.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/client.js#L46-L64)

```javascript
async function sendHeartbeatList(socket, monitorID, toUser = false, overwrite = false) {
    let list = await R.getAll(
        `SELECT * FROM heartbeat WHERE monitor_id = ? ORDER BY time DESC LIMIT 100`,
        [monitorID]
    );
    
    let result = list.reverse();
    
    if (toUser) {
        io.to(socket.userID).emit("heartbeatList", monitorID, result, overwrite);
    } else {
        socket.emit("heartbeatList", monitorID, result, overwrite);
    }
}
```

**注意**:
- `editMonitor` 后 **不会** 调用 `sendHeartbeatList`
- 只有在首次连接、重新连接或特定操作时才会发送完整的 heartbeat 列表

### 6. 前端 socket mixin 数据合并

**位置**: [socket.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/mixins/socket.js)

#### monitorList 更新 (立即生效)
```javascript
socket.on("updateMonitorIntoList", (data) => {
    this.assignMonitorUrlParser(data);
    Object.entries(data).forEach(([monitorID, updatedMonitor]) => {
        this.monitorList[monitorID] = updatedMonitor;  // 立即替换
    });
});
```

#### heartbeatList 更新 (增量合并)
```javascript
socket.on("heartbeat", (data) => {
    if (!(data.monitorID in this.heartbeatList)) {
        this.heartbeatList[data.monitorID] = [];
    }
    
    this.heartbeatList[data.monitorID].push(data);  // 追加新数据
    
    if (this.heartbeatList[data.monitorID].length >= 150) {
        this.heartbeatList[data.monitorID].shift();  // 超过 150 条才移除旧数据
    }
});

socket.on("heartbeatList", (monitorID, data, overwrite = false) => {
    if (!(monitorID in this.heartbeatList) || overwrite) {
        this.heartbeatList[monitorID] = data;
    } else {
        this.heartbeatList[monitorID] = data.concat(this.heartbeatList[monitorID]);
    }
});
```

---

## 二、哪些数据会立即更新

| 数据类型 | 更新时机 | 原因 |
|---------|---------|------|
| **monitorList 中的 interval** | 立即 | `sendUpdateMonitorIntoList` 直接替换整个 monitor 对象 |
| **数据库中的 monitor 配置** | 立即 | `R.store(bean)` 持久化到数据库 |
| **后端 monitor 实例的 interval** | 立即 | `restartMonitor` 创建新实例使用新值 |
| **下一次心跳的执行时间** | 立即 | 新的 `safeBeat` 使用新的 interval |

---

## 三、哪些数据需要等下一次 heartbeat

| 数据类型 | 延迟原因 |
|---------|---------|
| **heartbeat bar 显示的历史数据** | 旧的 60 秒间隔的心跳数据仍在 `heartbeatList` 中，需要等待新的 20 秒间隔的心跳逐步追加进来 |
| **心跳时间轴的密度** | 需要累积足够多的新间隔心跳才能看到明显变化 |
| **avgPing / uptime 统计** | `sendStats` 只在每次心跳后更新，统计值需要时间平滑过渡 |

**HeartbeatBar 渲染逻辑** [HeartbeatBar.vue](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/components/HeartbeatBar.vue):
- 从 `$root.heartbeatList[monitorId]` 读取数据
- 旧数据不会被清除，只会追加新数据
- 最多保留 150 条心跳记录

---

## 四、长请求超过 interval 时是否会并发执行

**答案：不会并发执行**

### 关键机制分析

**位置**: [monitor.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1119-L1129)

```javascript
if (!this.isStop) {
    let intervalRemainingMs = Math.max(1, 
        beatInterval * 1000 - dayjs().diff(dayjs.utc(bean.time)));
    
    // 只有在当前 beat() 完成后才会设置下一次的 setTimeout
    this.heartbeatInterval = setTimeout(safeBeat, intervalRemainingMs);
}
```

### 执行流程

```
1. safeBeat() 被调用
   ↓
2. await beat()  （HTTP 请求可能耗时超过 interval）
   │  ├─ 发送 HTTP 请求
   │  ├─ 等待响应（可能耗时 30 秒，而 interval 是 20 秒）
   │  └─ 存储 heartbeat 记录
   ↓
3. 设置 setTimeout(safeBeat, intervalRemainingMs)
   ↓
4. 等待 intervalRemainingMs 后执行下一次 safeBeat()
```

### 关键保证

1. **串行执行**: `safeBeat` 是 `async` 函数，使用 `await beat()` 确保上一个请求完成后才设置下一个定时器
2. **动态计算间隔**: `intervalRemainingMs = interval - 已用时间`，如果请求耗时 > interval，则间隔为 1ms
3. **单实例**: 每个 monitor 只有一个 `heartbeatInterval` 定时器，`stop()` 时会被 `clearTimeout`

**结论**: 即使 HTTP 请求时间超过 interval，也不会有并发请求，只会"追赶"执行。

---

## 五、Reconnect 后 clearData 与服务端补发数据的一致性分析

### 1. clearData 触发时机

**位置**: [socket.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/mixins/socket.js#L274-L286)

```javascript
socket.on("connect", () => {
    if (this.socket.connectCount >= 2) {
        this.clearData();  // 重连时清空本地 heartbeatList
    }
});

clearData() {
    console.log("reset heartbeat list");
    this.heartbeatList = {};  // 清空所有心跳数据
}
```

### 2. 服务端补发数据

**位置**: [server.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1823-L1829)

```javascript
for (let monitorID in monitorList) {
    monitorPromises.push(sendHeartbeatList(socket, monitorID));  // 补发最近 100 条
    monitorPromises.push(Monitor.sendStats(io, monitorID, user.id));
}
await Promise.all(monitorPromises);
```

### 3. 数据流动时序

```
前端重连:
  ├─ connect 事件触发
  ├─ 执行 clearData() → heartbeatList = {}  （清空本地数据）
  └─ 发送认证请求

后端处理:
  ├─ 认证通过
  ├─ 发送 monitorList
  ├─ 发送 heartbeatList (最近 100 条) → 覆盖 overwrite=false
  └─ 发送 stats (uptime, avgPing)
```

### 4. 一致性效果分析

#### ✅ 消除短暂不一致的场景

**场景 A: 编辑 interval 后立即断线重连**
- 断线前: heartbeatList 混合了 60s 和 20s 间隔的心跳
- 重连后:
  1. `clearData()` 清空所有本地数据
  2. 服务端补发最新的 100 条 heartbeat（包含新间隔的数据）
  3. 前端 `heartbeatList` 被完整替换为服务端数据
- **效果**: 消除了编辑后新旧数据混合的临时不一致状态

**场景 B: 前端离线期间有心跳产生**
- 离线期间: 后端继续产生 heartbeat 但前端无法接收
- 重连后:
  1. `clearData()` 清除旧数据（可能不完整）
  2. 服务端补发最新 100 条，包含离线期间的数据
- **效果**: 确保数据完整性

#### ❌ 可能放大不一致的场景

**场景 C: 频繁断线重连**
- 每次重连都会:
  1. 清空 `heartbeatList`
  2. 请求服务端 100 条数据
  3. 前端重新渲染
- **效果**: heartbeat bar 频繁闪烁，用户体验下降

**场景 D: 超过 100 条的历史数据丢失**
- 前端本地最多保留 150 条心跳
- 服务端每次只补发 100 条
- 重连后: 丢失 50 条最旧的历史数据
- **效果**: 历史视图变短，长时间跨度的可视化受影响

### 5. overwrite 参数的影响

`sendHeartbeatList(socket, monitorID, toUser, overwrite)`:
- `overwrite = false` (默认): `newData.concat(oldData)` → 新数据在前，旧数据在后
- `overwrite = true`: 直接替换整个数组

**重连时使用 `overwrite=false`**，但由于 `clearData()` 后数组为空，效果等同于 `overwrite=true`。

---

## 六、总结

### 核心结论

1. **Monitor 配置（interval）更新是即时的**：通过 `updateMonitorIntoList` 事件直接替换前端列表数据
2. **Heartbeat 历史数据更新是渐进的**：只能通过 `heartbeat` 事件追加，需要等待新间隔的心跳累积
3. **长请求不会导致并发**：`safeBeat` 使用 `await` 保证串行执行
4. **Reconnect 机制是一把双刃剑**：
   - ✅ 可以清除编辑操作带来的临时数据不一致
   - ❌ 但会导致历史数据丢失和 UI 闪烁

### 代码参考链接

| 模块 | 文件 | 关键行 |
|------|------|--------|
| 前端编辑提交 | [EditMonitor.vue](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/pages/EditMonitor.vue#L4176) | L4176 |
| 后端 editMonitor | [server.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L800) | L800-L970 |
| restartMonitor | [server.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1901) | L1901-L1903 |
| Monitor.start | [monitor.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L410) | L410-L1158 |
| 前端 socket mixin | [socket.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/mixins/socket.js) | L204-L242 |
| HeartbeatBar 组件 | [HeartbeatBar.vue](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/components/HeartbeatBar.vue) | L104-L110 |
