# Etherpad 多作者协作颜色分配与归属机制

## 概述

Etherpad 的多作者协作系统通过一套完整的颜色分配、作者元数据管理和会话生命周期机制，实现了实时协作中作者身份的可视化标识。每个作者被分配一个独特的颜色，用于在编辑器中标记其输入的文本，并在用户列表中展示。

---

## 1. 颜色池 (Color Palette)

### 1.1 颜色池定义

颜色池在 [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L27-L92) 中定义，包含 64 种预定义的柔和背景色：

```typescript
exports.getColorPalette = () => [
  '#ffc7c7', // 浅红
  '#fff1c7', // 浅黄
  '#e3ffc7', // 浅绿
  '#c7ffd5', // 浅青绿
  // ... 共 64 种颜色
];
```

### 1.2 颜色池特性

- **64 种预设颜色**：足够支撑大部分协作场景，颜色经过挑选确保可读性
- **柔和背景色**：均为浅色背景，避免与文本颜色冲突
- **全局共享**：所有 Pad 实例共享同一个颜色池
- **可通过插件扩展**：通过 `clientVars` 钩子可修改发送到客户端的颜色面板

---

## 2. 作者元数据管理 (Author Metadata)

### 2.1 作者创建与颜色分配

作者创建逻辑位于 [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L201-L212) 的 `createAuthor` 函数：

```typescript
exports.createAuthor = async (name: string) => {
  const author = `a.${randomString(16)}`;  // 生成 a. 开头的 16 位随机 ID
  const now = Date.now();
  const authorObj = {
    colorId: Math.floor(Math.random() * (exports.getColorPalette().length)),  // 随机分配颜色索引
    name,
    timestamp: now,
    lastSeen: now,
  };
  await db.set(`globalAuthor:${author}`, authorObj);
  return {authorID: author};
};
```

**关键设计点**：
- `colorId` 存储的是**颜色池索引**（0-63），而非直接存储颜色值
- 颜色分配是**完全随机**的，不考虑当前已在线用户的颜色使用情况
- 作者 ID 格式：`a.` + 16 位随机字符串（如 `a.xk3jd92mcn48fhqp`）

### 2.2 作者数据结构

存储在数据库中的作者对象结构（键：`globalAuthor:${authorID}`）：

```typescript
{
  colorId: number | string,  // 颜色索引 (0-63) 或直接存储 CSS 颜色值
  name: string | null,       // 作者显示名称
  timestamp: number,         // 创建时间戳
  lastSeen: number,          // 最后活动时间戳
  padIDs?: {                 // 参与的 Pad 列表
    [padId: string]: 1
  },
  erased?: boolean,          // 是否已被匿名化 (GDPR)
  erasedAt?: string,         // 匿名化时间
}
```

### 2.3 作者颜色与名称的读写 API

[AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts) 提供以下核心方法：

| 方法 | 用途 |
|------|------|
| `getAuthor(authorId)` | 获取完整作者对象 |
| `getAuthorColorId(authorId)` | 获取作者颜色 ID |
| `setAuthorColorId(authorId, colorId)` | 设置作者颜色（同时更新 `lastSeen`） |
| `getAuthorName(authorId)` | 获取作者名称 |
| `setAuthorName(authorId, name)` | 设置作者名称（同时更新 `lastSeen`） |

---

## 3. 作者身份映射 (Author Identity Mapping)

### 3.1 Token 到作者的映射

用户通过 token 与作者身份绑定，映射关系存储在数据库中：

```typescript
// 从 token 解析 authorId
const getAuthor4Token = async (token: string) => {
  const author = await mapAuthorWithDBKey('token2author', token);
  return author ? author.authorID : author;
};
```

[AuthorManager.ts:148-153](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L148-L153)

### 3.2 映射创建逻辑

[mapAuthorWithDBKey](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L117-L141) 函数处理映射的创建与查找：

```typescript
const mapAuthorWithDBKey = async (mapperkey: string, mapper: string) => {
  const author = await db.get(`${mapperkey}:${mapper}`);

  if (author == null) {
    // 映射不存在，创建新作者
    const author = await exports.createAuthor(null);
    await db.set(`${mapperkey}:${mapper}`, author.authorID);
    return author;
  }

  // 映射存在，更新 lastSeen
  const now = Date.now();
  await db.setSub(`globalAuthor:${author}`, ['timestamp'], now);
  await db.setSub(`globalAuthor:${author}`, ['lastSeen'], now);

  return {authorID: author};
};
```

**数据库键模式**：
- `token2author:${token}` → `authorID` （浏览器 token 映射）
- `mapper2author:${mapper}` → `authorID` （外部系统映射）

### 3.3 getAuthorId 钩子扩展点

[AuthorManager.ts:161-166](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L161-L166) 提供了 `getAuthorId` 钩子，允许插件覆盖作者身份解析逻辑：

```typescript
exports.getAuthorId = async (token: string, user: object) => {
  const context = {dbKey: token, token, user};
  let [authorId] = await hooks.aCallFirst('getAuthorId', context);
  if (!authorId) authorId = await getAuthor4Token(context.dbKey);
  return authorId;
};
```

**设计意图**：
- 支持 SSO/OAuth 场景：插件可根据 `req.session.user` 中的身份信息映射到固定 authorID
- 默认回退：若钩子未返回，则走 `token2author` 标准映射

---

## 4. 多作者颜色归属访问校验链路 (Access Validation Chain)

本节描述 Etherpad 如何通过 **HTTP 会话层（express-session）**、**浏览器 token（HttpOnly Cookie）**、**作者身份（authorID）** 三层递进校验，最终确定颜色归属所对应的 authorID。

### 4.1 Token 的生成与 HTTP Cookie 下发

浏览器首次访问 Pad 前，由 [ensureAuthorTokenCookie.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/utils/ensureAuthorTokenCookie.ts) 确保存在作者 token：

```typescript
export const ensureAuthorTokenCookie = (req, res, settings) => {
  const prefix = settings.cookie?.prefix || '';
  const cookieName = `${prefix}token`;
  const existing = req.cookies?.[cookieName];

  // 已有合法 token → 直接复用（保证跨请求、跨 Pad 颜色一致）
  if (typeof existing === 'string' && padutils.isValidAuthorToken(existing)) {
    return existing;
  }

  // 否则生成新 token: t.<base64url字符串>
  const token = padutils.generateAuthorToken();
  res.cookie(cookieName, token, {
    httpOnly: true,        // 禁止 JS 读取，防止 XSS 窃取
    secure: Boolean(req.secure),
    sameSite: isCrossSiteEmbed(req) ? 'none' : 'lax',
    maxAge: 60 * 24 * 60 * 60 * 1000,  // 60 天 —— 颜色身份可长期保持
    path: '/',
  });
  return token;
};
```

**Token 格式校验** ([pad_utils.ts:393-404](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/pad_utils.ts#L393-L404))：

```typescript
isValidAuthorToken = (t) => {
  if (typeof t !== 'string' || !t.startsWith('t.')) return false;
  const v = t.slice(2);
  return v.length > 0 && base64url.test(v);
};

generateAuthorToken = () => `t.${randomString()}`
```

**关键安全设计**：
- `httpOnly: true` → 前端 JS 无法读取 token，Socket.IO 握手时浏览器自动带 Cookie
- 60 天 maxAge → 同一浏览器长期保留同一颜色身份
- `t.` 前缀 + base64url 校验 → 防止构造恶意键名访问数据库

### 4.2 HTTP 层访问控制 (webaccess)

在 [webaccess.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/hooks/express/webaccess.ts) 中，`checkAccess` 中间件执行 4 步检查：

```
Step 1: preAuthorize 钩子
    ├─ 插件可提前放行或拒绝（/admin 禁止插件放行）
    └─ 任一钩子显式处理 → 跳过后续步骤

Step 2: 首次授权检查 authorize()
    ├─ 管理员 → create
    ├─ 无需认证 → create
    ├─ 未认证 → deny
    └─ 需授权 → authorize 钩子判断等级

Step 3: 认证 (authenticate)
    ├─ 插件 authenticate 钩子
    └─ Fallback: HTTP Basic Auth（settings.users）
        └─ 成功后把 user 对象存入 req.session.user

Step 4: 二次授权检查 authorize()
    └─ 通过则 next()，否则 403
```

**与颜色归属的关系**：
- `webaccess.checkAccess` 仅解决"能不能访问"，不决定 authorID
- 但它把 `req.session.user.padAuthorizations[padId]` 写入，为后续 SecurityManager 提供授权级别
- 认证通过不意味着颜色身份确定——颜色归属由 SecurityManager 在 Socket 层通过 token/sessionCookie 决定

### 4.3 Socket 层安全访问校验 (SecurityManager)

[SecurityManager.checkAccess](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SecurityManager.ts#L60-L150) 是**颜色归属的最终裁决者**，决定了 `handleClientReady` 将用哪个 authorID 读取/写入 `colorId`。

完整校验流程：

```typescript
exports.checkAccess = async (padID, sessionCookie, token, userSettings) => {
  // ── 校验 1: padID 合法性 ──
  if (!padID) return DENY;

  // 只读 ID 转换为真实 padID
  if (readOnlyManager.isReadOnlyId(padID)) {
    canCreate = false;
    padID = await readOnlyManager.getPadId(padID);
    if (padID == null) return DENY;
  }

  // ── 校验 2: HTTP 认证/授权 ──
  if (settings.requireAuthentication) {
    if (userSettings == null) return DENY;
    if (userSettings.readOnly) canCreate = false;
    const level = webaccess.normalizeAuthzLevel(
        userSettings.padAuthorizations?.[padID]);
    if (!level) return DENY;
    if (level !== 'create') canCreate = false;
  }

  // ── 校验 3: 插件 onAccessCheck 钩子 ──
  if (hooks.callAll('onAccessCheck', ...).some(isFalse)) return DENY;

  // ── 校验 4: HTTP API 会话 (sessionCookie) ──
  const sessionAuthorID = await sessionManager.findAuthorID(
      padID.split('$')[0], sessionCookie);

  // requireSession=true: 必须通过 HTTP API 创建了有效会话
  if (settings.requireSession && !sessionAuthorID) return DENY;

  // ── 校验 5: 浏览器 author token ──
  if (!sessionAuthorID && token != null &&
      !padutils.isValidAuthorToken(token)) {
    return DENY;
  }

  // ── 裁决: 决定颜色归属的 authorID ──
  // 优先级: HTTP API 会话 > getAuthorId 插件钩子 > 浏览器 token
  const grant = {
    accessStatus: 'grant',
    authorID: sessionAuthorID ||
              await authorManager.getAuthorId(token, userSettings),
  };

  // ── 校验 6: 群组 Pad 额外规则 ──
  if (padID.includes('$')) {
    if (!padExists && sessionAuthorID == null) return DENY;     // 创建群组 pad 必须有 session
    if (padExists && !pad.getPublicStatus() &&
        sessionAuthorID == null) return DENY;                  // 访问私有群组 pad 必须有 session
  }

  return grant;
};
```

### 4.4 颜色归属决策优先级

从上述代码中提炼出 authorID（颜色归属）的优先级：

| 优先级 | 来源 | 触发条件 | 颜色归属的稳定性 |
|------|------|---------|-------------|
| 1 | **HTTP API sessionCookie** | 调用 `createSession` API，且 `sessionCookie` 中的会话未过期且匹配群组 | ✅ 极高：会话期间固定 authorID，跨设备同账号可共享 |
| 2 | **`getAuthorId` 钩子** | SSO 插件根据 `userSettings` 返回 authorID | ✅ 高：插件可实现 SSO → 固定 authorID 映射 |
| 3 | **浏览器 token（HttpOnly Cookie）** | 无 session 且无钩子，最常见的 Web UI 场景 | ✅ 中高：同一浏览器 60 天内固定，清除 Cookie 则换身份/换颜色 |

### 4.5 访问校验失败对颜色归属的影响

| 失败点 | 原因 | 对颜色归属的影响 |
|-------|------|---------------|
| padID 缺失或无效 | 请求格式错误 | 直接拒绝，无颜色归属 |
| requireAuthentication + 未登录 | webaccess 未通过 | 401 拒绝，用户需先登录再建立 Socket 连接 |
| requireSession + 无有效 session | HTTP API 未创建会话 | 拒绝，必须通过 API 先拿 sessionCookie |
| token 格式非法 | 被篡改/伪造的 token | 拒绝，防止攻击者构造 `token2author:` 任意键访问 |
| 创建/访问私有群组 pad 无 session | 未通过 HTTP API 进群 | 拒绝，群组 pad 颜色归属只能来自 API 会话 |

### 4.6 颜色归属链路完整时序图

```
浏览器                          Node.js (Express + Socket.IO)
  │                                  │
  │ GET /p/abc                       │
  │────────────────────────────────▶ │
  │                                  │ ensureAuthorTokenCookie()
  │                                  │   ├─ 检查 token cookie 是否合法
  │                                  │   └─ 不合法 → Set-Cookie: token=t.xxx (60天)
  │                                  │
  │                                  │ webaccess.checkAccess()
  │                                  │   ├─ preAuthorize 钩子
  │                                  │   ├─ 授权检查
  │                                  │   ├─ 必要时 HTTP Basic 认证 → req.session.user
  │                                  │   └─ 二次授权检查 → next()
  │◀──────────────────────────────── │
  │                                  │
  │ socket.emit(CLIENT_READY)        │
  │   auth:{sessionCookie,token}     │
  │   userInfo:{colorId,name}        │
  │────────────────────────────────▶ │
  │                                  │
  │                                  │ SecurityManager.checkAccess()
  │                                  │   ├─ sessionAuthorID = findAuthorID(group, sessionCookie)
  │                                  │   ├─ requireSession? 无则 DENY
  │                                  │   ├─ token 格式校验
  │                                  │   └─ authorID = sessionAuthorID
  │                                  │                  || hooks.getAuthorId()
  │                                  │                  || getAuthor4Token(token)
  │                                  │
  │                                  │ AuthorManager.getAuthor(authorID)
  │                                  │   └─ 读取 colorId / name
  │                                  │
  │                                  │ (可选) setAuthorColorId / setAuthorName
  │                                  │   └─ 应用客户端带来的初始颜色/昵称
  │                                  │
  │ CLIENT_VARS                      │
  │   colorPalette: [64色]           │
  │   userColor: colorId             │◀ 颜色归属的最终值
  │   userId: authorID               │
  │   historicalAuthorData: {...}    │
  │◀──────────────────────────────── │
  │                                  │
  │ 广播 USER_NEWINFO(authorID,colorId) │
  │────────────────────────────────▶ │──────────▶ 其他客户端更新颜色显示
```

### 4.7 每条消息前的逐次校验与 authorID 变更拒绝机制

**关键点**：`checkAccess` **不是只在 CLIENT_READY 时调用一次，而是在每条 Socket.IO 消息到达时都重新执行**。

[handleMessage](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L377-L614) 的整体执行流程：

```
消息到达 (任何类型: CLIENT_READY / USER_CHANGES / USERINFO_UPDATE / CHAT_MESSAGE 等)
  │
  ├─ 1. 速率限制 (rateLimiter.consume)
  ├─ 2. sessioninfos[socket.id] 存在性校验
  │
  ├─ 3. CLIENT_READY 特殊处理 (首次进入)
  │     ├─ 从 Cookie 头解析 token + sessionID (HttpOnly, GDPR PR3)
  │     ├─ 保存到 thisSession.auth = {sessionID, padID, token}
  │     └─ 解析 readOnly / 真实 padId
  │
  ├─ 4. Object.defineProperty(message, 'padId') 拦截访问 (防止漏洞)
  │
  ├─ 5. 确认 thisSession.auth 已存在 (即必须先通过 CLIENT_READY)
  │     └─ 无 auth → 抛 pre-CLIENT_READY message 错误
  │
  ├─ 6. ★ 每条消息都调用 securityManager.checkAccess() ★
  │     ├─ padID + sessionCookie + token + req.session.user
  │     ├─ findAuthorID() → now < validUntil 实时校验会话是否过期
  │     └─ 返回 {accessStatus, authorID}
  │
  ├─ 7. accessStatus !== 'grant' → 发 {accessStatus} → 抛 'access denied'
  │
  ├─ 8. ★ authorID 变化检测 (颜色归属防篡改核心) ★
  │     if (thisSession.author != null
  │         && thisSession.author !== authorID) {
  │       socket.emit('message', {disconnect: 'rejected'});
  │       throw new Error('Author ID changed mid-session...');
  │     }
  │
  ├─ 9. 首次连接: thisSession.author = authorID  (写入内存快照)
  │
  ├─ 10. handleMessageSecurity 钩子 → 可临时解除只读限制
  ├─ 11. handleMessage 钩子 → 插件可拦截
  │
  └─ 12. switch(type) 分发到具体处理器:
          CLIENT_READY → handleClientReady
          USER_CHANGES → padChannels.enqueue → handleUserChanges
          USERINFO_UPDATE → handleUserInfoUpdate
          CHAT_MESSAGE → handleChatMessage
          ...
```

**核心防线：authorID 变化拒绝（L510-L521）**

```typescript
if (thisSession.author != null && thisSession.author !== authorID) {
  socket.emit('message', {disconnect: 'rejected'});
  throw new Error([
    'Author ID changed mid-session. Bad or missing token or sessionID?',
    `socket:${socket.id}`,
    `IP:${logIp(socket.request.ip)}`,
    `originalAuthorID:${thisSession.author}`,   // 连接初始时的颜色归属
    `newAuthorID:${authorID}`,                  // 本次 checkAccess 算出的归属
    ...(user && user.username) ? [`username:${user.username}`] : [],
    `message:${message}`,
  ].join(' '));
}
```

#### 4.7.1 三种触发 authorID 变化的场景

| 场景 | originalAuthor | newAuthor | 触发 disconnect:rejected？ |
|------|---------------|-----------|------------------------|
| **HTTP API 会话过期，token 对应另一位作者** | `a.sessionAuthor`（来自 createSession） | `a.tokenAuthor`（浏览器 token 对应） | ✅ 是，立即断开 |
| **会话被管理员 API 主动 deleteSession 删除** | `a.sessionAuthor` | `a.tokenAuthor`（降级到 token） | ✅ 是，下一条消息就断开 |
| **会话过期，但 token 恰对应同一作者** | `a.abc123` | `a.abc123` | ❌ 不变，允许继续 |
| **requireSession=true，会话过期** | `a.sessionAuthor` | DENY（无降级） | 先走 access denied，不会走到比较 |

**对颜色身份的直接影响**：
- 一旦触发 `disconnect: 'rejected'`，客户端 Socket 连接被服务端强制关闭
- 客户端必须重新走完整的 Pad 页面加载 → Socket 重连 → 新一轮 CLIENT_READY
- 若此时会话已过期且 requireSession=false → 新连接获得 `a.tokenAuthor` 的颜色身份，**用户可能看到自己的颜色突然变了**
- 若 requireSession=true → 无法重连，提示需要重新登录/授权

---

## 5. 会话生命周期 (Session Lifecycle)

### 5.1 会话创建与反向索引建立

会话管理位于 [SessionManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L105-L159)，通过 API 创建的会话将作者与群组绑定：

```typescript
exports.createSession = async (groupID: string, authorID: string, validUntil: number) => {
  // validUntil 必须是未来的时间戳（秒级）
  if (validUntil < Math.floor(Date.now() / 1000)) {
    throw new CustomError('validUntil is in the past', 'apierror');
  }

  const sessionID = `s.${randomString(16)}`;

  // 1. 写入会话主记录
  await db.set(`session:${sessionID}`, {groupID, authorID, validUntil});

  // 2. 建立双向反向索引
  await Promise.all([
    // group → sessions：用于遍历组内所有会话、组删除时清理
    db.setSub(`group2sessions:${groupID}`, ['sessionIDs', sessionID], 1),
    // author → sessions：用于遍历作者所有会话、作者清理时回收
    db.setSub(`author2sessions:${authorID}`, ['sessionIDs', sessionID], 1),
  ]);

  return {sessionID};
};
```

**数据库记录全景**：
| 数据库键 | 结构 | 用途 |
|---------|------|------|
| `session:${sid}` | `{groupID, authorID, validUntil}` | 会话主数据，过期判断依据 |
| `group2sessions:${gid}` | `{sessionIDs: {[sid]: 1}}` | 群组 → 会话 反向索引 |
| `author2sessions:${aid}` | `{sessionIDs: {[sid]: 1}}` | 作者 → 会话 反向索引 |

### 5.2 会话查找与过期验证

[findAuthorID](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L39-L85) 用于从 cookie 中解析有效会话对应的作者：

```typescript
exports.findAuthorID = async (groupID: string, sessionCookie: string) => {
  const sessionIDs = sessionCookie.replace(/^"|"$/g, '').split(',');
  const now = Math.floor(Date.now() / 1000);

  // 查找未过期且匹配群组的会话
  const isMatch = (si) => si != null && si.groupID === groupID && now < si.validUntil;
  const sessionInfo = await firstSatisfies(sessionInfoPromises, isMatch);

  return sessionInfo?.authorID;
};
```

**过期判断核心**：`now < validUntil`（Unix 秒级时间戳对比）。

**注意**：`findAuthorID` 在会话过期时**静默忽略**（不报错，返回 undefined），交给 SecurityManager 决定是否降级使用 token。

### 5.3 会话过期对颜色归属的影响

| 场景 | 会话状态 | 颜色归属 authorID | 用户可见行为 |
|-----|---------|----------------|-----------|
| **requireSession=false**（默认） | 会话过期 | ✅ 自动降级使用浏览器 token 对应的 authorID | 颜色可能切换（若 session.author ≠ token.author），提示重新登录 |
| **requireSession=true**（严格） | 会话过期 | ❌ SecurityManager.checkAccess 返回 DENY | Socket 连接被拒绝，需重新调用 createSession API |
| 群组 pad 私有 + 无有效会话 | 会话过期 | ❌ DENY | 无法访问该 pad |
| 多个 sessionID 中只有一个有效 | 部分过期 | ✅ 使用第一个未过期且匹配群组的会话 | 颜色归属不变（只要会话的 authorID 相同） |

### 5.4 会话删除与反向索引回收

[deleteSession](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L184-L206) 是会话清理的唯一入口，必须**先回收反向索引再删除主记录**以保持一致性：

```typescript
exports.deleteSession = async (sessionID) => {
  // 读取会话信息获取 groupID/authorID（必须先读）
  const session = await db.get(`session:${sessionID}`);
  if (session == null) throw new CustomError('sessionID does not exist', 'apierror');

  const {groupID, authorID} = session;

  // 第一步：从双向反向索引中移除 sessionID
  //   使用 setSub(undefined) 而非 db.remove
  //   因为 UeberDB 会在 JSON.stringify 时忽略 undefined 属性
  await Promise.all([
    db.setSub(`group2sessions:${groupID}`,
              ['sessionIDs', sessionID], undefined),
    db.setSub(`author2sessions:${authorID}`,
              ['sessionIDs', sessionID], undefined),
  ]);

  // 第二步：删除会话主记录（最后操作，保证一致性）
  await db.remove(`session:${sessionID}`);
};
```

**一致性保障要点**：
- 写入顺序：先建索引后建主记录 → 删除顺序：先删索引后删主记录
- UeberDB 的 `setSub(path, undefined)` 是原子操作：读 → 修改属性 → 写回
- 索引对象中的值设为 `1`（仅表示存在），删除时设 `undefined`

### 5.5 反向索引回收的级联场景

#### 场景 A：删除群组时级联清理所有会话

[GroupManager.deleteGroup](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/GroupManager.ts#L46-L81)：

```typescript
exports.deleteGroup = async (groupID) => {
  const group = await db.get(`group:${groupID}`);

  // 1. 先删除该群组的所有 Pad
  await Promise.all(Object.keys(group.pads).map(padId => ...));

  // 2. 级联删除所有会话
  //    注意 Issue #5798：undefined 的 sessionID 键仍存在于对象中，
  //    必须 filter 掉才不会 deleteSession 抛错
  const {sessionIDs = {}} = await db.get(`group2sessions:${groupID}`) || {};
  await Promise.all(
      Object.keys(sessionIDs)
            .filter(id => sessionIDs[id])  // 过滤已被 setSub(undefined) 标记删除的
            .map(sessionId => sessionManager.deleteSession(sessionId))
  );

  // 3. 清理群组记录与映射
  await Promise.all([
    db.remove(`group2sessions:${groupID}`),
    db.setSub('groups', [groupID], undefined),
    ...Object.keys(group.mappings || {})
            .map(m => db.remove(`mapper2group:${m}`)),
  ]);

  await db.remove(`group:${groupID}`);
};
```

**对颜色归属的影响**：群组删除意味着其中所有会话的 authorID 不再优先于 token，Web UI 用户会回退到浏览器 token 的颜色身份。

#### 场景 B：作者被删除（GDPR 匿名化）时的会话回收

目前 Etherpad 核心并未提供 `deleteAuthor` API，但作者匿名化（GDPR 功能）会保留 authorID 并将颜色/名称清空：
- 已建立的会话**继续有效**（`session:${sid}` 中引用的 authorID 仍存在）
- 颜色会变为默认（若 name/colorId 被清空），用户再次进入 pad 时 `handleClientReady` 读到空 colorId → 客户端重新分配
- `author2sessions:${aid}` 索引仅用于 `listSessionsOfAuthor` API 查询，不会自动清理

#### 场景 C：listSessionsWithDBKey 中的僵尸会话

反向索引中残留已删除的会话（setSub undefined 留下的键）时，[listSessionsWithDBKey](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L246-L263) 会尝试逐个读取：

```typescript
const listSessionsWithDBKey = async (dbkey) => {
  const sessionObject = await db.get(dbkey);
  const sessions = sessionObject ? sessionObject.sessionIDs : null;

  for (const sessionID of Object.keys(sessions || {})) {
    try {
      // 若会话主记录已删除，getSessionInfo 抛错 → 静默 catch
      sessions[sessionID] = await exports.getSessionInfo(sessionID);
    } catch (err: any) {
      if (err.message === 'sessionID does not exist') {
        // 残留索引：catch 后此键会保留原来的 1 值（非会话信息）
        // 返回的对象中会混合出现 {sid1: {sessionInfo}, sid2: 1}
        console.debug(`no session exists with ID ${sessionID}`);
      } else {
        throw err;
      }
    }
  }
  return sessions;
};
```

**潜在问题（僵尸索引）**：
- 若主记录已删但反向索引键还在，`listSessionsOfGroup` 返回值中会混入 `1` 值
- Issue #5798 的修复：在 deleteGroup 中 `.filter(id => sessionIDs[id])` 规避
- 但日常运行中的孤立索引暂无后台 GC 机制

### 5.6 会话过期的被动清理与主动清理

#### 被动清理（当前实现）

**没有定时删除过期 `session:` 记录的后台任务！**

过期会话的处理完全是**查询时判断**：
- `findAuthorID(group, sessionCookie)`：`now < validUntil` 时才使用
- `getSessionInfo(sessionID)`：**不会**主动校验 `validUntil`，API 可查询到已过期会话
- `listSessionsOfGroup/Author`：返回列表包含过期会话，由 API 调用方自行判断

#### 主动清理策略（需要额外实现）

当前版本需管理员通过脚本调用 `deleteSession` 清理，可基于：
1. `listSessionsOfGroup(groupID)` 遍历 → 检查 `validUntil` → 过期则 `deleteSession`
2. 按 `author2sessions:*` 范围扫描数据库（视底层存储能力）

#### 会话过期长期积压对颜色归属的风险

| 风险 | 说明 |
|-----|------|
| **authorID 永久绑定** | 过期 session 的 authorID 仍存在于 `session:` 记录中，GDPR 场景下无法证明已删除 |
| **性能影响** | `findAuthorID` 每连接要遍历 cookie 中的所有 sessionID 并行查库，过期记录越多无效查询越多 |
| **反向索引膨胀** | `group2sessions:` 对象持续增长，`setSub` 的读改写成本上升 |
| **颜色归属混淆** | 若管理员误将 `validUntil` 设得极长（999999999999），即使用户退出登录也会一直使用旧 authorID |

### 5.7 连接级会话信息 (sessioninfos) 的精确生命周期

`sessioninfos` 是 [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L102-L103) 中维护的内存对象，记录每个 socket 连接的颜色归属状态。它的每个属性在**不同的代码位置、不同的时机**被写入，理解这个时序是搞清会话过期与颜色归属关系的关键。

#### 5.7.1 sessioninfos 各属性的写入时序

```
时间线                    代码位置                              写入的属性
─────────────────────────────────────────────────────────────────────────
T1: Socket.IO 连接建立   handleConnect() [L221-L226]         sessioninfos[socket.id] = {}
                                                                   → {} 空对象，author = undefined

T2: CLIENT_READY 消息    handleMessage() [L398-L486]         thisSession.auth = {sessionID, padID, token}
  (仅此消息类型)          ├── Cookie 解析                       thisSession.embed = ...
                          ├── padId 解析                       thisSession.padId = padIds.padId
                          └── readOnly 判断                    thisSession.readOnlyPadId = ...
                                                               thisSession.readonly = ...

T3: 每条消息             handleMessage() [L503-L522]         ★ thisSession.author = authorID ★
  (含 CLIENT_READY)       ├── checkAccess() 返回 authorID         首次: undefined → 'a.xxxxx'
                          ├── L510 比较 authorID 是否变化          后续: 'a.xxxxx' → 'a.xxxxx' (幂等覆写)
                          └── L522 写入                           若变化: disconnect:'rejected' → 不走到这里

T4: CLIENT_READY 分支    handleClientReady() [L1118-L1458]   sessionInfo.rev = message.client_rev
  (正常连接)              ├── L1121 assert(sessionInfo.author)   sessionInfo.time = ...
  或 reconnect 分支       └── [L1208-L1269 reconnect]           (author 不在 handleClientReady 中写入，
                                                                而是在 handleMessage L522 已经写入)

T5: USER_CHANGES         handleUserChanges() [L819-L1006]    thisSession.rev = newRev
  (每次打字提交)          └── L995                              thisSession.time = ...
                                                               (author 来自 thisSession.author 只读引用)

T6: Socket 断开          handleDisconnect() [L246-L287]      delete sessioninfos[socket.id]
                          └── L249                              → 立即删除，先于 USER_LEAVE 广播
```

#### 5.7.2 关键代码：authorID 写入与校验的精确位置

[handleMessage L503-L522](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L503-L522)：

```typescript
// ★ 从 HTTP 请求中提取已认证的用户信息（express-session）
const {session: {user} = {}} = socket.client.request as SocketClientRequest;

// ★ 每条消息都重新调用 checkAccess
//    参数来源：thisSession.auth 在 CLIENT_READY 时写入，后续消息复用
const {accessStatus, authorID} =
    await securityManager.checkAccess(auth.padID, auth.sessionID, auth.token, user);

if (accessStatus !== 'grant') {
  socket.emit('message', {accessStatus});
  throw new Error('access denied');
}

// ★ authorID 变化检测
//    thisSession.author 在首次消息时为 undefined (nullish)，不会进入此 if
//    从第二条消息起，thisSession.author 已有值，任何变化都会触发 reject
if (thisSession.author != null && thisSession.author !== authorID) {
  socket.emit('message', {disconnect: 'rejected'});
  throw new Error([
    'Author ID changed mid-session. Bad or missing token or sessionID?',
    `socket:${socket.id}`,
    `IP:${logIp(socket.request.ip)}`,
    `originalAuthorID:${thisSession.author}`,   // T3 首次写入的值
    `newAuthorID:${authorID}`,                  // 本次 checkAccess 算出的值
    ...(user && user.username) ? [`username:${user.username}`] : [],
    `message:${message}`,
  ].join(' '));
}

// ★ 幂等覆写：每条消息都写一次，值不变则为幂等操作
thisSession.author = authorID;
```

#### 5.7.3 auth 对象的生命周期：Cookie 快照而非实时读取

[L462-L469](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L462-L469) 的注释明确说明了 auth 的来源：

> "Remember this information since we won't have the cookie in further socket.io messages. This information will be used to check if the sessionId of this connection is still valid since it could have been deleted by the API."

**关键含义**：
- `auth.token` 和 `auth.sessionID` 是从 CLIENT_READY 消息到达时的 **HTTP Cookie 头一次性快照**
- Socket.IO 后续消息**不再携带 Cookie**，所以 checkAccess 每次使用的 `auth.token` / `auth.sessionID` 都是同一个快照值
- 这意味着：
  - 如果用户在另一个标签页清除了浏览器 Cookie → 当前连接的 auth 不受影响 → 仍用旧 token 继续校验
  - 如果管理员在 API 端 `deleteSession` → auth.sessionID 对应的 session 记录从数据库删除 → 下一条消息 checkAccess 中 `findAuthorID()` 会返回 undefined → authorID 降级到 token → 触发 rejected

#### 5.7.4 handleClientReady 中 authorID 的角色：只读使用而非写入

[handleClientReady L1118-L1121](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1118-L1121)：

```typescript
const sessionInfo = sessioninfos[socket.id];
if (sessionInfo == null) throw new Error('client disconnected');
assert(sessionInfo.author);  // ★ 证明 author 在 handleMessage 中已写入
```

**assert(sessionInfo.author) 的重要性**：
- `handleClientReady` 在 `handleMessage` 的 switch 分支 (L569) 中被调用
- 调用之前 L522 已经执行了 `thisSession.author = authorID`
- 所以 `handleClientReady` 中的 `sessionInfo.author` 一定有值（否则 assert 失败）
- `handleClientReady` 只**读取** `sessionInfo.author`，用于：
  - 写入颜色/名称到数据库 (`setAuthorColorId`, `setAuthorName`)
  - 收集历史作者信息 (`pad.getAllAuthors()`)
  - 组装 clientVars (`userId: sessionInfo.author`)
  - 同作者踢出检测 (`sinfo.author === sessionInfo.author`)
  - 判断是否为 pad 创建者 (`sessionInfo.author === pad.getRevisionAuthor(0)`)

---

## 6. 客户端初始化流程 (Client Initialization)

### 6.1 CLIENT_READY 消息处理

用户连接后发送 `CLIENT_READY` 消息，服务器在 [handleClientReady](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1118-L1458) 中进行处理：

```typescript
const handleClientReady = async (socket, message) => {
  // 1. 解析 token/sessionID，获取 authorID
  const {accessStatus, authorID} =
      await securityManager.checkAccess(auth.padID, auth.sessionID, auth.token, user);

  // 2. 应用客户端可能自带的颜色/名称
  let {colorId: authorColorId, name: authorName} = message.userInfo || {};
  await Promise.all([
    authorName && authorManager.setAuthorName(sessionInfo.author, authorName),
    authorColorId && authorManager.setAuthorColorId(sessionInfo.author, authorColorId),
  ]);

  // 3. 重新从数据库读取（确保一致性）
  ({colorId: authorColorId, name: authorName} =
      await authorManager.getAuthor(sessionInfo.author));

  // 4. 收集 Pad 上所有历史作者信息
  const authors = pad.getAllAuthors();
  const historicalAuthorData = {};
  await Promise.all(authors.map(async (authorId) => {
    const author = await authorManager.getAuthor(authorId);
    historicalAuthorData[authorId] = {name: author.name, colorId: author.colorId};
  }));

  // 5. 组装 clientVars
  const clientVars = {
    collab_client_vars: {
      initialAttributedText: atext,
      historicalAuthorData,        // 所有历史作者的颜色/名称
      apool,
      rev: headRev,
    },
    colorPalette: authorManager.getColorPalette(),  // 发送完整颜色池
    userColor: authorColorId,                        // 当前用户的颜色
    userId: sessionInfo.author,                      // 当前用户的 authorID
    userName: authorName,                            // 当前用户名称
    // ... 其他字段
  };

  // 6. 发送 clientVars 给客户端
  socket.emit('message', {type: 'CLIENT_VARS', data: clientVars});

  // 7. 广播给其他在线用户
  socket.broadcast.to(sessionInfo.padId).emit('message', {
    type: 'COLLABROOM',
    data: {
      type: 'USER_NEWINFO',
      userInfo: {
        colorId: authorColorId,
        name: authorName,
        userId: sessionInfo.author,
      },
    },
  });
};
```

### 6.2 客户端协作客户端初始化

客户端收到 `CLIENT_VARS` 后，在 [collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts#L39-L529) 中初始化协作客户端：

```typescript
const getCollabClient = (ace2editor, serverVars, initialUserInfo, options, _pad) => {
  const userId = initialUserInfo.userId;
  const userSet = {};  // userId -> userInfo
  userSet[userId] = initialUserInfo;

  // 初始化时通知 ACE 编辑器历史作者信息
  const tellAceAboutHistoricalAuthors = (hadata) => {
    for (const [author, data] of Object.entries(hadata)) {
      if (!userSet[author]) {
        tellAceAuthorInfo(author, data.colorId, true);  // true = 淡出显示
      }
    }
  };

  // 通知 ACE 编辑器单个作者的颜色信息
  const tellAceAuthorInfo = (userId, colorId, inactive) => {
    if (typeof colorId === 'number') {
      colorId = clientVars.colorPalette[colorId];  // 索引转颜色值
    }

    if (inactive) {
      editor.setAuthorInfo(userId, {bgcolor: colorId, fade: 0.5});
    } else {
      editor.setAuthorInfo(userId, {bgcolor: colorId});
    }
  };

  // 初始化时调用
  tellAceAboutHistoricalAuthors(serverVars.historicalAuthorData);
  tellAceActiveAuthorInfo(initialUserInfo);
};
```

---

## 7. 实时颜色同步 (Real-time Color Sync)

### 7.1 用户信息更新流程

当用户修改颜色或名称时，触发以下流程：

**1. 客户端发起更新** ([pad_userlist.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/pad_userlist.ts#L642-L649))：

```typescript
const closeColorPicker = (accept) => {
  if (accept) {
    myUserInfo.colorId = newColor;
    pad.notifyChangeColor(newColor);  // 发送 USERINFO_UPDATE
    paduserlist.renderMyUserInfo();
  }
};
```

**2. collab_client 转发** ([collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts#L326-L336))：

```typescript
const updateUserInfo = (userInfo) => {
  userInfo.userId = userId;
  userSet[userId] = userInfo;
  tellAceActiveAuthorInfo(userInfo);  // 本地立即更新
  sendMessage({
    type: 'USERINFO_UPDATE',
    userInfo,
  });
};
```

**3. 服务端处理** ([PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L771-L803))：

```typescript
const handleUserInfoUpdate = async (socket, {data: {userInfo: {name, colorId}}}) => {
  const author = session.author;

  // 验证颜色格式
  if (!/(^#[0-9A-F]{6}$)|(^#[0-9A-F]{3}$)/i.test(colorId)) {
    throw new Error(`malformed color: ${colorId}`);
  }

  // 更新数据库
  await Promise.all([
    authorManager.setAuthorColorId(author, colorId),
    authorManager.setAuthorName(author, name),
  ]);

  // 广播给其他客户端
  socket.broadcast.to(padId).emit('message', {
    type: 'COLLABROOM',
    data: {
      type: 'USER_NEWINFO',
      userInfo: {userId: author, name, colorId},
    },
  });
};
```

**4. 其他客户端接收** ([collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts#L268-L278))：

```typescript
} else if (msg.type === 'USER_NEWINFO') {
  const userInfo = msg.userInfo;
  const id = userInfo.userId;
  if (userSet[id]) {
    userSet[id] = userInfo;
    callbacks.onUpdateUserInfo(userInfo);
  } else {
    userSet[id] = userInfo;
    callbacks.onUserJoin(userInfo);
  }
  tellAceActiveAuthorInfo(userInfo);
}
```

### 7.2 用户离开处理

当用户断开连接时，在 [handleDisconnect](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L246-L287) 中处理：

```typescript
exports.handleDisconnect = async (socket) => {
  const session = sessioninfos[socket.id];

  // 仅当该作者的最后一个 socket 离开时才广播 USER_LEAVE
  const isLastSocketForAuthor = !_getRoomSockets(session.padId).some(
      (s) => sessioninfos[s.id]?.author === session.author);

  if (isLastSocketForAuthor) {
    socket.broadcast.to(session.padId).emit('message', {
      type: 'COLLABROOM',
      data: {
        type: 'USER_LEAVE',
        userInfo: {
          colorId: await authorManager.getAuthorColorId(session.author),
          userId: session.author,
        },
      },
    });
  }
};
```

客户端收到 `USER_LEAVE` 后，将该作者标记为淡出状态：

```typescript
} else if (msg.type === 'USER_LEAVE') {
  const userInfo = msg.userInfo;
  const id = userInfo.userId;
  if (userSet[id]) {
    delete userSet[userInfo.userId];
    fadeAceAuthorInfo(userInfo);  // 设置 fade: 0.5
    callbacks.onUserLeave(userInfo);
  }
}
```

---

## 8. 文本作者属性标记 (Author Attribution on Text)

### 8.1 插入时的作者标记

在 [stampAuthorOnInserts.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/stampAuthorOnInserts.ts) 中，确保每个插入操作都带有作者属性：

```typescript
export const stampAuthorOnInserts = (changeset, apoolJsonable, authorId) => {
  const pool = (new AttributePool()).fromJsonable(apoolJsonable);
  const unpacked = unpack(changeset);
  const assem = new SmartOpAssembler();
  let modified = false;

  for (const op of deserializeOps(unpacked.ops)) {
    if (op.opcode === '+') {  // 仅处理插入操作
      const attribs = AttributeMap.fromString(op.attribs, pool);
      if (!attribs.get('author')) {
        attribs.set('author', authorId);  // 补充缺失的 author 属性
        op.attribs = attribs.toString();
        modified = true;
      }
    }
    assem.append(op);
  }

  if (!modified) return {changeset, apool: apoolJsonable};
  // 返回重写后的 changeset
};
```

### 8.2 协作客户端调用时机

在 [collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts#L59-L67) 的 `prepareUserChangeset` 中调用：

```typescript
const prepareUserChangeset = () => {
  const data = editor.prepareUserChangeset();
  if (data.changeset) {
    // 确保所有插入操作都带有作者标记
    const stamped = stampAuthorOnInserts(data.changeset, data.apool, userId);
    data.changeset = stamped.changeset;
    data.apool = stamped.apool;
  }
  return data;
};
```

### 8.3 服务端验证

在 [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L859-L920) 中，服务器会验证每个变更的作者属性：

```typescript
for (const op of deserializeOps(unpack(changeset).ops)) {
  const opAuthorId = AttributeMap.fromString(op.attribs, wireApool).get('author');

  // 防止冒充其他作者
  if (opAuthorId && opAuthorId !== thisSession.author) {
    if (op.opcode === '=') {
      // 允许在已有文本上恢复作者属性（用于撤销清除作者色）
      // 但必须是该 pad 已存在的作者
      const knownAuthor = pad.pool.putAttrib(['author', opAuthorId], true) !== -1;
      if (!knownAuthor) throw new Error('unknown author');
    } else {
      // 插入操作必须是当前用户
      throw new Error(`Author ${thisSession.author} tried to submit changes as author ${opAuthorId}`);
    }
  }

  // 插入操作必须有 author 属性
  if (op.opcode === '+' && !opAuthorId) {
    throw new Error('insert without an author attribute');
  }
}
```

---

## 9. 颜色工具函数 (Color Utilities)

[colorutils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/colorutils.ts) 提供完整的颜色处理工具集：

### 9.1 颜色格式转换

```typescript
colorutils.css2triple(cssColor)     // "#fff" → [1.0, 1.0, 1.0]
colorutils.css2sixhex(cssColor)     // "#fff" → "ffffff"
colorutils.triple2css(triple)       // [1.0, 1.0, 1.0] → "#ffffff"
```

### 9.2 WCAG 2.1 可访问性支持

为确保文本与背景色有足够对比度，实现了 WCAG 2.1 标准的对比度计算：

```typescript
// 计算相对亮度
colorutils.relativeLuminance = (c) => {
  const toLinear = (v) => v <= 0.03928 ? v / 12.92 : Math.pow((v + 0.055) / 1.055, 2.4);
  return 0.2126 * toLinear(c[0]) + 0.7152 * toLinear(c[1]) + 0.0722 * toLinear(c[2]);
};

// 计算对比度比 (1-21，4.5 = AA，7.0 = AAA)
colorutils.contrastRatio = (c1, c2) => {
  const l1 = colorutils.relativeLuminance(c1);
  const l2 = colorutils.relativeLuminance(c2);
  return (Math.max(l1, l2) + 0.05) / (Math.min(l1, l2) + 0.05);
};
```

### 9.3 智能文本颜色选择

根据背景色自动选择深色或浅色文本以确保可读性：

```typescript
colorutils.textColorFromBackgroundColor = (bgcolor, skinName) => {
  const refs = skinTextColors(skinName);
  const triple = colorutils.css2triple(bgcolor);
  const ratioDark = colorutils.contrastRatio(triple, colorutils.css2triple(refs.darkRef));
  const ratioLight = colorutils.contrastRatio(triple, colorutils.css2triple(refs.lightRef));
  return ratioDark >= ratioLight ? refs.darkOut : refs.lightOut;
};
```

### 9.4 背景色可读性保障

如果背景色与文本对比度不足，自动调整背景色：

```typescript
colorutils.ensureReadableBackground = (cssColor, skinName, minContrast = 4.5) => {
  if (!colorutils.isCssHex(cssColor)) return cssColor;

  // 检查对比度是否达标
  const ratioDark = colorutils.contrastRatio(triple, dark);
  const ratioLight = colorutils.contrastRatio(triple, light);
  if (Math.max(ratioDark, ratioLight) >= minContrast) return cssColor;

  // 逐步向白色或黑色混合，直到对比度达标
  const blendTarget = ratioDark >= ratioLight ? [1, 1, 1] : [0, 0, 0];
  for (let i = 1; i <= 20; i++) {
    const blended = colorutils.blend(triple, blendTarget, i * 0.05);
    if (colorutils.contrastRatio(blended, textRef) >= minContrast) {
      return colorutils.triple2css(blended);
    }
  }
  return colorutils.triple2css(blendTarget);
};
```

---

## 10. 用户列表 UI 展示 (User List UI)

[pad_userlist.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/pad_userlist.ts) 负责用户列表的渲染：

### 10.1 颜色索引转颜色值

```typescript
// 设置当前用户信息时转换颜色
setMyUserInfo: (info) => {
  if (typeof info.colorId === 'number') {
    info.colorId = clientVars.colorPalette[info.colorId];
  }
  myUserInfo = $.extend({}, info);
  self.renderMyUserInfo();
};
```

### 10.2 用户行渲染

```typescript
const createUserRowTds = (height, data) => {
  return $()
      .add($('<td>')
          .addClass('usertdswatch')
          .append($('<div>')
              .addClass('swatch')
              .css('background', padutils.escapeHtml(data.color))  // 颜色块
              .html('&nbsp;')))
      .add($('<td>')
          .addClass('usertdname')
          .append(name))  // 用户名
      .add($('<td>')
          .addClass('activity')
          .text(data.activity));  // 活动状态
};
```

### 10.3 颜色选择器

```typescript
const showColorPicker = () => {
  const palette = pad.getColorPalette();
  const colorsList = $('#colorpickerswatches');

  // 渲染颜色选择面板
  for (let i = 0; i < palette.length; i++) {
    const li = $('<li>', {style: `background: ${palette[i]};`});
    li.appendTo(colorsList);

    li.on('click', (event) => {
      const newColorId = getColorPickerSwatchIndex($(event.target));
      pad.notifyChangeColor(newColorId);  // 点击后立即更新
    });
  }
};
```

---

## 11. 完整数据流图

### 11.1 新用户加入流程

```
浏览器连接
    ↓
CLIENT_READY (token, padId)
    ↓
[Server] securityManager.checkAccess()
    ↓
[Server] AuthorManager.getAuthor4Token()
    ├─ 存在 → 更新 lastSeen
    └─ 不存在 → createAuthor() → 随机分配 colorId
    ↓
[Server] 收集 pad.getAllAuthors() → 批量查询所有作者的 colorId/name
    ↓
[Server] 组装 clientVars:
    - colorPalette: [64种颜色]
    - userColor: 当前用户 colorId
    - userId: 当前用户 authorID
    - collab_client_vars.historicalAuthorData: {authorId: {name, colorId}}
    ↓
CLIENT_VARS → 浏览器
    ↓
[Client] collab_client 初始化:
    - tellAceAboutHistoricalAuthors() → 所有历史作者设置为 fade:0.5
    - tellAceActiveAuthorInfo() → 当前用户正常显示
    ↓
[Server] 广播 USER_NEWINFO 给其他客户端
    ↓
[Other Clients] userJoin 回调 → 更新用户列表、设置 ACE 作者颜色
```

### 11.2 颜色修改流程

```
用户在 UI 选择新颜色
    ↓
[Client] pad.notifyChangeColor(newColor)
    ↓
[Client] collab_client.updateUserInfo():
    - 本地更新 userSet[userId]
    - tellAceActiveAuthorInfo() → 编辑器立即显示新颜色
    - 发送 USERINFO_UPDATE 到服务器
    ↓
[Server] handleUserInfoUpdate():
    - 验证颜色格式
    - AuthorManager.setAuthorColorId() → 持久化
    - AuthorManager.setAuthorName() → 持久化
    - 广播 USER_NEWINFO 给其他客户端
    ↓
[Other Clients] onUpdateUserInfo 回调
    - 更新 userSet[id]
    - tellAceActiveAuthorInfo() → 更新编辑器显示
    - paduserlist.userJoinOrUpdate() → 更新用户列表
```

---

## 12. 关键设计权衡

### 12.1 颜色分配策略

| 策略 | 优点 | 缺点 |
|------|------|------|
| **完全随机分配** | 实现简单，无状态 | 可能出现颜色冲突（多个用户分到相近颜色） |
| 基于用户 ID 哈希 | 同一用户始终获得相同颜色 | 仍可能冲突 |
| 动态分配可用颜色 | 避免冲突 | 实现复杂，需要维护使用状态 |

**当前选择**：完全随机分配。对于 64 色的池，在常用协作场景（<10 人）中冲突概率较低，且用户可手动修改颜色。

### 12.2 颜色 ID 的两种形式

- **存储形式**：`colorId` 可以是数字索引（0-63）或直接存储 CSS 颜色字符串
- **原因**：允许用户选择自定义颜色（不在预设池中），通过颜色选择器的自由取色功能
- **处理**：在使用时检查 `typeof colorId === 'number'`，若是则从 colorPalette 查找

### 12.3 历史作者的淡出处理

- 历史作者（不在当前会话中）在编辑器中显示为 `fade: 0.5` 半透明
- 但在用户列表中，离开的用户会在 8 秒后完全移除
- 设计考量：编辑器中需要保留历史上下文，用户列表只需显示当前活跃用户

---

## 13. 代码溯源索引

| 功能模块 | 核心文件 | 关键函数/位置 |
|---------|---------|-------------|
| 颜色池定义 | [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts) | `getColorPalette()` [L27-L92](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L27-L92) |
| 作者创建 | [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts) | `createAuthor()` [L201-L212](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L201-L212) |
| 作者映射 | [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts) | `mapAuthorWithDBKey()` [L117-L141](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L117-L141) |
| 作者ID优先级决策 | [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts) | `getAuthorId()` [L161-L166](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L161-L166) |
| 会话创建与反向索引 | [SessionManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts) | `createSession()` [L105-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L105-L159) |
| 会话查找与过期验证 | [SessionManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts) | `findAuthorID()` [L39-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L39-L85) |
| 会话删除与索引回收 | [SessionManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts) | `deleteSession()` [L184-L206](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L184-L206) |
| 反向索引列表（含僵尸处理） | [SessionManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts) | `listSessionsWithDBKey()` [L246-L263](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L246-L263) |
| 群组删除级联会话清理 | [GroupManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/GroupManager.ts) | `deleteGroup()` [L46-L81](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/GroupManager.ts#L46-L81) |
| HTTP层访问认证/授权 | [webaccess.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/hooks/express/webaccess.ts) | 四步检查流程：preAuthorize→authorize→authenticate→authorize |
| Socket层安全访问校验 | [SecurityManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SecurityManager.ts) | `checkAccess()` [L60-L150](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SecurityManager.ts#L60-L150) |
| 浏览器Token Cookie下发 | [ensureAuthorTokenCookie.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/utils/ensureAuthorTokenCookie.ts) | `ensureAuthorTokenCookie()` 60天 maxAge / SameSite 自适应 |
| 浏览器Token格式校验 | [pad_utils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/pad_utils.ts) | `isValidAuthorToken(t)` `generateAuthorToken()` |
| 连接级会话信息快照 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `sessioninfos[socket.id]` [L102-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L102-L103) |
| 客户端初始化 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleClientReady()` [L1118-L1458](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1118-L1458) |
| 协作客户端 | [collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts) | `getCollabClient()` [L39-L529](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts#L39-L529) |
| 作者标记 | [stampAuthorOnInserts.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/stampAuthorOnInserts.ts) | `stampAuthorOnInserts()` [L31-L57](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/stampAuthorOnInserts.ts#L31-L57) |
| 颜色工具 | [colorutils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/colorutils.ts) | `ensureReadableBackground()` [L171-L192](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/colorutils.ts#L171-L192) |
| 用户列表 | [pad_userlist.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/pad_userlist.ts) | `userJoinOrUpdate()` [L490-L548](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/pad_userlist.ts#L490-L548) |
| 用户信息更新 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleUserInfoUpdate()` [L771-L803](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L771-L803) |
| 断开连接 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleDisconnect()` [L246-L287](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L246-L287) |
| 变更验证 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleUserChanges()` 验证 [L859-L920](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L859-L920) |
| 消息处理全流程 + authorID变化拒绝 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleMessage()` [L377-L614](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L377-L614)，每次调用 checkAccess + 比较 authorID 是否变化 [L503-L521](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L503-L521) |
| 4层作者冒充拒绝（内层防线） | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleUserChanges()` op 循环 [L853-L920](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L853-L920)：属性编号合法性 / 跨 author 拒绝 / + 操作必有 author / 系统作者硬禁止 |
| 连接中会话过期实时检测 | [SessionManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts) + [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `findAuthorID()` [L39-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L39-L85) 实时 `now < validUntil`，每条消息重跑 + 触发 'rejected' 断开 |
| 重连流程 + 耐久身份判断 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleClientReady()` reconnect 分支 [L1208-L1269](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1208-L1269) + `hasDurableIdentity` 作者耐久判断 [L1295-L1327](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1295-L1327) |
