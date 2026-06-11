# Socket.IO 消息分派机制详解

本文档对照 Etherpad-lite 源码，梳理 Socket.IO 连接从建立、进入房间到消息路由与回执处理的完整链路。

---

## 一、连接建立：中间件流程

### 1.1 初始化阶段：注册中间件与事件监听

入口文件：[socketio.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/hooks/express/socketio.ts)

`expressCreateServer` 按以下顺序注册中间件和事件监听器：

```typescript
// ① 创建 Socket.IO Server 实例
io = new Server(args.server, { transports, cookie: false, maxHttpBufferSize });

// ② 注册 connection 事件处理器 A：handleConnection
io.on('connection', handleConnection);                          // L111

// ③ 注册命名空间级中间件：socketSessionMiddleware（第 1 个 io.use）
io.use(socketSessionMiddleware(args));                           // L113

// ④ 注册额外命名空间（各自的中间件链）
io.of('/pluginfw/installer').on('connection', handleConnection).use(socketSessionMiddleware(args)).use(renewSession);
io.of('/settings').on('connection', handleConnection).use(socketSessionMiddleware(args)).use(renewSession);

// ⑤ 注册命名空间级中间件：renewSession（第 2 个 io.use）
io.use(renewSession);                                           // L125

// ⑥ 注册 SocketIORouter（内部会再注册一个 connection 监听器 B）
socketIORouter.setSocketIO(io);                                 // L139
socketIORouter.addComponent('pad', padMessageHandler);           // L140
```

> **关键理解**：Socket.IO v4 的中间件机制是——每当一个新 socket 连接到某命名空间时，**先执行该命名空间上所有 `io.use()` 注册的中间件**（按注册顺序），**全部通过后才触发 `connection` 事件**。因此上面 ② 和 ⑥ 注册的两个 connection 监听器，都在中间件链之后才执行。

### 1.2 每次新连接的实际执行时序

当一个客户端连接到默认命名空间 `/` 时，执行流程如下：

```
客户端发起连接
       │
       ▼
┌─ 中间件阶段 ──────────────────────────────────────────────┐
│                                                            │
│  1. socketSessionMiddleware                                │
│     ├─ 将 Express session 绑定到 socket.request            │
│     ├─ 解析 IP（proxyaddr 或 handshake.address）           │
│     ├─ 若无 Cookie header，从 query.cookie 补充            │
│     └─ 调用 express.sessionMiddleware(req, {}, next)       │
│                                                            │
│  2. renewSession                                           │
│     ├─ 监听 socket.conn 的 'packet' 事件                   │
│     └─ 每次收到 packet 时调用 session.touch() 保活         │
│                                                            │
└────────────────────────────────────────────────────────────┘
       │ 中间件全部通过
       ▼
┌─ connection 事件阶段 ─────────────────────────────────────┐
│                                                            │
│  3. handleConnection  (socketio.ts#L83)                    │
│     ├─ sockets.add(socket)    加入全局连接集合              │
│     ├─ session.connections++  递增连接计数                  │
│     └─ session.save()         持久化 session               │
│                                                            │
│  4. SocketIORouter 的 connection 监听器 (L64)              │
│     ├─ 包装 socket.send 加入调试日志                       │
│     ├─ 调用所有组件的 handleConnect(socket)                 │
│     │   └─ padMessageHandler.handleConnect                  │
│     │       └─ sessioninfos[socket.id] = {}  ← 占位空对象  │
│     ├─ 注册 socket.on('message', ...) 消息处理器            │
│     └─ 注册 socket.on('disconnect', ...) 断开处理器        │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**核心要点**：

- `handleConnection`（socketio.ts）和 `SocketIORouter` 的 connection 监听器是**两个独立的监听器**，都会执行，按注册顺序依次触发。
- `sessioninfos[socket.id] = {}` 在 SocketIORouter 的 `handleConnect` 中完成，此时只是空对象——还没有 author 和 pad 信息。
- 中间件阶段完成后，`socket.request.session` 已就绪，后续所有消息处理都可以安全访问 session。

---

## 二、会话绑定：从空连接到进入房间

### 2.1 sessioninfos 字典结构

核心文件：[PadMessageHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts)

```
sessioninfos = {
  [socketId]: {
    auth: {              // 由 CLIENT_READY 消息填充
      sessionID,         // 来自 HttpOnly Cookie 或已废弃的消息体字段
      padID,             // 用户请求的 padId（可能是只读 ID）
      token,             // author-token，来自 HttpOnly Cookie
    },
    author,              // 当前用户的 authorID (a.XXXXXXXXXXXX)
    padId,               // 真实（非只读）padId
    readOnlyPadId,       // 只读 padId
    readonly,            // 是否只读访问
    rev,                 // 已同步到该客户端的最新 revision 号
    time,                // rev 对应的时间戳，用于计算 timeDelta
    embed,               // 是否为嵌入模式（in-place history iframe）
  }
}
```

### 2.2 绑定过程：两步走

**第一步 — handleConnect（连接时立即执行）**

[PadMessageHandler.ts#L221-L226](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L221-L226)

```typescript
exports.handleConnect = (socket) => {
  stats.meter('connects').mark();
  sessioninfos[socket.id] = {};   // 仅占位
};
```

此时 `sessioninfos[socket.id]` 是空对象。在 `handleMessage` 中（[L496-L501](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L496-L501)），如果 `thisSession.auth` 为空，任何非 `CLIENT_READY` 消息都会被拒绝：

```typescript
const auth = thisSession.auth;
if (!auth) {
  throw new Error(`pre-CLIENT_READY message from IP ${ip}: ${msg}`);
}
```

**第二步 — CLIENT_READY 消息处理**

[PadMessageHandler.ts#L398-L486](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L398-L486)

客户端发送的第一条消息必须是 `CLIENT_READY`，服务端完成：

1. **Cookie 解析**：优先从 `socket.request.headers.cookie` 读取 `token` 和 `sessionID`（支持 `settings.cookie.prefix`），兼容老客户端把 token 写在消息体里的做法（带 WARN 日志）。
2. **填充 `thisSession.auth`**：将 `sessionID`、`padID`、`token` 保存到 `sessioninfos[socket.id].auth`。
3. **Pad ID 规范化**：不存在的 pad 调用 `padManager.sanitizePadId()` 清洗；通过 `readOnlyManager.getIds()` 解析出真实 padId / 只读 padId。
4. **权限判定**：`webaccess.userCanModify()` 决定 `readonly` 标志。
5. **安全检查**（在 CLIENT_READY 之后、进入房间之前执行）：`securityManager.checkAccess()` 返回 `accessStatus` 和 `authorID`。若 authorID 在连接过程中改变，直接断开 `disconnect: 'rejected'`。
6. **加入房间**：在 `handleClientReady()` 内执行 `socket.join(sessionInfo.padId)`，把 socket 加入以 padId 命名的 Socket.IO Room——这是后续房间广播的基础。

### 2.3 重复作者检测（Stale Tab 踢下线）

[PadMessageHandler.ts#L1169-L1186](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1169-L1186)

对**未登录的匿名用户**（`req.session.user == null` 且非 embed 模式），遍历房间内现有 socket，若发现相同 authorID：
- 旧 socket 被踢：`otherSocket.emit('message', {disconnect: 'userdup'})` + `otherSocket.leave(padId)`
- `sessioninfos[otherSocket.id] = {}` 清空旧会话

已认证用户（SSO / basic auth）和 embed 模式不受此限制。

### 2.4 断开清理

[PadMessageHandler.ts#L246-L287](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L246-L287)

断开时有两个监听器分别响应：

1. **socketio.ts 的 handleConnection 中注册的**：`sockets.delete(socket)` + `socketsEvents.emit('updated')`
2. **SocketIORouter 中注册的**：遍历所有组件调用 `handleDisconnect(socket)` → `padMessageHandler.handleDisconnect`

在 `padMessageHandler.handleDisconnect` 中：
- 删除 `sessioninfos[socket.id]`
- 只有当该 author 在该 pad 的**最后一个 socket** 离开时，才向房间内广播 `USER_LEAVE` 并触发 `userLeave` hook——避免多窗口用户在关闭其中一个窗口时被错误标为离开。

---

## 三、消息路由：组件分派与类型分发

### 3.1 SocketIORouter：组件级路由

文件：[SocketIORouter.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/SocketIORouter.ts)

Etherpad 在 Socket.IO 之上实现了一层轻量的"组件路由"：

```
客户端消息
    │
    ▼
socket.on('message', handler)       ← SocketIORouter 注册
    │
    ├─► message.component  →  查找 components 字典
    │                          ('pad' → padMessageHandler)
    │
    ▼
components[name].handleMessage(socket, message)
```

`setSocketIO` 内部（[SocketIORouter.ts#L61-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/SocketIORouter.ts#L61-L107)）对每个 socket：
1. 通知所有组件 `handleConnect(socket)`；
2. 包装 `socket.send` 加入日志；
3. 监听 `message` 事件做组件分派 + ack 封装；
4. 监听 `disconnect` 通知所有组件 `handleDisconnect(socket)`。

组件模块必须实现 [SocketModule](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/types/SocketModule.ts) 接口（`setSocketIO`）以及 `handleConnect` / `handleMessage` / `handleDisconnect`。

### 3.2 客户端消息格式

客户端必须在每条消息里带上 `component: 'pad'`，否则 SocketIORouter 会抛 `unknown message component`。

示例 — [pad.ts#L327-L350](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/pad.ts#L327-L350)（CLIENT_READY）和 [collab_client.ts#L179-L184](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/collab_client.ts#L179-L184)（编辑提交）：

```javascript
// CLIENT_READY
socket.emit("message", {
  component: 'pad',
  type: 'CLIENT_READY',
  padId, userInfo, ...
});

// USER_CHANGES（编辑提交）
getSocket().emit('message', {
  type: 'COLLABROOM',
  component: 'pad',
  data: { type: 'USER_CHANGES', baseRev, changeset, apool }
});
```

### 3.3 PadMessageHandler.handleMessage：消息类型分发

[PadMessageHandler.ts#L377-L614](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L377-L614)

进入 `handleMessage` 后，依次经过：

1. **限流**：`rateLimiter.consume(socket.request.ip)` —— 超过阈值回 `disconnect: 'rateLimited'`；
2. **前置校验**：session 存在、CLIENT_READY 已完成（`auth` 非空）；
3. **安全检查**：
   - `securityManager.checkAccess()`：验证 token/sessionID 对该 pad 的权限；
   - `handleMessageSecurity` hook：插件可返回 `'permitOnce'` 对本消息临时授予写权限（绕过 readonly）；
   - `handleMessage` hook：任一插件返回 `null` 则丢弃该消息；
4. **断开检查**：处理期间若客户端已断开，抛错中止；
5. **按 `message.type` 分发**：

| 顶层 type | 处理函数 | 说明 |
|-----------|----------|------|
| `CLIENT_READY` | `handleClientReady` | 握手、发送初始 `CLIENT_VARS`、加入 room |
| `CHANGESET_REQ` | `handleChangesetRequest` | 时间轴请求聚合 changeset |
| `COLLABROOM` | 二级分发（见下） | 实时协作域消息 |

**COLLABROOM 二级分发**（在 `readOnly` 检查之后）：

| data.type | 处理函数 | 说明 |
|-----------|----------|------|
| `USER_CHANGES` | `padChannels.enqueue(padId, ...)` → `handleUserChanges` | 编辑提交，**每 pad 串行队列** |
| `PAD_DELETE` | `handlePadDelete` | 删除 pad |
| `USERINFO_UPDATE` | `handleUserInfoUpdate` | 修改昵称/颜色 |
| `CHAT_MESSAGE` | `handleChatMessage` | 聊天消息 |
| `GET_CHAT_MESSAGES` | `handleGetChatMessages` | 拉取聊天历史 |
| `SAVE_REVISION` | `handleSaveRevisionMessage` | 标记保存版本 |
| `CLIENT_MESSAGE` | 三级分发 | 自定义客户端消息 |

**CLIENT_MESSAGE 三级分发**：

| data.payload.type | 处理函数 |
|-------------------|----------|
| `suggestUserName` | `handleSuggestUserName` |
| `padoptions` | `handlePadOptionsMessage` |

### 3.4 USER_CHANGES 的串行队列保证

[PadMessageHandler.ts#L168-L207](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L168-L207)

同一 pad 可能同时收到多个客户端的编辑。若并行处理会导致 revisions 乱序，因此引入 `Channels` 类：

```typescript
const padChannels = new Channels((ch, {socket, message}) => handleUserChanges(socket, message));
```

- 每个 padId 对应一条 Promise 链；
- `enqueue(padId, task)` 把 task 追加到链尾，天然串行化；
- 链上某个 task 失败不会阻塞后续任务（`.catch(() => {})`），但每个任务自身会处理异常。

### 3.5 服务端→客户端推送方式汇总

| 方式 | API | 接收者 | 典型用途 |
|------|-----|--------|----------|
| 单播 | `socket.emit('message', msg)` | 仅该 socket | `CLIENT_VARS`、`ACCEPT_COMMIT`、`CHANGESETS_REQ` 回复 |
| 房间广播（不含自己） | `socket.broadcast.to(padId).emit('message', msg)` | 房间内除发送者外的所有 socket | `USER_NEWINFO`（新用户加入通知）、`USER_LEAVE` |
| 房间遍历逐个推送 | `_getRoomSockets(padId)` + 逐个 `socket.emit(...)` | 房间内满足条件的所有 socket（**含提交者**） | `NEW_CHANGES`（差异推送） |
| 房间全量广播 | `socketio.sockets.in(padId).emit('message', msg)` | 房间内所有 socket | `CHAT_MESSAGE`、`CUSTOM` 消息 |

获取某 pad 所有 socket 的辅助函数：[`_getRoomSockets(padID)`](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1638-L1650)，直接从 adapter 的 rooms Map 同步读。

---

## 四、编辑提交确认与房间广播：谁收到什么

这是最容易混淆的部分。一次 `USER_CHANGES` 的处理涉及**两条不同类型的消息**，分别发给**不同的人**。

### 4.1 handleUserChanges 核心流程

[PadMessageHandler.ts#L808-L995](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L808-L995)

```
客户端 A 提交 USER_CHANGES
       │
       ▼
  padChannels.enqueue(padId, ...)   ← 串行队列，避免同一 pad 的编辑并发
       │
       ▼
  handleUserChanges(socket_A, message)
       │
       ├─ 1. 校验 changeset（语法、author 属性、系统 author 拒绝）
       ├─ 2. moveOpsToNewPool：映射到 pad 全局属性池
       ├─ 3. rebasing：while (r < pad.head) follow() 逐步追上最新
       ├─ 4. 检查 oldLen 匹配 + 尾部 '\n' 不变量
       ├─ 5. pad.appendRevision(rebasedChangeset, author)
       ├─ 6. 可选：_correctMarkersInPad + appendRevision
       │
       ├─ 7. ★ socket.emit → ACCEPT_COMMIT（发给客户端 A）
       ├─ 8.   thisSession.rev = newRev（更新 A 的已同步 revision）
       │
       └─ 9. ★ updatePadClients(pad)（推送给房间内所有落后的 socket）
```

### 4.2 ACCEPT_COMMIT：只发给提交者

[PadMessageHandler.ts#L983-L984](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L983-L984)

```typescript
socket.emit('message', {type: 'COLLABROOM', data: {type: 'ACCEPT_COMMIT', newRev}});
thisSession.rev = newRev;
```

- `socket.emit` 是**单播**，只有提交 USER_CHANGES 的那个 socket 收到。
- 紧接着将 `thisSession.rev` 更新到 `newRev`，这样后续的 `updatePadClients` 就不会重复向该 socket 推送这次 revision。
- 客户端收到 `ACCEPT_COMMIT` 后确认本地编辑已落库，推进 baseRev，可以继续提交下一个编辑。

### 4.3 NEW_CHANGES：发给房间内所有"落后"的 socket

[PadMessageHandler.ts#L997-L1055](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L997-L1055)

```typescript
exports.updatePadClients = async (pad) => {
  const roomSockets = _getRoomSockets(pad.id);   // 房间内所有 socket
  if (roomSockets.length === 0) return;

  await Promise.all(roomSockets.map(async (socket) => {
    const sessioninfo = sessioninfos[socket.id];
    if (sessioninfo == null) return;

    while (sessioninfo.rev < pad.getHeadRevisionNumber()) {
      // 逐个推送缺失的 revision
      socket.emit('message', {type: 'COLLABROOM', data: {type: 'NEW_CHANGES', ...}});
      sessioninfo.rev = r;
    }
  }));
};
```

**谁收到 NEW_CHANGES？** 逻辑非常精确——基于每个 socket 的 `sessioninfo.rev` 与 `pad.head` 的差值：

| socket 身份 | 收到 ACCEPT_COMMIT？ | 收到 NEW_CHANGES？ | 原因 |
|-------------|---------------------|-------------------|------|
| **提交者 A** | ✅ 收到（单播） | ❌ 不收到（通常） | `thisSession.rev` 已在 L984 被更新为 `newRev`，while 循环条件不满足 |
| **房间内其他用户** | ❌ 不收到 | ✅ 收到 | 他们的 `sessioninfo.rev` 仍停留在旧值，while 循环触发推送 |
| **刚加入房间的新用户** | ❌ 不收到 | ✅ 收到 | 如果新用户在 `handleClientReady` 的 `socket.join` 之后、`updatePadClients` 之前连入，可能捕获到此次推送 |

> **特殊情况**：如果 `handleUserChanges` 产生了 `_correctMarkersInPad` 修正 changeset（[L971-L974](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L971-L974)），会追加一个额外的 revision。此时提交者 A 的 `thisSession.rev = newRev`（用户编辑的 revision），但 `pad.head` 已经推进到了修正 revision，所以 A 也会收到一条针对修正 revision 的 `NEW_CHANGES`。

### 4.4 顺序保证

[PadMessageHandler.ts#L976-L978](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L976-L978)

```typescript
// The client assumes that ACCEPT_COMMIT and NEW_CHANGES messages arrive in order.
assert.equal(thisSession.rev, r);
```

这个断言确保：在发送 `ACCEPT_COMMIT` 之前，该提交者之前所有的 `ACCEPT_COMMIT` / `NEW_CHANGES` 都已经发出。因为 `handleUserChanges` 在 `padChannels` 串行队列中执行，同一 pad 的编辑不会交叉，保证了 revision 的单调递增。

### 4.5 失败处理

[PadMessageHandler.ts#L987-L991](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L987-L991)

```typescript
catch (err) {
  socket.emit('message', {disconnect: 'badChangeset'});
  stats.meter('failedChangesets').mark();
}
```

失败时**只通知提交者**，其他房间成员不受影响。客户端收到 `{disconnect: 'badChangeset'}` 后断开并尝试重建会话。

---

## 五、回执处理（Acknowledgement / ACK）

### 5.1 Socket.IO 原生 ACK 机制

Socket.IO 支持"请求-响应"语义：客户端发送时可附带一个回调函数，服务端处理完后调用 `ack(...)` 把结果回传。

客户端侧（概念）：
```javascript
socket.emit('message', msg, (err, result) => {
  if (err) console.error(err);
  else console.log('got:', result);
});
```

### 5.2 服务端 ack 的实现

在 [SocketIORouter.ts#L80-L92](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/SocketIORouter.ts#L80-L92)：

```typescript
socket.on('message', (message, ack = () => {}) => (async () => {
  if (!message.component || !components[message.component]) {
    throw new Error(`unknown message component: ${message.component}`);
  }
  return await components[message.component].handleMessage(socket, message);
})().then(
    (val) => ack(null, val),
    (err) => {
      logger.error(`Error handling ...: ${err.stack || err}`);
      ack({name: err.name, message: err.message});
    }));
```

关键点：

1. `handleMessage` 是 `async` 函数，其返回值通过 Promise 链传给 ack；
2. **成功**：`ack(null, val)` —— Node.js 惯例，第一个参数为错误位；
3. **失败**：`ack({name, message})` —— Socket.IO 序列化不能传 Error 对象，手动拆出两个字段；
4. 默认值 `ack = () => {}` 保证客户端没传回调也不会崩溃；
5. 当前 `padMessageHandler.handleMessage` 没有显式 `return`，ack 值为 `undefined`。

### 5.3 两套回执机制对比

| 维度 | Socket.IO 原生 ACK | 应用层 ACCEPT_COMMIT / NEW_CHANGES |
|------|--------------------|------------------------------------|
| 传输通道 | Socket.IO 内建回调 | 普通 `socket.emit('message', ...)` |
| 触发时机 | `handleMessage` 函数 resolve/reject 时 | `handleUserChanges` 内，revision 落库后 |
| 接收者 | 仅发起请求的客户端 | ACCEPT_COMMIT → 提交者；NEW_CHANGES → 房间内所有落后 socket |
| 语义 | "消息是否被服务器处理了" | "编辑是否被服务器采纳并持久化了" |
| 数据内容 | 成功/失败的简单信号 | `newRev` 号、changeset、apool 等完整数据 |
| 当前用途 | 保留通道，多数消息不用 | 编辑流程的核心确认机制 |

---

## 六、完整消息流时序图

```
[客户端 A]                            [服务端]                        [客户端 B]
    │                                    │                               │
    │  socket.connect()                  │                               │
    │───────────────────────────────────►│                               │
    │                                    │                               │
    │              ┌ 中间件 ──────────┐  │                               │
    │              │ socketSession    │  │                               │
    │              │ Middleware       │  │                               │
    │              │     → renewSession│  │                               │
    │              └─────────────────┘  │                               │
    │                                    │                               │
    │              ┌ connection ───────┐ │                               │
    │              │ handleConnection  │ │                               │
    │              │ SocketIORouter    │ │                               │
    │              │  → handleConnect  │ │                               │
    │              │    sessioninfos   │ │                               │
    │              │    [sid] = {}     │ │                               │
    │              └──────────────────┘ │                               │
    │                                    │                               │
    │  emit('message', {                 │                               │
    │    component:'pad',                │                               │
    │    type:'CLIENT_READY',            │                               │
    │    padId, ...                      │                               │
    │  })                                │                               │
    │───────────────────────────────────►│                               │
    │                                    │                               │
    │                                    │ handleMessage:                │
    │                                    │  ├─ checkAccess + author 绑定 │
    │                                    │  └─ handleClientReady:        │
    │                                    │      ├─ socket.join(padId)    │
    │                                    │      ├─ 踢重复作者(匿名)      │
    │                                    │      └─ socket.emit:          │
    │  ◄─── {type:'CLIENT_VARS'} ───────│                               │
    │                                    │                               │
    │  ◄─── {type:'USER_NEWINFO'} ──────│  (broadcast to room)         │
    │                                    │                               │
    │                                    │  (A 加入房间后，B 也会收到)   │ ◄── USER_NEWINFO(A)
    │                                    │                               │
    │                                    │                               │
    │  (A 输入文字)                       │                               │
    │  emit('message', {                 │                               │
    │    component:'pad',                │                               │
    │    type:'COLLABROOM',              │                               │
    │    data:{type:'USER_CHANGES',...}  │                               │
    │  })                                │                               │
    │───────────────────────────────────►│                               │
    │                                    │                               │
    │                                    │ padChannels.enqueue(padId,…)  │
    │                                    │  └─ handleUserChanges:        │
    │                                    │      ├─ rebase + appendRev    │
    │                                    │      ├─ socket.emit ──────┐   │
    │  ◄─── ACCEPT_COMMIT(newRev:N) ────│◄──────────────────────────┘   │
    │      (仅 A 收到)                    │                               │
    │                                    │      └─ updatePadClients:     │
    │                                    │          遍历 roomSockets:    │
    │                                    │          A: rev==head, 跳过   │
    │                                    │          B: rev<head, 推送 ──┐│
    │                                    │                               ││
    │                                    │◄──────────────────────────────┘│
    │                                    │                               │
    │                                    │              ◄── NEW_CHANGES ─│
    │                                    │                (仅 B 收到)     │
    │                                    │                               │
    │                                    │                               │
    │  socket.disconnect()               │                               │
    │───────────────────────────────────►│                               │
    │                                    │ handleDisconnect:              │
    │                                    │  ├─ delete sessioninfos[sid]  │
    │                                    │  └─ 若为 A 最后一个 socket:   │
    │                                    │     broadcast USER_LEAVE ─────│──►
    │                                    │                               │
```

---

## 七、关键文件索引

| 文件 | 角色 |
|------|------|
| [socketio.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/hooks/express/socketio.ts) | Socket.IO 服务端初始化、中间件注册、组件注册 |
| [SocketIORouter.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/SocketIORouter.ts) | 组件级消息路由 + ack 封装 |
| [PadMessageHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts) | pad 组件核心：会话绑定、消息分发、编辑处理、广播 |
| [collab_client.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/collab_client.ts) | 客户端协作层：发送 USER_CHANGES、接收 ACCEPT_COMMIT / NEW_CHANGES |
| [pad.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/pad.ts) | 客户端 pad 页面：发起 CLIENT_READY 握手 |
| [SocketIOMessage.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/types/SocketIOMessage.ts) | 所有 Socket.IO 消息的 TypeScript 类型定义 |
| [SocketModule.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/types/SocketModule.ts) | Socket.IO 组件模块接口定义 |
