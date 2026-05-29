# DNS Monitor 嵌套条件求值分析

## 1. 条件 UI 层

DNS monitor 的条件编辑 UI 由三个 Vue 组件构成：

- [EditMonitorConditions.vue](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/components/EditMonitorConditions.vue)：顶层容器，管理一个条件/组的扁平数组 `model`
- [EditMonitorCondition.vue](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/components/EditMonitorCondition.vue)：单条条件表达式，包含 `andOr` 下拉框（非首条时显示）、`variable`、`operator`、`value` 四个字段
- [EditMonitorConditionGroup.vue](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/components/EditMonitorConditionGroup.vue)：条件组，拥有 `andOr`（非首条时显示）和 `children` 数组，children 内可递归嵌套组或表达式

### andOr 连接符的归属规则

UI 中 **每条条件/组都自带一个 `andOr` 字段**，表示"本条与 **前一条** 的逻辑关系"：

- **第一条** 条件/组的 `andOr` 在 UI 上不渲染（`v-if="!isFirst"`），但数据模型中仍存在，默认为 `"and"`
- **第二条及以后** 条件/组的 `andOr` 通过下拉框由用户选择 `"and"` 或 `"or"`

### 用户配置 A(AND)、B(OR)、C(AND) 的 JSON 结构

假设用户在根级别添加了三条独立条件，则 JSON 模型如下：

```json
[
  { "type": "expression", "andOr": "and",      "variable": "record", "operator": "contains", "value": "值A" },
  { "type": "expression", "andOr": "or",       "variable": "record", "operator": "contains", "value": "值B" },
  { "type": "expression", "andOr": "and",      "variable": "record", "operator": "contains", "value": "值C" }
]
```

**关键**：`andOr` 挂在每条条件自身上，不是挂在"组"上。这意味着：
- A 的 `andOr = "and"`（首条，实际不参与运算，仅初始化 result）
- B 的 `andOr = "or"`（B 与 A 之间是 OR）
- C 的 `andOr = "and"`（C 与前序结果之间是 AND）

---

## 2. Socket 下发的 monitorTypeList

当客户端连接时，后端通过 [sendMonitorTypeList()](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/client.js#L214-L236) 向前端发送 `monitorTypeList` 事件。对 DNS 类型，其结构为：

```js
{
  dns: {
    supportsConditions: true,
    conditionVariables: [
      {
        id: "record",
        operators: [
          { id: "equals",        caption: "equals" },
          { id: "not_equals",    caption: "not equals" },
          { id: "contains",      caption: "contains" },
          { id: "not_contains",  caption: "not contains" },
          { id: "starts_with",   caption: "starts with" },
          { id: "not_starts_with", caption: "not starts with" },
          { id: "ends_with",     caption: "ends with" },
          { id: "not_ends_with", caption: "not ends with" }
        ]
      }
    ]
  }
}
```

前端在 [socket.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/mixins/socket.js#L146) 监听 `monitorTypeList` 事件并存储到 `this.$root.monitorTypeList`。

[EditMonitor.vue](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/pages/EditMonitor.vue#L3500-L3506) 中通过计算属性获取：

```js
supportsConditions() {
    return this.$root.monitorTypeList[this.monitor.type]?.supportsConditions || false;
},
conditionVariables() {
    return this.$root.monitorTypeList[this.monitor.type]?.conditionVariables || [];
}
```

DNS monitor 仅暴露一个变量 `record`（字符串类型），配备 8 个字符串运算符。

---

## 3. 后端 JSON 序列化

### 保存（前端 → 数据库）

在 [server.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L743) 中：

```js
monitor.conditions = JSON.stringify(monitor.conditions);
```

前端传来的 `monitor.conditions` 是一个 JS 数组，通过 `JSON.stringify` 序列化为字符串存入数据库的 `conditions` 字段。

新建 monitor 时（[server.js L931](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L931)）：

```js
bean.conditions = JSON.stringify(monitor.conditions);
```

同时在 [monitor.js L1720](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L1720) 做了校验：

```js
if (this.conditions) {
    try {
        JSON.parse(this.conditions);
    } catch (e) {
        throw new Error(`Conditions must be valid JSON: ${e.message}`);
    }
}
```

### 读取（数据库 → 前端）

在 [monitor.js L203](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L203) 的 `toJSON()` 中：

```js
conditions: JSON.parse(this.conditions),
```

数据库中的 JSON 字符串被反序列化为 JS 对象后返回给前端。

---

## 4. ConditionExpressionGroup.fromMonitor()

定义在 [expression.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-conditions/expression.js#L59-L69)：

```js
static fromMonitor(monitor) {
    const conditions = JSON.parse(monitor.conditions);
    if (conditions.length === 0) {
        return null;
    }
    const root = new ConditionExpressionGroup();
    processMonitorConditions(conditions, root);
    return root;
}
```

`processMonitorConditions()` 递归遍历原始 JSON 数组：

1. 对每个元素读取其 `andOr` 字段，默认归约为 `"and"`
2. 若 `type === "group"`，创建新的 `ConditionExpressionGroup([], andOr)`，递归处理 children，然后 push 到父组
3. 若 `type === "expression"`，创建 `ConditionExpression(variable, operator, value, andOr)`，push 到父组

### 针对用户配置的 A(AND)、B(OR)、C(AND)，构建结果：

```
root (ConditionExpressionGroup, andOr="and" [无意义，根组无父级])
 ├── children[0]: ConditionExpression  (value="值A", andOr="and")   ← A
 ├── children[1]: ConditionExpression  (value="值B", andOr="or")    ← B
 └── children[2]: ConditionExpression  (value="值C", andOr="and")   ← C
```

**注意**：根组的 `andOr` 字段默认为 `"and"`，但根组没有父级，其 `andOr` 不会被任何求值逻辑使用。

---

## 5. evaluateExpressionGroup() 的求值顺序

定义在 [evaluator.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-conditions/evaluator.js#L38-L78)：

```js
function evaluateExpressionGroup(group, context) {
    let result = null;
    for (const child of group.children) {
        let childResult;
        if (child instanceof ConditionExpression) {
            childResult = evaluateExpression(child, context);
        } else if (child instanceof ConditionExpressionGroup) {
            childResult = evaluateExpressionGroup(child, context);
        }
        if (result === null) {
            result = childResult;
        } else if (child.andOr === LOGICAL.OR) {
            result = result || childResult;
        } else if (child.andOr === LOGICAL.AND) {
            result = result && childResult;
        }
    }
    return result;
}
```

### 逐步求值（A AND, B OR, C AND）

| 步骤 | child | child.andOr | 运算 | result |
|------|-------|-------------|------|--------|
| 1 | A (值A) | "and" | result 为 null，直接赋值 | result = A_result |
| 2 | B (值B) | "or" | result = result \|\| B_result | result = A_result \|\| B_result |
| 3 | C (值C) | "and" | result = result && C_result | result = (A \|\| B) && C |

**最终等价表达式**：`(A || B) && C`

### 与常规布尔优先级的对比

在标准布尔代数中，AND 优先级高于 OR，因此 `A AND B OR C AND` 的常规解析应为 `A && (B || C)` —— 即 AND 先结合。

但本系统的实现采用 **严格从左到右的顺序求值**，`andOr` 完全由用户显式指定在每条条件上，不做任何优先级推断。因此：

- 用户配置 A(AND) B(OR) C(AND) 的实际求值 = **(A ∥ B) & C**
- 若用户期望 **A & (B ∥ C)** 的语义，必须在 UI 中将 B 和 C 放入一个 **条件组（group）**，让 B 和 C 在子组内部以 OR/AND 求值后，子组整体结果再与 A 做 AND

### 嵌套组示例

若用户期望 `A AND (B OR C)`，正确的 UI 配置为：

```json
[
  { "type": "expression", "andOr": "and", "variable": "record", "operator": "contains", "value": "值A" },
  {
    "type": "group",
    "andOr": "and",
    "children": [
      { "type": "expression", "andOr": "and", "variable": "record", "operator": "contains", "value": "值B" },
      { "type": "expression", "andOr": "or",  "variable": "record", "operator": "contains", "value": "值C" }
    ]
  }
]
```

求值过程：
1. 子组内部：`B_result || C_result`（B 首条直接赋值，C 的 andOr="or"）
2. 根组：`A_result && 子组结果`

---

## 6. DnsMonitorType.check() 中 context.record 的值

[dns.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/dns.js#L22-L100) 的 `check()` 方法中，`handleConditions({ record: ... })` 传入的 `context.record` 值因 DNS 记录类型而异：

| 记录类型 | dnsRes 返回类型 | context.record 值 | 求值方式 |
|----------|-----------------|-------------------|----------|
| **A / AAAA / PTR** | `string[]` | 单个 IP/域名字符串，如 `"1.2.3.4"` | `dnsRes.some(record => handleConditions({ record }))`，逐条 DNS 记录分别求值，任一通过即可 |
| **TXT** | `string[][]`（二维数组） | 扁平化后的单个 TXT 字符串片段 | `dnsRes.flat().some(record => handleConditions({ record }))`，先展平再逐条求值 |
| **CNAME** | `string[]` | `dnsRes[0]`（首条 CNAME 值，字符串） | `handleConditions({ record: dnsRes[0] })`，仅对首条求值 |
| **CAA** | `object[]` | `record.issue`（字符串或 `undefined`） | `dnsRes.some(record => handleConditions({ record: record.issue }))`，逐条取 issue 字段传入 |
| **MX** | `object[]` | `record.exchange`（如 `"mx1.example.com"`） | `dnsRes.some(record => handleConditions({ record: record.exchange }))`，逐条取 exchange 字段传入 |
| **NS** | `string[]` | 单个 NS 服务器名字符串 | `dnsRes.some(record => handleConditions({ record }))`，逐条求值 |
| **SOA** | `object`（单对象） | `dnsRes.nsname`（SOA 主域名服务器名） | `handleConditions({ record: dnsRes.nsname })`，仅对 nsname 求值 |
| **SRV** | `object[]` | `record.name`（SRV 目标名） | `dnsRes.some(record => handleConditions({ record: record.name }))`，逐条取 name 字段传入 |

### `.some()` 的含义

对于 A/AAAA/PTR/TXT/MX/NS/SRV/CAA 等多条记录类型，外层使用 `Array.some()`，意味着 **只要有一条 DNS 记录使条件组求值为 `true`，整体即判定为通过**。这与 `every()` 语义相反。

### SOA 和 CNAME 的特殊性

SOA 和 CNAME 返回的不是数组而是单个值/对象，因此直接调用 `handleConditions()` 而非 `.some()`，条件必须对这唯一值成立。

---

## 7. CAA 缺少 issue 字段的边界行为

[dns.js L52-L60](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/monitor-types/dns.js#L52-L60)：

```js
case "CAA":
    dnsMessage = `Records: ${dnsRes
        .map((record) => record.issue)
        .filter(Boolean)
        .join(" | ")}`;
    conditionsResult = dnsRes.some((record) => handleConditions({ record: record.issue }));
    break;
```

### 问题分析

Node.js 的 `dns.resolve()` 解析 CAA 记录时返回对象数组，每个对象可能包含以下字段：

```js
{ critical: 0, issue: "letsencrypt.org" }
{ critical: 0, issue: "comodoca.com" }
{ critical: 0, issuewild: "letsencrypt.org" }   // 无 issue 字段
{ critical: 0 }                                  // issue 和 issuewild 都不存在
```

当某条 CAA 记录 **没有 `issue` 字段** 时：

1. **dnsMessage**：`.map(r => r.issue)` 返回 `undefined`，被 `.filter(Boolean)` 过滤掉，不影响显示消息 —— 这部分是安全的
2. **conditionsResult**：`handleConditions({ record: record.issue })` 传入的 `context.record` 为 `undefined`

### 下游影响

在 `evaluateExpression()` 中，运算符会对 `context["record"]`（即 `undefined`）与用户指定的 `value` 做比较。各运算符的行为：

| 运算符 | `undefined` 时的行为 |
|--------|---------------------|
| `equals` | `undefined === "letsencrypt.org"` → `false` |
| `not_equals` | `undefined !== "letsencrypt.org"` → `true` |
| `contains` | `undefined.indexOf(...)` → **TypeError: Cannot read properties of undefined** |
| `not_contains` | 同上 → **TypeError** |
| `starts_with` | `undefined.startsWith(...)` → **TypeError** |
| `not_starts_with` | 同上 → **TypeError** |
| `ends_with` | `undefined.endsWith(...)` → **TypeError** |
| `not_ends_with` | 同上 → **TypeError** |

### 实际后果

- 若用户使用 `contains` / `starts_with` / `ends_with` 等字符串方法运算符，且 DNS 响应中存在不含 `issue` 的 CAA 记录，则 `handleConditions()` 会抛出 `TypeError`，导致 `dnsRes.some()` 中断
- 由于 `.some()` 在回调抛出异常时会向上传播，`check()` 方法会捕获该异常，heartbeat 状态变为 DOWN，消息为错误信息而非 DNS 记录内容
- 若用户仅使用 `equals` / `not_equals`，则不会崩溃：`undefined === "某值"` 返回 `false`，`undefined !== "某值"` 返回 `true`

### 潜在修复方向

1. 在 `handleConditions` 调用前过滤掉无 `issue` 的 CAA 记录：`dnsRes.filter(r => r.issue).some(...)`
2. 在运算符中增加对 `undefined` / `null` 的防御处理
3. 为 CAA 类型提供额外的 `record.issue` / `record.issuewild` 变量选择

---

## 8. 总结

| 层次 | 关键机制 |
|------|---------|
| **UI 层** | 每条条件/组自带 `andOr` 表示与前一元素的逻辑关系；首条 `andOr` 不显示 |
| **Socket 传输** | `monitorTypeList` 事件下发各 monitor 类型的 `supportsConditions` 和 `conditionVariables`；DNS 仅有一个 `record` 变量配 8 个字符串运算符 |
| **JSON 序列化** | 前端→数据库：`JSON.stringify(conditions数组)`；数据库→前端：`JSON.parse(conditions字符串)` |
| **fromMonitor()** | `JSON.parse` 后递归构建 `ConditionExpressionGroup` 树，`andOr` 存储在每个节点上 |
| **evaluateExpressionGroup()** | 严格从左到右顺序求值，第二个子节点起按其 `andOr` 与累积结果做 `&&` 或 `||`；**不存在运算符优先级** |
| **A(AND) B(OR) C(AND) 实际求值** | `(A ∥ B) & C`，而非标准布尔优先级的 `A & (B ∥ C)` |
| **context.record 差异** | 各 DNS 记录类型传入不同字段：A/AAAA/PTR/NS 传原始字符串；TXT 先展平；CAA 传 `.issue`；MX 传 `.exchange`；SOA 传 `.nsname`；SRV 传 `.name`；CNAME 传首条值 |
| **CAA 边界** | 无 `issue` 字段时 `context.record = undefined`，`contains`/`starts_with`/`ends_with` 系运算符会抛 TypeError 导致监控判定 DOWN |
