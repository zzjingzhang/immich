# Immich Web 鉴权：Cookie / Session 链路追踪

本文按调用顺序梳理 Immich Web 端的认证主线：密码登录 → Cookie 写入 → Token 校验 → Session 表查询 → 会话 `updatedAt` 刷新 → OAuth Backchannel Logout / Session 失效事件 → 前端 `SessionDelete` 跳转。最后回答两个问题。

---

## 1. 密码登录：Cookie 的写入

登录入口在 `POST /auth/login`，由 [auth.controller.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/controllers/auth.controller.ts#L30-L50) 的 `AuthController.login` 处理：

```ts
const body = await this.service.login(loginCredential, loginDetails);
return respondWithCookie(res, body, {
  isSecure: loginDetails.isSecure,
  values: [
    { key: ImmichCookie.AccessToken,   value: body.accessToken },
    { key: ImmichCookie.AuthType,      value: AuthType.Password },
    { key: ImmichCookie.IsAuthenticated, value: 'true' },
  ],
});
```

其中 `this.service.login` 在 [auth.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/auth.service.ts#L59-L76)：

1. 查 `userRepository.getByEmail(email, { withPassword: true })`；
2. `compareBcrypt(password, user.password)` 比对哈希；
3. 成功则 `createLoginResponse(user, loginDetails)`，**生成随机 token → SHA-256 哈希 → 落库 session 表**，并把明文 token 作为 `accessToken` 放在响应体。

Cookie 的属性在 [response.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/utils/response.ts#L6-L32) 中统一定义：

| Cookie | httpOnly | maxAge |
|---|---|---|
| `immich_access_token` | `true` | 400 天 |
| `immich_auth_type` | `true` | 400 天 |
| `immich_is_authenticated` | **`false`**（注释：供前端读取） | 400 天 |

> 关键点：`immich_is_authenticated` 只是一个「有我就说明服务端曾给过登录」的**提示位**。真正的凭据是 `httpOnly` 的 `immich_access_token`（即 **session token 的明文**）。

---

## 2. 请求进入：Token 校验入口

每个带 `@Authenticated()` 的接口会经过 [auth.guard.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/middleware/auth.guard.ts#L78-L109) 的 `AuthGuard.canActivate`：

```ts
request.user = await this.authService.authenticate({
  headers: request.headers,
  queryParams: request.query,
  metadata: { adminRoute, sharedLinkRoute, permission, uri: request.path },
});
```

真正负责「挑哪条鉴权路径」的是 [auth.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/auth.service.ts#L244-L271) 的 `validate()`：

```ts
const shareKey  = headers[SharedLinkKey] || queryParams[SharedLinkKey];
const shareSlug = headers[SharedLinkSlug] || queryParams[SharedLinkSlug];
const session   = headers[UserToken] || headers[SessionToken] ||
                  queryParams[SessionKey] || getBearerToken(headers) ||
                  getCookieToken(headers);   // ← 读 immich_access_token
const apiKey    = headers[ApiKey] || queryParams[ApiKey];

if (shareKey)  return validateSharedLinkKey(shareKey);
if (shareSlug) return validateSharedLinkSlug(shareSlug);
if (session)   return validateSession(session, headers);
if (apiKey)    return validateApiKey(apiKey);
```

> 优先级：`shared link` > `session/cookie` > `api key`。

---

## 3. Session 校验：查表 + 刷新 `updatedAt`

`validateSession` 位于 [auth.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/auth.service.ts#L537-L579)：

```ts
const hashed = cryptoRepository.hashSha256(token);
const session = await sessionRepository.getByToken(hashed);

if (session?.user) {
  const now = DateTime.now();
  const diff = now.diff(DateTime.fromJSDate(session.updatedAt), ['hours']);
  if (diff.hours > 1 || appVersion !== session.appVersion) {
    await sessionRepository.update(session.id, {
      updatedAt: new Date(), appVersion, deviceOS, deviceType,
    });
  }
  // Pin 检查与 hasElevatedPermission 计算（略）
  return { user: session.user, session: { id: session.id, hasElevatedPermission } };
}
throw new UnauthorizedException('Invalid user token');
```

`getByToken` 在 [session.repository.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/repositories/session.repository.ts#L50-L69)：

```sql
SELECT ... FROM session
  JOIN user ON user.id = session.userId AND user.deletedAt IS NULL
 WHERE session.token = ?
   AND (session.expiresAt IS NULL OR session.expiresAt > NOW())
```

也就是说服务端在**查 token 时**同时过滤了：
- token 哈希不存在（例如被 `DELETE FROM session`）→ 无行；
- 行存在但 `expiresAt` 已过 → 也无行。

两种情况最终都会走到 `throw new UnauthorizedException('Invalid user token')`。

`updatedAt` 的刷新是「懒更新」：只有距上次更新超过 1 小时、或 UA 发生变化时才写库。但真正用于「判定是否过期」的字段是 `expiresAt`（只在显式给 `duration` 创建的子 session 上有值；主登录 session 不设 `expiresAt`，靠后台 `cleanup` 任务按 `updatedAt > 90 天` 淘汰，见下文）。

---

## 4. Session 清理与过期

[session.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/session.service.ts#L18-L28) 的 `handleCleanup` 通过 `@OnJob(SessionCleanup)` 周期性触发：

```ts
sessionRepository.cleanup()
  // DELETE FROM session
  // WHERE updatedAt <= NOW() - 90 DAYS
  //    OR (expiresAt IS NOT NULL AND expiresAt <= NOW())
```

这是「会话失效」的两个判定源：
1. `expiresAt` 显式过期（用于子 session）；
2. 90 天未活动（`updatedAt` 停止推进）被后台任务清除。

---

## 5. OAuth Backchannel Logout 与 Session 失效事件

入口在 `POST /auth/oauth/backchannel-logout` → [auth.service.ts](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/auth.service.ts#L90-L122) `backchannelLogout`：

1. `oauthRepository.validateLogoutToken(oauth, logout_token)` 拿到 claims；
2. 必须有 `sub` 或 `sid` 其中之一；
3. `sessionRepository.invalidateOAuth({ oauthSid, oauthId })` 批量 DELETE 匹配的 session，返回 `deletedSessionIds`；
4. **对每个被删的 sessionId 发出事件**：`eventRepository.emit('SessionDelete', { sessionId })`。

除了 backchannel logout，以下路径也会 emit `SessionDelete`：

- 用户登出：[auth.service.ts#L78-L88](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/auth.service.ts#L78-L88) `logout()`：
  ```ts
  await sessionRepository.delete(auth.session.id);
  await eventRepository.emit('SessionDelete', { sessionId: auth.session.id });
  ```
- 会话被管理员/自己在 UI 里删：走 `SessionController.deleteSession` → `sessionRepository.delete(id)`，**注意此处不主动发事件**；但当前浏览器看到的效果等同「后台 401」。
- 修改密码触发 `AuthChangePassword` → [session.service.ts#L84-L87](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/session.service.ts#L84-L87) `invalidateAll({ userId, excludeId: currentSessionId })`；这里**不**发 `SessionDelete`，因为当前会话被保留。

`SessionDelete` 事件由 [notification.service.ts#L233-L237](file:///c:/Users/10244/Desktop/0508-under/immich/server/src/services/notification.service.ts#L233-L237) 订阅，并经 WebSocket 广播给目标连接：

```ts
@OnEvent({ name: 'SessionDelete' })
onSessionDelete({ sessionId }: ArgOf<'SessionDelete'>) {
  setTimeout(
    () => this.websocketRepository.clientSend('on_session_delete', sessionId, sessionId),
    500,
  );
}
```

---

## 6. 前端：`authManager.load` 与 `SessionDelete` 跳转

前端核心在 [auth-manager.svelte.ts](file:///c:/Users/10244/Desktop/0508-under/immich/web/src/lib/managers/auth-manager.svelte.ts)：

```ts
constructor() {
  eventManager.on({
    SessionDelete: () => goto(Route.logout()),   // 无论哪个 session 被删，都跳 logout
  });
}

async load() {
  if (authManager.authenticated) return;
  if (!this.#hasAuthCookie()) return;           // 只看 immich_is_authenticated
  return this.refresh();
}

async refresh() {
  try {
    const [user, preferences] = await Promise.all([getMyUser(), getMyPreferences()]);
    this.#preferences = preferences;
    this.#user        = user;
    eventManager.emit('AuthUserLoaded', user);
  } catch { /* noop */ }
}
```

- `#hasAuthCookie()` 只看 `document.cookie` 里的 `immich_is_authenticated`（该 cookie `httpOnly=false`，所以能读到）。
- 真正的 `immich_access_token` 是 `httpOnly=true`，前端永远读不到；只由浏览器自动带到 `/api/users/me`、`/api/users/me/preferences` 等请求里，由服务端在 `AuthGuard → validate → getCookieToken → validateSession` 链路上取出来并校验。

`authenticated` 的判定是：

```ts
get authenticated() { return !!(this.#user && this.#preferences); }
```

页面路由的守卫在 [auth.ts](file:///c:/Users/10244/Desktop/0508-under/immich/web/src/lib/utils/auth.ts#L13-L28)：

```ts
await authManager.load();
if (publicRoute) return;                              // 共享链接路由直接放行
if (!authManager.authenticated) redirect(Route.login());
```

WebSocket 消息在 [websocket.ts#L81](file:///c:/Users/10244/Desktop/0508-under/immich/web/src/lib/stores/websocket.ts#L81) 转回事件：

```ts
.on('on_session_delete', () => eventManager.emit('SessionDelete'))
```

与 `authManager` 构造器里的监听拼起来，就是「收到后端任何 session 失效通知 → 前端 `goto(Route.logout())`」的闭环。

---

## 7. 综合时序图

```
Browser                                    Server
  │                                         │
  │  POST /auth/login (email,password)      │
  │────────────────────────────────────────▶│ AuthController.login
  │                                         │  └▶ AuthService.login → bcrypt
  │                                         │  └▶ createLoginResponse
  │                                         │      random 32B → sha256 → session 表
  │  Set-Cookie: immich_access_token=…      │
  │  Set-Cookie: immich_auth_type=password  │
  │  Set-Cookie: immich_is_authenticated=true│
  │◀────────────────────────────────────────│
  │                                         │
  │  GET /api/users/me                      │
  │    Cookie: immich_access_token=…        │
  │────────────────────────────────────────▶│ AuthGuard.canActivate
  │                                         │  └▶ AuthService.authenticate
  │                                         │      └▶ validate → getCookieToken
  │                                         │      └▶ validateSession
  │                                         │          └▶ sessionRepository.getByToken(tokenHash)
  │                                         │          └▶ IF diff>1h: UPDATE session SET updatedAt=NOW()
  │  200 OK / 401 Unauthorized              │
  │◀────────────────────────────────────────│
  │                                         │
  │  ...（空闲超过 90 天 / expiresAt 到点 / 登出 / backchannel logout）...
  │                                         │
  │  WS: on_session_delete(id)              │
  │◀────────────────────────────────────────│ event:SessionDelete → notification
  │                                         │   → websocketRepository.clientSend
  │                                         │
  │  eventManager.emit('SessionDelete')     │
  │  goto(Route.logout())                   │
```

---

## 8. 问题回答

### Q1：Cookie 还在，但 Session 已被删除或过期时，Web 会表现为怎样？

**表现为：用户在刷新/路由切换时被自动踢回登录页，但 Cookie 不会被立刻清除。**

具体路径：

1. 页面 load 时调用 `authenticate(url)` → `authManager.load()`。
2. `#hasAuthCookie()` 返回 `true`（`immich_is_authenticated` 还在，maxAge=400 天）。
3. `refresh()` 发起 `getMyUser()` / `getMyPreferences()`。
4. 这两个请求走到服务端，`AuthGuard.validateSession` 执行 `sessionRepository.getByToken(hashed)`，由于 session 已过期/被删，WHERE 条件过滤掉，返回 `undefined`，直接 `throw new UnauthorizedException('Invalid user token')`。
5. `refresh()` 的 `catch { /* noop */ }` 吞掉错误，`#user` / `#preferences` 保持 `undefined`，于是 `authenticated === false`。
6. `authenticate()` 里 `!authManager.authenticated` 成立 → `redirect(307, Route.login({ continue }))`。

**副作用：**
- 页面视觉上会像「登录过期」，停在 `/auth/login`。
- `immich_is_authenticated` 以及 `immich_access_token`、`immich_auth_type` 三个 Cookie 都**还在浏览器里**；只有真正走到 `/auth/logout`（由后端 `respondWithoutCookie` 显式 `clearCookie`）或服务端再次 Set-Cookie 时才会被清除。
- 如果此时 WebSocket 收到过 `on_session_delete`，`authManager` 会直接 `goto(Route.logout())`，效果类似但路由多一次 `/auth/logout` 的 `+page.svelte` 执行 `authManager.logout()` 完成 Cookie 清除。

### Q2：API Key 和 Shared Link 认证为什么不会依赖这个前端 Cookie？

根本原因：**这两种凭据走的是 `AuthService.validate()` 里与 Cookie 完全独立的分支，且服务端校验完会返回一个不带 `session` 的 `AuthDto`。**

- **API Key**：
  - 由 SDK [packages/sdk/src/index.ts](file:///c:/Users/10244/Desktop/0508-under/immich/packages/sdk/src/index.ts#L27-L28) `defaults.headers['x-api-key'] = apiKey` 写入请求头，或者作为 `?apiKey=` 查询参数；
  - 服务端 `validate()` 解析到 `apiKey` → `validateApiKey(key)` → `apiKeyRepository.getKey(hashSha256(key))` → 命中则返回 `{ user, apiKey }`；
  - 不查 `session` 表，也不读 Cookie。浏览器里就算有 `immich_is_authenticated` 也不影响 API Key 路径。

- **Shared Link**：
  - 通过 URL `?key=` / `?slug=` 或 `x-immich-share-key` / `x-immich-share-slug` 头传入；
  - `validate()` 优先级最高：先匹配 `shareKey` 或 `shareSlug` → `validateSharedLinkKey / validateSharedLinkSlug` → `sharedLinkRepository.getByKey/getBySlug` → 返回 `{ user, sharedLink }`；
  - 不查 `session` 表、不读 Cookie；共享链接页面在前端路由里被标记为 `public: true`（见 [+(user)/+layout.ts](file:///c:/Users/10244/Desktop/0508-under/immich/web/src/routes/(user)/+layout.ts) `authenticate(url, { public: isSharedLinkRoute(route.id) })`），所以即便 `authManager.authenticated` 为 `false` 也不会被重定向到登录页。

- **Cookie 路径（session token）** 与之相反：
  - 依赖 `immich_access_token`（httpOnly，前端看不见）；
  - `immich_is_authenticated` 只是前端的「要不要尝试去调 `/users/me`」的提示位，**不是凭据本身**。

因此 API Key / Shared Link 的校验完全发生在服务端 `validate()` 的另外两条分支，与前端 Cookie 的存在与否、前端 `authManager` 的状态都没有耦合。
