# Socket.IO 消息分派机制详解

本文档对照 Etherpad-lite 源码，梳理 Socket.IO 连接从建立、进入房间到消息路由与回执处理的完整链路。

---

## 一、会话绑定（Session Binding）

### 1.1 Socket.IO 服务端初始化与连接建立

入口文件：[socketio.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/hooks/express/socketio.ts)

**expressCreateServer** 完成以下工作：

1. 创建 Socket.IO Server 实例（挂在 Express HTTP server 上），配置传输协议、最大 HTTP 缓冲等。
2. 注册三层中间件链：

| 中间件 | 作用 | 代码位置 |
|--------|------|----------|
| `socketSessionMiddleware` | 把 Express session 绑定到 `socket.request`，解析 IP、从 query 或 Cookie 头还原 Cookie，然后调用 `express.sessionMiddleware` | [socketio.ts#L54-L69](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/hooks/express/socketio.ts#L54-L69) |
| `renewSession` | 监听每个 packet 到达时调用 `socket.request.session.touch()`，通知 express-session 保活；若 rolling=true 还会刷新 Cookie 过期时间 | [socketio.ts#L97-L108](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/hooks/express/socketio.ts#L97-L108) |
| `handleConnection` | 把 socket 加入全局 `sockets` Set（用于优雅关闭时等待所有客户端断开），并对 `session.connections++` 计数 | [socketio.ts#L83-L95](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/hooks/express/socketio.ts#L83-L95) |

此外还对 `/pluginfw/installer` 和 `/settings` 两个 namespace 单独注册同样的中间件链。

### 1.2 sessioninfos：Socket→会话 映射表

核心文件：[PadMessageHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts)

全局 `sessioninfos` 对象是整个消息分派的基石：

```
sessioninfos = {
  [socketId]: {          // key = socket.io socket.id
    auth: {              // 由 CLIENT_READY 填充
      sessionID,         // 来自 createSession() API，读自 HttpOnly Cookie
      padID,             // 用户请求的 padId（可能是只读 ID）
      token,             // author-token，读自 HttpOnly Cookie
    },
    author,              // 当前用户的 authorID (a.XXXXXXXXXXXX)
    padId,               // 真实（非只读）padId
    readOnlyPadId,       // 只读 padId
    readonly,            // 是否只读访问
    rev,                 // 已同步到该客户端的最新 revision
    time,                // rev 对应时间戳，用于计算 timeDelta
    embed,               // 是否为嵌入模式（历史 iframe）
  }
}
```

绑定过程分两步：

**第一步 — handleConnect：**
[PadMessageHandler.ts#L221-L226](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L221-L226)

Socket 刚连上时，只分配一个空对象占位 `sessioninfos[socket.id] = {}`。此时没有作者和 pad 信息，任何非 `CLIENT_READY` 消息都会被拒绝。

**第二步 — CLIENT_READY：**
[PadMessageHandler.ts#L398-L486](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L398-L486)

客户端发送的第一条消息必须是 `CLIENT_READY`，服务端做：

1. **Cookie 解析**：优先从 `socket.request.headers.cookie` 读取 `token` 和 `sessionID`（支持 Cookie `prefix`），兼容老客户端把 token 写在消息体里的做法（带 WARN 日志）。
2. **Pad ID 规范化**：不存在的 pad 调用 `padManager.sanitizePadId()` 清洗；通过 `readOnlyManager.getIds()` 解析出真实 padId / 只读 padId。
3. **权限判定**：`webaccess.userCanModify()` 决定 `readonly` 标志。
4. **安全检查**：`securityManager.checkAccess(padID, sessionID, token, user)` 返回 `accessStatus` 和 `authorID`。若作者 ID 在连接过程中改变（mid-session），直接断开 `disconnect: 'rejected'`。
5. **加入房间**：在 `handleClientReady()` 内执行 `socket.join(sessionInfo.padId)`，把 socket 加入以 padId 命名的 Socket.IO Room —— 这是后续广播的基础。

### 1.3 重复作者检测（Stale Tab 踢下线）

[PadMessageHandler.ts#L1169-L1186](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1169-L1186)

对于**未登录的匿名用户**（`req.session.user == null`），由于 authorID 是 cookie 派生的"每浏览器一份"，同一作者在同一 pad 出现两次通常意味着旧标签页未关闭。此时会对旧 socket 发送 `disconnect: 'userdup'` 并调用 `otherSocket.leave(padId)`。

已认证用户（SSO / basic auth）不受此限制，因为他们的 authorID 代表真实身份，跨窗口/跨设备同时在线是合法场景。嵌入模式（`embed=1`，历史 iframe）也不受限。

### 1.4 断开清理

[PadMessageHandler.ts#L246-L287](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L246-L287)

断开时：
1. 删除 `sessioninfos[socket.id]`。
2. 只有当该 author 在该 pad 的**最后一个 socket** 离开时，才向房间内广播 `USER_LEAVE` 并触发 `userLeave` hook —— 避免多窗口用户在关闭其中一个窗口时被错误标为离开。

---

## 二、消息路由（Message Routing）

### 2.1 SocketIORouter：组件级路由

文件：[SocketIORouter.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/SocketIORouter.ts)

Etherpad 在 Socket.IO 之上实现了一层轻量的"组件路由"：

```
客户端消息
    │
    ▼
socket.on('message', handler)
    │
    ├─► message.component  →  查找 components 字典
    │                          ('pad' → padMessageHandler)
    │
    ▼
components[name].handleMessage(socket, message)
```

**组件注册**发生在 [socketio.ts#L138-L140](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/hooks/express/socketio.ts#L138-L140)：

```typescript
socketIORouter.setSocketIO(io);
socketIORouter.addComponent('pad', padMessageHandler);
```

`setSocketIO` 内部监听 `io.sockets.on('connection')`，对每个 socket：

1. 通知所有组件 `handleConnect(socket)`；
2. 包装 `socket.send` 加入日志；
3. 监听 `message` 事件做组件分派；
4. 监听 `disconnect` 通知所有组件 `handleDisconnect(socket)`。

组件模块必须实现 `SocketModule` 接口（[SocketModule.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/types/SocketModule.ts)）以及 `handleConnect` / `handleMessage` / `handleDisconnect`。

### 2.2 客户端消息格式

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

### 2.3 PadMessageHandler.handleMessage：消息类型分发

[PadMessageHandler.ts#L377-L614](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L377-L614)

进入 `handleMessage` 后，会依次经过：

1. **限流**：`rateLimiter.consume(socket.request.ip)` —— 超过阈值回 `disconnect: 'rateLimited'`；
2. **前置校验**：session 存在、CLIENT_READY 已完成；
3. **安全检查**：
   - `securityManager.checkAccess()`：验证 token/sessionID 对该 pad 的权限；
   - `handleMessageSecurity` hook：插件可返回 `'permitOnce'` 对本消息临时授予写权限（绕过 readonly）；
   - `handleMessage` hook：任一插件返回 `null` 则丢弃该消息；
4. **断开检查**：处理期间若客户端已断开，抛错中止；
5. **按 `message.type` 分发表**：

| 顶层 type | 处理函数 | 说明 |
|-----------|----------|------|
| `CLIENT_READY` | `handleClientReady` | 握手、发送初始 `CLIENT_VARS`、加入 room |
| `CHANGESET_REQ` | `handleChangesetRequest` | 时间轴（timeslider）请求聚合 changeset |
| `COLLABROOM` | 二级分发（见下） | 实时协作域消息 |

**COLLABROOM 二级分发**（在 `readOnly` 检查之后）：

| data.type | 处理函数 | 说明 |
|-----------|----------|------|
| `USER_CHANGES` | `padChannels.enqueue(padId, ...)` → `handleUserChanges` | 编辑提交，**每 pad 串行队列** |
| `PAD_DELETE` | `handlePadDelete` | 删除 pad（创作者 / token / 全局开关三种授权） |
| `USERINFO_UPDATE` | `handleUserInfoUpdate` | 修改昵称/颜色，广播给房间其他用户 |
| `CHAT_MESSAGE` | `handleChatMessage` | 聊天消息入库 + 广播 |
| `GET_CHAT_MESSAGES` | `handleGetChatMessages` | 拉取聊天历史（最多 100 条） |
| `SAVE_REVISION` | `handleSaveRevisionMessage` | 标记保存版本 |
| `CLIENT_MESSAGE` | 三级分发（见下） | 自定义客户端消息 |

**CLIENT_MESSAGE 三级分发**：

| data.payload.type | 处理函数 |
|-------------------|----------|
| `suggestUserName` | `handleSuggestUserName` |
| `padoptions` | `handlePadOptionsMessage` |

### 2.4 USER_CHANGES 的串行队列保证

[PadMessageHandler.ts#L168-L207](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L168-L207)

同一 pad 可能同时收到多个客户端的编辑。若并行处理会导致 revisions 乱序，因此引入 `Channels` 类：

```typescript
const padChannels = new Channels((ch, {socket, message}) => handleUserChanges(socket, message));
```

- 每个 padId 对应一条 Promise 链；
- `enqueue(padId, task)` 把 task 追加到链尾，天然串行化；
- 链上某个 task 失败不会阻塞后续任务（`.catch(() => {})`），但每个任务自身会处理异常。

### 2.5 服务端→客户端广播方式

服务端推送消息有三种典型形式，均通过 `socket.emit('message', ...)`：

| 形式 | 代码示例 | 用途 |
|------|----------|------|
| 单播 | `socket.emit('message', {type: 'CLIENT_VARS', data: clientVars})` | 只发给当前连接的客户端 |
| 房间广播（不含自己） | `socket.broadcast.to(padId).emit('message', {...})` | 告知其他用户有新成员加入、某人离开等 |
| 房间广播（含自己） | `socketio.sockets.in(padId).emit('message', {...})` | 聊天消息、pad 设置变更 |

获取某 pad 所有 socket 的辅助函数：[`_getRoomSockets(padID)`](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1638-L1650)，直接从 adapter 的 rooms Map 同步读，避免了 socket.io v2/v3 `adapter.clients()` 的异步竞态。

---

## 三、回执处理（Acknowledgement / ACK）

### 3.1 Socket.IO 原生 ACK 机制

Socket.IO 支持"请求-响应"语义：客户端发送时可附带一个回调函数，服务端处理完后调用 `ack(...)` 把结果回传。

客户端侧（概念）：
```javascript
socket.emit('message', msg, (err, result) => {
  if (err) console.error(err);
  else console.log('got:', result);
});
```

### 3.2 服务端 ack 的实现

在 [SocketIORouter.ts#L80-L92](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/SocketIORouter.ts#L80-L92)：

```typescript
socket.on('message', (message: any, ack: any = () => {}) => (async () => {
  if (!message.component || !components[message.component]) {
    throw new Error(`unknown message component: ${message.component}`);
  }
  return await components[message.component].handleMessage(socket, message);
})().then(
    (val) => ack(null, val),
    (err) => {
      logger.error(`Error handling ${message.component} message from ${socket.id}: ${err.stack || err}`);
      ack({name: err.name, message: err.message}); // socket.io 不能传 Error 对象
    }));
```

关键点：

1. `handleMessage` 是 `async` 函数，其返回值会通过 Promise 链传给 ack；
2. **成功**：`ack(null, val)` —— 第一个参数为错误位（Node.js 惯例），第二个为返回值；
3. **失败**：`ack({name, message})` —— 由于 Socket.IO 序列化不能直接传 Error 对象，这里手动拆出 `name` 和 `message` 两个字段；
4. 默认值 `ack = () => {}` 保证即使客户端没传回调也不会炸。

当前 `padMessageHandler.handleMessage` 的返回值始终是 `undefined`（没有显式 `return`），所以大多数情况下 ack 只是回传一个空的 success 信号。但该机制为未来扩展和插件保留了通道。

### 3.3 应用层"回执"：ACCEPT_COMMIT 与 NEW_CHANGES

除了 Socket.IO 原生 ack，Etherpad 在应用层实现了**编辑提交的显式回执**，走的是普通广播通道而非 ack：

[PadMessageHandler.ts#L983-L984](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L983-L984)：

```typescript
socket.emit('message', {type: 'COLLABROOM', data: {type: 'ACCEPT_COMMIT', newRev}});
thisSession.rev = newRev;
```

客户端（collab_client）：
- 发送 `USER_CHANGES` 后进入等待状态，若收到 `ACCEPT_COMMIT` 则确认提交成功，推进本地 baseRev；
- 其他客户端收到 `NEW_CHANGES` 消息（由 `updatePadClients` 统一推送）把远程变更合入本地文档；
- 顺序保证：代码用 `assert.equal(thisSession.rev, r)` 断言 `ACCEPT_COMMIT` 和 `NEW_CHANGES` 按 revision 顺序发出。

失败时则发送 `socket.emit('message', {disconnect: 'badChangeset'})`，客户端据此断开并重建会话。

### 3.4 updatePadClients：批量差异推送

[PadMessageHandler.ts#L997-L1055](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L997-L1055)

每次编辑落库后调用 `updatePadClients(pad)`，遍历房间内所有 socket，把每个客户端从各自 `sessioninfo.rev` 到 `pad.head` 之间缺失的 revisions 逐个以 `NEW_CHANGES` 推送。同一 revision 会缓存在 `revCache` 中，避免重复从数据库读取。

---

## 四、完整消息流时序图

```
[Browser]                              [Server]
   │                                     │
   │  socket.connect()                   │
   │────────────────────────────────────►│
   │                                     │  handleConnection (socketio.ts)
   │                                     │   ├─ sockets.add(socket)
   │                                     │   ├─ session.connections++
   │                                     │   └─ handleConnect → sessioninfos[sid]={}
   │                                     │
   │  emit('message', {                  │
   │    component:'pad',                 │
   │    type:'CLIENT_READY',             │
   │    padId, token, ...                │
   │  })                                 │
   │────────────────────────────────────►│
   │                                     │  SocketIORouter.on('message')
   │                                     │   └─ components['pad'].handleMessage()
   │                                     │       ├─ securityManager.checkAccess()
   │                                     │       ├─ authorID 绑定
   │                                     │       ├─ socket.join(padId)  ◄── 进入房间
   │                                     │       └─ handleClientReady()
   │                                     │
   │  {type:'CLIENT_VARS', data:{...}}   │  ◄── 初始 pad 内容、作者列表、插件列表等
   │◄────────────────────────────────────│
   │                                     │
   │  {type:'USER_NEWINFO', ...}         │  ◄── 告知现有用户
   │◄────────────────────────────────────│
   │                                     │
   │  (用户输入)                          │
   │  emit('message', {                  │
   │    component:'pad',                 │
   │    type:'COLLABROOM',               │
   │    data:{type:'USER_CHANGES',...}   │
   │  }, ack)                            │
   │────────────────────────────────────►│
   │                                     │  padChannels.enqueue(padId, ...)
   │                                     │   └─ handleUserChanges()
   │                                     │       ├─ rebasing + 安全校验
   │                                     │       ├─ pad.appendRevision()
   │                                     │       ├─ emit ACCEPT_COMMIT  ◄── 回执
   │                                     │       └─ updatePadClients()
   │  {type:'ACCEPT_COMMIT', newRev:N}   │
   │◄────────────────────────────────────│
   │                                     │
   │                (其他客户端)          │
   │  {type:'NEW_CHANGES', newRev:N}     │  ◄── 广播给房间内其他客户端
   │◄────────────────────────────────────│
   │                                     │
   │  socket.disconnect()                │
   │────────────────────────────────────►│
   │                                     │  handleDisconnect()
   │                                     │   ├─ delete sessioninfos[sid]
   │                                     │   └─ 若为该作者最后一个 socket：
   │                                     │        广播 USER_LEAVE
   │                                     │
```

---

## 五、关键文件索引

| 文件 | 角色 |
|------|------|
| [socketio.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/hooks/express/socketio.ts) | Socket.IO 服务端初始化、中间件、组件注册 |
| [SocketIORouter.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/SocketIORouter.ts) | 组件级消息路由 + ack 封装 |
| [PadMessageHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts) | pad 组件核心：会话绑定、消息分发、编辑处理、广播 |
| [collab_client.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/collab_client.ts) | 客户端协作层：发送 USER_CHANGES、接收 ACCEPT_COMMIT / NEW_CHANGES |
| [pad.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/pad.ts) | 客户端 pad 页面：发起 CLIENT_READY 握手 |
| [SocketIOMessage.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/types/SocketIOMessage.ts) | 所有 Socket.IO 消息的 TypeScript 类型定义 |
| [SocketModule.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/types/SocketModule.ts) | Socket.IO 组件模块接口定义 |
