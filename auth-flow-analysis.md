# Uptime Kuma 认证流程深度分析

## 一、核心认证路径分析

### 1. loginByToken 路径

**后端实现**：[server.js:L383-L430](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L383-L430)

```javascript
socket.on("loginByToken", async (token, callback) => {
    try {
        let decoded = jwt.verify(token, server.jwtSecret);
        let user = await R.findOne("user", " username = ? AND active = 1 ", [decoded.username]);

        if (user) {
            // 关键：检查密码是否变更
            if (decoded.h !== shake256(user.password, SHAKE256_LENGTH)) {
                throw new Error("The token is invalid due to password change or old token");
            }
            await afterLogin(socket, user);
            callback({ ok: true });
        }
    } catch (error) {
        callback({ ok: false, msg: "authInvalidToken", msgi18n: true });
    }
});
```

**JWT 结构**：[user.js:L41-L49](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/user.js#L41-L49)

```javascript
static createJWT(user, jwtSecret) {
    return jwt.sign(
        {
            username: user.username,
            h: shake256(user.password, SHAKE256_LENGTH),  // 密码哈希的哈希
        },
        jwtSecret
    );
}
```

**前端调用**：[socket.js:L445-L456](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/mixins/socket.js#L445-L456)

```javascript
loginByToken(token) {
    socket.emit("loginByToken", token, (res) => {
        this.allowLoginDialog = true;
        if (!res.ok) {
            this.logout();
        } else {
            this.loggedIn = true;
            this.username = this.getJWTPayload()?.username;
        }
    });
}
```

---

### 2. login / afterLogin 路径

**login 事件处理**：[server.js:L432-L510](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L432-L510)

```javascript
socket.on("login", async (data, callback) => {
    // 1. Rate Limit 检查
    if (!(await loginRateLimiter.pass(callback))) {
        return;
    }

    // 2. 用户名密码验证
    let user = await login(data.username, data.password);

    if (user) {
        // 3. 2FA 状态检查
        if (user.twofa_status === 0) {
            // 无 2FA，直接登录
            await afterLogin(socket, user);
            callback({
                ok: true,
                token: User.createJWT(user, server.jwtSecret),
            });
        }

        if (user.twofa_status === 1 && !data.token) {
            // 需要 2FA token
            callback({ tokenRequired: true });
        }

        if (data.token) {
            // 验证 2FA token
            let verify = notp.totp.verify(data.token, user.twofa_secret, twoFAVerifyOptions);
            if (user.twofa_last_token !== data.token && verify) {
                await afterLogin(socket, user);
                // 记录最后使用的 token 防止重放
                await R.exec("UPDATE `user` SET twofa_last_token = ? WHERE id = ? ", [data.token, socket.userID]);
                callback({
                    ok: true,
                    token: User.createJWT(user, server.jwtSecret),
                });
            }
        }
    }
});
```

**afterLogin 函数**：[server.js:L1805-L1837](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1805-L1837)

```javascript
async function afterLogin(socket, user) {
    socket.userID = user.id;       // 设置用户 ID 到 socket
    socket.join(user.id);          // 加入用户专属房间

    // 发送初始化数据
    let monitorList = await server.sendMonitorList(socket);
    await Promise.allSettled([
        sendInfo(socket),
        server.sendMaintenanceList(socket),
        sendNotificationList(socket),
        // ... 其他初始化数据
    ]);
}
```

**login 函数（认证核心）**：[auth.js:L15-L34](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L15-L34)

```javascript
exports.login = async function (username, password) {
    let user = await R.findOne("user", "TRIM(username) = ? AND active = 1 ", [username.trim()]);

    if (user && passwordHash.verify(password, user.password)) {
        // 自动升级哈希算法（如需要）
        if (passwordHash.needRehash(user.password)) {
            await R.exec("UPDATE `user` SET password = ? WHERE id = ? ", [
                await passwordHash.generate(password),
                user.id,
            ]);
        }
        return user;
    }
    return null;
};
```

---

### 3. checkLogin 路径

**checkLogin 函数**：[util-server.js:L637-L641](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/util-server.js#L637-L641)

```javascript
exports.checkLogin = (socket) => {
    if (!socket.userID) {
        throw new Error("You are not logged in.");
    }
};
```

**典型使用场景**：
- [changePassword](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1423-L1453) - 修改密码前检查
- [getSettings](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1455-L1474) - 获取设置前检查
- [setSettings](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1476-L1500) - 修改设置前检查
- 所有需要认证的 socket 事件处理函数入口

---

### 4. apiAuth / basic auth 路径

**apiAuth 中间件**：[auth.js:L155-L176](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L155-L176)

```javascript
exports.apiAuth = async function (req, res, next) {
    if (!(await Settings.get("disableAuth"))) {
        let usingAPIKeys = await Settings.get("apiKeysEnabled");
        let middleware;
        if (usingAPIKeys) {
            middleware = basicAuth({
                authorizer: apiAuthorizer,  // API Key 验证
                authorizeAsync: true,
                challenge: true,
            });
        } else {
            middleware = basicAuth({
                authorizer: userAuthorizer, // 用户名密码验证
                authorizeAsync: true,
                challenge: true,
            });
        }
        middleware(req, res, next);
    } else {
        next();  // disableAuth 时跳过认证
    }
};
```

**basicAuth 中间件**：[auth.js:L132-L146](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L132-L146)

```javascript
exports.basicAuth = async function (req, res, next) {
    const middleware = basicAuth({
        authorizer: userAuthorizer,
        authorizeAsync: true,
        challenge: true,
    });

    const disabledAuth = await Settings.get("disableAuth");

    if (!disabledAuth) {
        middleware(req, res, next);
    } else {
        next();
    }
};
```

**apiAuthorizer（API Key 验证）**：[auth.js:L79-L97](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L79-L97)

```javascript
function apiAuthorizer(username, password, callback) {
    apiRateLimiter.pass(null, 0).then((pass) => {
        if (pass) {
            verifyAPIKey(password).then((valid) => {
                callback(null, valid);
                apiRateLimiter.removeTokens(1);
            });
        } else {
            callback(null, false);
        }
    });
}
```

**userAuthorizer（用户名密码验证）**：[auth.js:L106-L123](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L106-L123)

```javascript
function userAuthorizer(username, password, callback) {
    loginRateLimiter.pass(null, 0).then((pass) => {
        if (pass) {
            exports.login(username, password).then((user) => {
                callback(null, user != null);
                if (user == null) {
                    loginRateLimiter.removeTokens(1);
                }
            });
        } else {
            callback(null, false);
        }
    });
}
```

---

### 5. 前端 loginRequired / loginByToken 路径

**Socket 连接初始化**：[socket.js:L122-L145](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/mixins/socket.js#L122-L145)

```javascript
socket.on("autoLogin", () => {
    this.loggedIn = true;
    this.storage().token = "autoLogin";
    this.socket.token = "autoLogin";
    this.allowLoginDialog = false;
});

socket.on("loginRequired", () => {
    let token = this.storage().token;
    if (token && token !== "autoLogin") {
        this.loginByToken(token);  // 尝试用保存的 token 登录
    } else {
        this.$root.storage().removeItem("token");
        this.allowLoginDialog = true;  // 显示登录对话框
    }
});
```

**后端 Socket 连接处理**：[server.js:L1727-L1735](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1727-L1735)

```javascript
if (await setting("disableAuth")) {
    log.info("auth", "Disabled Auth: auto login to admin");
    await afterLogin(socket, await R.findOne("user"));
    socket.emit("autoLogin");
} else {
    socket.emit("loginRequired");
}
```

---

## 二、关键问题深度解析

### 1. 旧 JWT 为什么会失效？

**核心机制**：JWT 中嵌入了密码哈希的二次哈希值 `h`

**失效场景分析**：

1. **密码修改导致失效**
   - JWT 创建时：[user.js:L45](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/model/user.js#L45) 会将 `shake256(user.password, SHAKE256_LENGTH)` 存入 JWT payload 的 `h` 字段
   - loginByToken 验证时：[server.js:L397](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L397) 会比较 JWT 中的 `h` 与当前数据库中用户密码的 `shake256` 哈希值
   - 如果密码被修改（另一个会话调用 `changePassword`），数据库中的密码哈希会变化，导致 `h` 值不匹配，JWT 失效

2. **主动断开所有连接**
   - 修改密码时：[server.js:L1438](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1438) 会调用 `server.disconnectAllSocketClients(user.id, socket.id)` 断开该用户的所有其他 socket 连接
   - 这确保了即使旧 JWT 还在有效期内，其他会话也会被立即断开

3. **JWT 签名验证失败**
   - 如果 `jwtSecret` 被重置（调用 `initJWTSecret`），所有已签发的 JWT 都会因为签名验证失败而失效

**修改密码流程**：[server.js:L1423-L1453](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1423-L1453)

```javascript
socket.on("changePassword", async (password, callback) => {
    checkLogin(socket);
    let user = await doubleCheckPassword(socket, password.currentPassword);
    await user.resetPassword(password.newPassword);  // 更新密码哈希
    
    // 断开该用户的所有其他 socket 连接
    server.disconnectAllSocketClients(user.id, socket.id);
    
    // 返回新 JWT
    callback({
        ok: true,
        token: User.createJWT(user, server.jwtSecret),
    });
});
```

---

### 2. 2FA token 何时被要求？

**2FA 状态字段**：`user.twofa_status`
- `0` = 未启用 2FA
- `1` = 已启用 2FA

**要求 2FA token 的条件**：[server.js:L466-L472](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L466-L472)

```javascript
if (user.twofa_status === 1 && !data.token) {
    log.info("auth", `2FA token required for user ${data.username}. IP=${clientIP}`);
    callback({
        tokenRequired: true,
    });
}
```

**2FA 验证流程**：

1. **首次登录请求**（只有用户名密码）
   - 后端检查 `user.twofa_status === 1`
   - 检查请求中没有 `data.token`
   - 返回 `{ tokenRequired: true }`，不生成 JWT
   - 前端收到后显示 2FA 输入框

2. **二次登录请求**（用户名密码 + 2FA token）
   - 后端验证 2FA token：[server.js:L474-L500](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L474-L500)
   - 检查 `twofa_last_token` 防止重放攻击
   - 验证通过后执行 `afterLogin` 并返回 JWT

**2FA 相关操作的 Rate Limit**：
- `prepare2FA`：[server.js:L528](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L528) - 使用 `twoFaRateLimiter`
- `save2FA`：[server.js:L573](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L573) - 使用 `twoFaRateLimiter`
- `disable2FA`：[server.js:L603](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L603) - 使用 `twoFaRateLimiter`

---

### 3. Rate Limiter 作用在哪些入口？

**Rate Limiter 实现**：[rate-limiter.js:L1-L75](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/rate-limiter.js#L1-L75)

系统定义了三个独立的限速器：

| 限速器 | 配置 | 作用入口 |
|--------|------|----------|
| **loginRateLimiter** | 20次/分钟 | 1. `login` socket 事件 [server.js:L447](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L447)<br>2. `logout` socket 事件 [server.js:L514](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L514)<br>3. Basic Auth `userAuthorizer` [auth.js:L108](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L108) |
| **apiRateLimiter** | 60次/分钟 | 1. API Auth `apiAuthorizer` [auth.js:L81](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L81) |
| **twoFaRateLimiter** | 30次/分钟 | 1. `prepare2FA` socket 事件 [server.js:L528](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L528)<br>2. `save2FA` socket 事件 [server.js:L573](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L573)<br>3. `disable2FA` socket 事件 [server.js:L603](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L603) |

**限速逻辑**：

```javascript
async pass(callback, num = 1) {
    const remainingRequests = await this.removeTokens(num);
    if (remainingRequests < 0) {
        if (callback) {
            callback({
                ok: false,
                msg: this.errorMessage,  // "Too frequently, try again later."
            });
        }
        return false;
    }
    return true;
}
```

**注意**：
- `loginRateLimiter` 在 `userAuthorizer` 中只在认证失败时才扣除令牌 [auth.js:L115](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L115)
- `apiRateLimiter` 在 `apiAuthorizer` 中无论成功失败都扣除令牌 [auth.js:L90](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L90)

---

### 4. disableAuth 如何改变认证边界？

**disableAuth 配置检查**：通过 `Settings.get("disableAuth")` 获取

#### 4.1 Socket 首次连接的认证边界变化

**正常认证模式**（disableAuth = false）：
- Socket 连接后：[server.js:L1732-L1734](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1732-L1734)
  ```javascript
  socket.emit("loginRequired");
  ```
- 前端收到 `loginRequired` 事件后：[socket.js:L137-L145](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/mixins/socket.js#L137-L145)
  - 如有保存的 token，调用 `loginByToken` 自动登录
  - 如无 token，显示登录对话框
- **认证边界**：所有需要 `checkLogin` 的 socket 事件都需要先登录

**禁用认证模式**（disableAuth = true）：
- Socket 连接后：[server.js:L1728-L1731](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1728-L1731)
  ```javascript
  if (await setting("disableAuth")) {
      log.info("auth", "Disabled Auth: auto login to admin");
      await afterLogin(socket, await R.findOne("user"));  // 自动登录第一个用户（admin）
      socket.emit("autoLogin");
  }
  ```
- 前端收到 `autoLogin` 事件后：[socket.js:L130-L135](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/src/mixins/socket.js#L130-L135)
  ```javascript
  socket.on("autoLogin", () => {
      this.loggedIn = true;
      this.storage().token = "autoLogin";
      this.socket.token = "autoLogin";
      this.allowLoginDialog = false;
  });
  ```
- **认证边界**：所有 socket 事件都可以直接访问，因为 `socket.userID` 已被 `afterLogin` 设置

#### 4.2 /metrics API 的认证边界变化

**/metrics API 定义**：[server.js:L337](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L337)

```javascript
app.get("/metrics", apiAuth, prometheusAPIMetrics());
```

**正常认证模式**（disableAuth = false）：
- `apiAuth` 中间件会执行 Basic Auth 验证
- 根据 `apiKeysEnabled` 配置选择：
  - API Key 认证：`apiAuthorizer`（使用 `apiRateLimiter`）
  - 用户名密码认证：`userAuthorizer`（使用 `loginRateLimiter`）
- **认证边界**：必须提供有效的认证信息才能访问 metrics

**禁用认证模式**（disableAuth = true）：
- `apiAuth` 中间件直接跳过认证：[auth.js:L173-L175](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L173-L175)
  ```javascript
  } else {
      next();  // 直接放行
  }
  ```
- **认证边界**：/metrics API 公开访问，无需任何认证

#### 4.3 其他受 disableAuth 影响的边界

**Basic Auth 中间件**：[auth.js:L139-L145](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/auth.js#L139-L145)
- disableAuth = true 时跳过 basic auth 验证

**启用/禁用认证时的安全处理**：[server.js:L1485-L1494](file:///c:/Users/10244/Desktop/0508-under/uptime-kuma/server/server.js#L1485-L1494)

```javascript
// 从禁用认证切换到启用认证时，需要验证密码
if (!currentDisabledAuth && data.disableAuth) {
    await doubleCheckPassword(socket, currentPassword);
}

// 从禁用认证切换到启用认证时，断开所有其他客户端
// GHSA-23q2-5gf8-gjpp
if (currentDisabledAuth && !data.disableAuth) {
    server.disconnectAllSocketClients(socket.userID, socket.id);
}
```

---

## 三、完整认证流程图

```
用户打开页面
    │
    ▼
Socket 连接建立
    │
    ├─► disableAuth = true ──► afterLogin(admin) ──► autoLogin ──► 已登录
    │
    └─► disableAuth = false ──► loginRequired
                                    │
                                    ├─► 有保存的 token ──► loginByToken
                                    │                       │
                                    │                       ├─► JWT 签名验证失败 ──► 显示登录框
                                    │                       ├─► 用户不存在/未激活 ──► 显示登录框
                                    │                       └─► 密码哈希匹配 ──► afterLogin ──► 已登录
                                    │                                                  │
                                    │                                                  └─► 密码哈希不匹配 ──► 失效（密码已改）
                                    │
                                    └─► 无 token ──► 显示登录框
                                                      │
                                                      ▼
                                                输入用户名密码
                                                      │
                                                      ▼
                                                login 事件
                                                      │
                                                      ├─► Rate Limit 检查 ──► 超限 ──► 返回错误
                                                      │
                                                      ├─► 用户名密码验证失败 ──► 返回错误
                                                      │
                                                      └─► 验证成功
                                                              │
                                                              ├─► 2FA 未启用 ──► afterLogin ──► 返回 JWT
                                                              │
                                                              ├─► 2FA 已启用 + 无 token ──► 返回 tokenRequired
                                                              │
                                                              └─► 2FA 已启用 + 有 token
                                                                          │
                                                                          ├─► token 无效 ──► 返回错误
                                                                          └─► token 有效 ──► afterLogin ──► 返回 JWT
```

---

## 四、安全设计要点总结

1. **JWT 绑定密码哈希**：通过在 JWT 中嵌入密码哈希的二次哈希，确保密码修改后所有旧 JWT 立即失效
2. **2FA 重放防护**：记录最后使用的 2FA token，防止同一 token 被重复使用
3. **分级 Rate Limiting**：不同操作使用不同的限速策略，有效防止暴力破解
4. **disableAuth 安全切换**：从禁用认证切换到启用认证时，强制断开所有其他客户端连接
5. **认证边界清晰**：disableAuth 配置统一控制 socket 连接和 HTTP API 的认证行为
6. **密码哈希自动升级**：登录时自动检测并升级旧的哈希算法，保证密码存储安全
