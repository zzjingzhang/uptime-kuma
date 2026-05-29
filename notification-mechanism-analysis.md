# HTTPS Monitor 通知机制详解

本文档围绕一个 HTTPS monitor 经历 DOWN → UP 恢复、证书被替换、域名到期天数跨过多个提醒阈值的完整场景，逐层剖析通知机制的实现细节。

---

## 1. 恢复通知如何找到 lastDownTime

### 触发入口

在 [monitor.js](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1005-L1021) 中，每次心跳结束后，系统判断当前心跳是否为"重要的"（状态变更），若是且符合通知条件，则调用 `Monitor.sendNotification()`。

判断逻辑分两层：

1. **`isImportantBeat`**（[L1430-L1456](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1430-L1456)）：判断是否为状态转换心跳。DOWN → UP、UP → DOWN、PENDING → DOWN、MAINTENANCE 相关转换等都返回 `true`。
2. **`isImportantForNotification`**（[L1465-L1488](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1465-L1488)）：在 `isImportantBeat` 基础上进一步过滤——排除 MAINTENANCE → UP、DOWN → MAINTENANCE、UP → MAINTENANCE，确保维护期间的状态变化不触发通知。

当 `bean.status === UP` 且之前为 DOWN 时，两者都返回 `true`，进入 `sendNotification`。

### lastDownTime 的查询

在 [monitor.js#L1528-L1548](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1528-L1548)，恢复通知构造 `heartbeatJSON` 时：

```sql
SELECT time FROM heartbeat
WHERE monitor_id = ? AND status = ? AND important = 1
ORDER BY time DESC LIMIT 1
```

参数为 `[monitor.id, DOWN]`。

### 为什么要查 `important = 1` 的 DOWN heartbeat？

`important = 1` 标记的是**状态转换心跳**，而非普通心跳。一个 DOWN 周期可能产生大量 DOWN 心跳（每次轮询一条），但只有**第一条**（从 UP/PENDING 变为 DOWN 的那条）`important = 1`。

如果查询 `important` 无限制的最近 DOWN 心跳，拿到的是**中断结束前最后一次轮询的时间**，而不是**中断开始的时间**。只有 `important = 1` 的那条心跳的 `time` 才代表服务真正**开始宕机的时刻**，即 `lastDownTime`。

恢复通知将 `lastDownTime` 写入 `heartbeatJSON`，所有 Notification Provider 即可据此计算宕机持续时长。

---

## 2. TLS 信息如何通过 keylog/secureConnect 与 fallback socket 更新

### HTTP monitor 的双路获取

对于 HTTP/HTTPS 类型 monitor，TLS 信息的获取依赖 Node.js 的底层事件机制，位于 [monitor.js#L625-L669](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L625-L669)。

#### 主路径：keylog + secureConnect

```javascript
options.httpsAgent.once("keylog", async (line, tlsSocket) => {
    tlsSocket.once("secureConnect", async () => {
        tlsInfo = checkCertificate(tlsSocket);
        tlsInfo.valid = tlsSocket.authorized || false;
        tlsInfo.hostnameMatchMonitorUrl = checkCertificateHostname(
            tlsInfo.certInfo.raw, this.getUrl()?.hostname
        );
        await this.handleTlsInfo(tlsInfo);
    });
});
```

`httpsAgent` 的 `keylog` 事件是获取底层 `tls.TLSSocket` 的**唯一途径**——Axios 封装后并不直接暴露 TLS socket。在 `keylog` 回调中拿到 `tlsSocket` 后，监听其 `secureConnect` 事件（TLS 握手完成），调用 [checkCertificate()](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/util-server.js#L498-L520) 提取证书信息，再通过 `handleTlsInfo()` 存储。

#### fallback 路径：response socket

```javascript
if (this.getUrl()?.protocol === "https:" && tlsInfo.valid === undefined) {
    const tlsSocket = res.request.res.socket;
    if (tlsSocket) {
        tlsInfo = checkCertificate(tlsSocket);
        tlsInfo.valid = tlsSocket.authorized || false;
        tlsInfo.hostnameMatchMonitorUrl = checkCertificateHostname(
            tlsInfo.certInfo.raw, this.getUrl()?.hostname
        );
        await this.handleTlsInfo(tlsInfo);
    }
}
```

当连接经过代理（proxy）时，`keylog` 事件可能不会被触发。此时 `tlsInfo.valid` 仍为 `undefined`，代码检测到协议为 `https:` 且主路径未成功后，从 `res.request.res.socket` 获取 response 的底层 socket 作为 fallback，再次提取 TLS 信息。

### TCP monitor 的直接获取

对于 TCP 类型的 monitor（含 SMTP/IMAP 等），TLS 信息的获取在 [tcp.js#L236-L279](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/tcp.js#L236-L279)：

```javascript
socket = tls.connect(options);
socket.on("secureConnect", () => {
    const info = checkCertificate(socket);
    resolve(info);
});
```

TCP monitor 使用 `tls.connect()` 直接建立 TLS 连接，`secureConnect` 事件触发后即可从 socket 提取证书信息。对于 STARTTLS，先通过 [performStartTls()](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/tcp.js#L154-L228) 完成 SMTP/IMAP/XMPP 协议的 TLS 协商，再将复用的 socket 传入 `tls.connect()` 的 `reuseSocket` 参数。

### checkCertificate 做了什么

[util-server.js#L498-L520](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/util-server.js#L498-L520)：调用 `socket.getPeerCertificate(true)` 获取完整证书链，经 `parseCertificateInfo` 解析后返回 `{ valid, certInfo }` 结构。

### handleTlsInfo 流程

[monitor.js#L2107-L2115](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L2107-L2115)：

1. 调用 `updateTlsInfo(tlsInfo)` 存入 `monitor_tls_info` 表，同时比较指纹决定是否清理通知历史
2. 更新 Prometheus 指标
3. 若 TLS 过期通知已启用，调用 `checkCertExpiryNotifications()`

---

## 3. 证书 fingerprint 变化为什么清理 certificate notification_sent_history

### 触发位置

在 [monitor.js#L1303-L1333](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1303-L1333) 的 `updateTlsInfo` 方法中：

```javascript
if (oldCertInfo.certInfo.fingerprint256 !== checkCertificateResult.certInfo.fingerprint256) {
    await R.exec(
        "DELETE FROM notification_sent_history WHERE type = 'certificate' AND monitor_id = ?",
        [this.id]
    );
}
```

### 为什么需要清理

`notification_sent_history` 表的结构（[patch-notification_sent_history.sql](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/db/old_migrations/patch-notification_sent_history.sql)）：

| 列 | 说明 |
|---|---|
| `type` | 通知类型（如 `'certificate'`） |
| `monitor_id` | 监控器 ID |
| `days` | 已发送通知的阈值天数 |

UNIQUE 约束：`(type, monitor_id, days)`，保证同一个 monitor 对同一个天数阈值只发送一次通知。

**清理原因**：证书被替换后，新证书有新的过期时间。旧证书的 `notification_sent_history` 记录的是基于旧证书过期日的阈值通知状态，不再适用于新证书。如果不清理，新证书即使快要过期，由于 `notification_sent_history` 中已存在对应 `days` 的记录，`sendCertNotificationByTargetDays`（[monitor.js#L1589-L1624](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1589-L1624)）会误判"已发送"而跳过通知。

指纹对比 (`fingerprint256`) 是判断证书是否更换的可靠方式——SHA-256 指纹在全局唯一标识一张证书。

---

## 4. DomainExpiry 如何查找/创建 domain、计算 daysRemaining 并按 targetDays 发送通知

### 整体调用链

在 [monitor.js#L1050-L1060](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1050-L1060)，每次心跳若 `domainExpiryNotification` 启用且非维护状态：

```javascript
const supportInfo = await DomainExpiry.checkSupport(this);
const domainExpiryDate = await DomainExpiry.checkExpiry(supportInfo.domain);
if (domainExpiryDate) {
    DomainExpiry.sendNotifications(supportInfo.domain, (await Monitor.getNotificationList(this)) || []);
}
```

### checkSupport — 验证域名是否支持到期查询

[domain_expiry.js#L211-L245](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/domain_expiry.js#L211-L245)：

1. 检查 monitor 类型是否支持域名到期查询（`TYPES_WITH_DOMAIN_EXPIRY_SUPPORT_VIA_FIELD`）
2. 从 monitor 的 URL/hostname 字段解析域名
3. 用 `tldts` 解析域名，检查 `isIcann`（排除私有 TLD）
4. 通过 IANA RDAP 数据库查找该 TLD 的 RDAP 服务器，若无则抛出 `unsupported_tld_no_rdap_endpoint`

### findByDomainNameOrCreate — 查找或创建 domain 记录

[domain_expiry.js#L251-L257](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/domain_expiry.js#L251-L257)：

```javascript
static async findByDomainNameOrCreate(domainName) {
    let domain = await DomainExpiry.findByName(domainName);    // R.findOne("domain_expiry", "domain = ?", [domain])
    if (!domain && domainName) {
        domain = await DomainExpiry.createByName(domainName);  // R.dispense("domain_expiry")
    }
    return domain;
}
```

先按 `domain` 字段查询 `domain_expiry` 表，若无记录则创建新的空 bean。

### checkExpiry — 获取/缓存到期日期

[domain_expiry.js#L278-L302](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/domain_expiry.js#L278-L302)：

1. 调用 `findByDomainNameOrCreate` 获取/创建 domain bean
2. 若 `lastCheck` 距今不足 1 天，直接返回缓存的 `expiry`（避免频繁查询 RDAP）
3. 否则通过 RDAP 查询新的到期日期
4. **关键**：若新到期日比旧的更晚（域名续期），则重置 `lastExpiryNotificationSent = null`——续期后应该重新通知
5. 更新 `expiry` 和 `lastCheck`，存入数据库

### daysRemaining 计算

[domain_expiry.js#L262-L264](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/domain_expiry.js#L262-L264)：

```javascript
get daysRemaining() {
    return dayjs.utc(this.expiry).diff(dayjs.utc(), "day");
}
```

使用 dayjs 的 UTC 模式计算到期日与当前时间的天数差。

### sendNotifications — 按 targetDays 发送通知

[domain_expiry.js#L309-L365](file:///C:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/domain_expiry.js#L309-L365)：

1. 获取 domain bean 和 `notificationList`，快速失败检查
2. 校验 `expiry` 日期是否有效
3. 获取 `domainExpiryNotifyDays` 设置（默认 `[7, 14, 21]`）
4. **升序排序** `notifyDays`，这是关键——确保先处理最小阈值
5. 对每个 `targetDays`：
   - `daysRemaining > targetDays`：跳过（距离到期还远）
   - `lastSent && lastSent <= targetDays`：跳过（已为该阈值或更低阈值发过通知）
   - 否则：调用 `sendDomainNotificationByTargetDays` 发送通知
6. 发送成功后，将 `domain.lastExpiryNotificationSent = targetDays` 并存入数据库，然后 `return`（只发一次，不再检查更高阈值）

**升序排序的意义**：假设 `notifyDays = [7, 14, 21]`，`daysRemaining = 5`：
- `targetDays = 7`：5 ≤ 7 且无 `lastSent` → 发送通知，`lastSent = 7`，return
- 下次心跳再进来时，`targetDays = 7`：5 ≤ 7 但 `lastSent(7) <= 7` → 跳过
- 不会再发送 14、21 天的通知，因为第一次已覆盖

---

## 5. 三种通知类型的去重状态对比

| 维度 | Monitor UP/DOWN 通知 | 证书到期通知 | 域名到期通知 |
|---|---|---|---|
| **去重机制** | `isImportantBeat` + `isImportantForNotification` 状态转换检测 | `notification_sent_history` 表 (`type='certificate'`) | `domain_expiry.lastExpiryNotificationSent` 字段 |
| **存储位置** | heartbeat 表的 `important` 列（隐式） | `notification_sent_history` 表 | `domain_expiry` 表 |
| **去重粒度** | 状态转换（UP↔DOWN），重复 DOWN 由 `resendInterval` 控制 | 每个 `(monitor_id, days)` 阈值只发一次 | 记录最后一次发送的 `targetDays` 值 |
| **重置条件** | 每次状态转换自动重置 | 证书 fingerprint256 变化时清空该 monitor 的全部 certificate 记录 | 域名续期（新到期日 > 旧到期日）时 `lastExpiryNotificationSent = null` |
| **与其它类型共享** | ❌ 不共享 | ❌ 不共享 | ❌ 不共享 |

### 不共享的去重状态

三种通知各自维护独立的去重状态，**互不影响**：

- **UP/DOWN 通知**完全依赖内存和 heartbeat 表中的状态比较，不涉及 `notification_sent_history` 表。
- **证书到期通知**使用 `notification_sent_history` 表且 `type = 'certificate'`，该表还有 UNIQUE 约束 `(type, monitor_id, days)`。但域名到期通知并不使用此表。
- **域名到期通知**使用 `domain_expiry.lastExpiryNotificationSent`（整数类型，存储最近一次发送的 `targetDays` 值），完全独立于 `notification_sent_history`。

因此，即使同一 monitor 同时产生 UP 恢复通知、证书到期通知和域名到期通知，它们的去重判断互不干扰，各自按照自己的逻辑决定是否发送。
