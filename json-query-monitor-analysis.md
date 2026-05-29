# JSON-Query Monitor 三种情况行为分析

## 配置参数
- `accepted_statuscodes=["200-299"]` - 接受 200-299 状态码
- `maxretries=2` - 最大重试次数为 2
- `retryInterval=10` - 重试间隔为 10 秒
- `retryOnlyOnStatusCodeFailure=true` - 仅在状态码失败时重试
- `saveResponse=true` - 保存成功响应
- `saveErrorResponse=true` - 保存错误响应

---

## 三种情况分析

### 情况一：HTTP 500（状态码错误）

#### 执行流程
1. Axios 请求返回 HTTP 500，由于 `accepted_statuscodes=["200-299"]`，`validateStatus` 返回 `false`
2. Axios 抛出异常，进入 [monitor.js#L959](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L959-L960) 的 catch 块
3. 由于是状态码错误（非 JSON query 错误），触发重试逻辑
4. 满足 `retryOnlyOnStatusCodeFailure=true` 且 `maxretries=2`，进行重试

#### 关键变量变化

| 重试次数 | bean.status | bean.msg | retries | response 保存 | important | notification | 下次调度间隔 |
|---------|------------|----------|---------|--------------|-----------|--------------|-------------|
| 第 1 次 | PENDING | `Request failed with status code 500` | 1 | ✅ 保存（saveErrorResponse=true） | ❌ 不标记 | ❌ 不发送 | 10 秒（retryInterval） |
| 第 2 次 | PENDING | `Request failed with status code 500` | 2 | ✅ 保存 | ❌ 不标记 | ❌ 不发送 | 10 秒 |
| 第 3 次 | DOWN | `Request failed with status code 500` | 3 | ✅ 保存 | ✅ 标记（PENDING→DOWN） | ✅ 发送 | 常规 interval |

#### 代码逻辑说明
- 在 [monitor.js#L974-L990](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L974-L990) 判断：
  - `error.message.includes("JSON query does not pass")` 为 `false`（HTTP 500 错误不包含此字符串）
  - 因此执行重试逻辑，`retries++`，状态设为 `PENDING`
  - 重试 2 次后（retries=2），第 3 次失败时 `retries >= maxretries`，状态转为 `DOWN`

---

### 情况二：HTTP 200 但 JSONPath 比较失败

#### 执行流程
1. Axios 请求成功返回 HTTP 200，状态码检查通过
2. 进入 [monitor.js#L705-L723](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L705-L723) 的 json-query 处理分支
3. 调用 `evaluateJsonQuery()`，返回 `status=false`
4. 抛出错误：`JSON query does not pass (comparing ...)`
5. 错误被 catch 捕获，进入重试逻辑判断

#### 关键变量变化

| 重试次数 | bean.status | bean.msg | retries | response 保存 | important | notification | 下次调度间隔 |
|---------|------------|----------|---------|--------------|-----------|--------------|-------------|
| 第 1 次 | DOWN | `JSON query does not pass (comparing ...)` | 0 | ✅ 保存（saveResponse=true） | ✅ 标记（UP→DOWN 或首次） | ✅ 发送 | 常规 interval |

**⚠️ 重要：不重试！** 直接标记为 DOWN。

#### 代码逻辑说明
- 在 [monitor.js#L974-L983](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L974-L983) 判断：
  ```javascript
  const isJsonQueryError =
      typeof error.message === "string" && error.message.includes("JSON query does not pass");

  if (isJsonQueryError) {
      // Don't retry on JSON query failures, mark as DOWN immediately
      retries = 0;
  }
  ```
- 由于 `error.message` 包含 "JSON query does not pass"，`isJsonQueryError=true`
- `retries` 被重置为 0，**跳过重试逻辑**，直接进入后续的 `isImportantBeat` 判断
- 响应保存：因为 HTTP 200 是成功响应，在 [monitor.js#L651-L653](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L651-L653) 执行 `saveResponseData()`

---

### 情况三：HTTP 200 且 JSONPath 通过

#### 执行流程
1. Axios 请求成功返回 HTTP 200
2. 进入 json-query 处理分支
3. 调用 `evaluateJsonQuery()`，返回 `status=true`
4. 设置 `bean.status=UP`，正常流程继续

#### 关键变量变化

| bean.status | bean.msg | retries | response 保存 | important | notification | 下次调度间隔 |
|------------|----------|---------|--------------|-----------|--------------|-------------|
| UP | `JSON query passes (comparing ...)` | 0 | ✅ 保存 | ❌（如果之前就是 UP） | ❌（无状态变化） | 常规 interval |

#### 代码逻辑说明
- 在 [monitor.js#L715-L717](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L715-L717)：
  ```javascript
  if (status) {
      bean.status = UP;
      bean.msg = `JSON query passes (comparing ${response} ${this.jsonPathOperator} ${this.expectedValue})`;
  }
  ```
- 不进入 catch 块，`retries` 被重置为 0
- 响应保存在 [monitor.js#L651-L653](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L651-L653) 执行

---

## 关键设计问题：通过 error.message 判断 JSON 条件失败

### 为什么这样实现？

在 [monitor.js#L978-L979](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L978-L979)：
```javascript
const isJsonQueryError =
    typeof error.message === "string" && error.message.includes("JSON query does not pass");
```

**实现原因：**
1. **简单直接**：在 throw 时构造特定消息，catch 时通过字符串匹配判断错误类型
2. **避免改造 Error 类**：不需要创建自定义 Error 子类或在 Error 对象上附加额外属性
3. **代码位置集中**：JSON query 失败的错误消息在 [monitor.js#L719-L721](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js#L719-L721) 定义，匹配逻辑在 catch 块中

### 维护性隐患

#### 1. **脆弱的字符串匹配（Magic String）**
- **问题**：依赖硬编码的字符串 "JSON query does not pass" 进行匹配
- **风险**：如果未来修改错误消息（如翻译、调整措辞），匹配逻辑会静默失效
- **示例**：将消息改为 "JSON condition failed"，`retryOnlyOnStatusCodeFailure` 功能将完全失效，JSON query 失败也会开始重试

#### 2. **错误来源不明确**
- **问题**：无法区分是 JSON query 比较失败，还是 evaluateJsonQuery 内部抛出的其他错误
- **风险**：`evaluateJsonQuery` 在 [util.ts#L774](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/util.ts#L774) 抛出的其他错误（如 "Empty or undefined response"、"Invalid condition" 等）也包含 "JSON query"，可能被误判
- 实际：当前代码中只有 monitor.js 中抛出的错误包含 "JSON query does not pass"，但这是隐式约定

#### 3. **违反错误处理最佳实践**
- **问题**：错误类型判断应该基于错误类型，而不是错误消息
- **更好的方案**：
  ```javascript
  // 方案一：自定义错误类
  class JsonQueryError extends Error {
      constructor(message) {
          super(message);
          this.name = 'JsonQueryError';
          this.isJsonQueryError = true;
      }
  }

  // catch 时判断
  if (error.isJsonQueryError) { ... }

  // 方案二：在错误对象上附加标记
  const error = new Error(...);
  error.isJsonQueryFailure = true;
  throw error;
  ```

#### 4. **可测试性差**
- **问题**：单元测试需要构造包含特定字符串的错误消息，测试与实现细节耦合
- **影响**：重构时测试容易失败，即使行为正确

#### 5. **国际化风险**
- **问题**：如果未来错误消息支持多语言，字符串匹配将彻底失效
- **风险**：这是一个潜在的技术债务，国际化时必须重构

---

## 总结对照表

| 情况 | bean.status | bean.msg | retries | 重试行为 | response 保存 | important | notification | 下次间隔 |
|-----|------------|----------|---------|----------|--------------|-----------|--------------|---------|
| **HTTP 500** | PENDING→DOWN | Request failed with status code 500 | 1→2→3 | ✅ 重试 2 次 | ✅ | PENDING→DOWN 时标记 | PENDING→DOWN 时发送 | 先 10s 后常规 |
| **HTTP 200 JSON 失败** | DOWN | JSON query does not pass... | 0 | ❌ 不重试 | ✅ | ✅ 立即标记 | ✅ 立即发送 | 常规 interval |
| **HTTP 200 JSON 成功** | UP | JSON query passes... | 0 | - | ✅ | ❌ | ❌ | 常规 interval |

---

## 代码引用

| 功能 | 文件 | 行号 |
|-----|------|------|
| json-query 主逻辑 | [monitor.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js) | L705-L723 |
| 重试逻辑判断 | [monitor.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js) | L974-L990 |
| evaluateJsonQuery 函数 | [util.ts](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/util.ts) | L691-L776 |
| 响应保存逻辑 | [monitor.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js) | L651-L653, L966-L968 |
| isImportantBeat 判断 | [monitor.js](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/monitor.js) | L1430-L1456 |
