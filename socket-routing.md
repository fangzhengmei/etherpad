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

### 2.3 handleClientReady：首次连接 vs 重连恢复（两条路径详解）

`handleClientReady` 函数（[PadMessageHandler.ts#L1107-L1477](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1107-L1477)）根据客户端 `CLIENT_READY` 消息中的 `reconnect` 字段走两条不同的分支。

客户端何时发送 reconnect：[pad.ts#L344-L348](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/pad.ts#L344-L348)

```javascript
// 发生网络抖动后 socket 重新连上时：
if (isReconnect) {
  msg.client_rev = pad.collabClient.getCurrentRevisionNumber();
  msg.reconnect = true;
}
```

#### 2.3.1 公共前置步骤（两条路径都执行）

在进入分支判断前，`handleClientReady` 会先完成一系列共用操作：

1. 保存/更新作者名与颜色（`authorManager.setAuthorName/ColorId`）
2. 加载 pad 对象、所有作者数据、历史作者信息（颜色+名称）
3. 重复作者检测（见下一节 2.4）
4. 写访问日志 `[ENTER]` / `[CREATE]`

之后才按 `message.reconnect` 分支。

---

#### 2.3.2 路径 A：重连恢复（`reconnect === true`）

代码位置：[PadMessageHandler.ts#L1197-L1258](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1197-L1258)

```typescript
if (message.reconnect) {
  socket.join(sessionInfo.padId);                        // ① 重新加入房间
  sessionInfo.rev = message.client_rev;                  // ② 以客户端上报的 rev 作为基线
  // ③ 计算 [client_rev+1, pad.head] 范围内的缺失 revisions
  // ④ 从数据库批量加载 changeset / author / timestamp
  // ⑤ 逐条发送 CLIENT_RECONNECT 消息
}
```

**为什么不再发送 CLIENT_VARS？**

重连的前提是客户端页面尚未刷新，内存中仍然保留着：
- 完整的 pad 文档内容（`atext`）与属性池
- 插件列表、用户列表、pad 设置等所有初始变量
- collab_client 的状态机（baseRev、待提交队列等）

发送 `CLIENT_VARS` 会让客户端重新初始化整个编辑器——清空当前输入、重建 DOM、重置光标位置，这在重连场景下是不可接受的。用户的期望是"短暂断网后继续打字，感觉不到掉线"，所以重连分支只补**增量 changeset**。

**重连分支向客户端发送的消息：**

对每条缺失 revision（从 `client_rev + 1` 到 `pad.head`）发送一条：

```javascript
{
  type: 'COLLABROOM',
  data: {
    type: 'CLIENT_RECONNECT',
    headRev: pad.getHeadRevisionNumber(),   // 当前最新 revision
    newRev: r,                                // 本条消息对应的 revision 号
    changeset: forWire.translated,           // 序列化后的 changeset
    apool: forWire.pool,                      // 该 changeset 涉及的属性池切片
    author: changesets[r].author,            // 该 revision 的作者
    currentTime: changesets[r].timestamp     // 该 revision 的时间戳
  }
}
```

如果期间没有任何新 revision（`startNum === endNum`），则发送一条 `noChanges: true` 的消息：

```javascript
{
  type: 'COLLABROOM',
  data: {
    type: 'CLIENT_RECONNECT',
    noChanges: true,
    newRev: pad.getHeadRevisionNumber()
  }
}
```

**注意**：重连分支不设置 `sessionInfo.time`，也不调用 `updatePadClients`——因为消息已经直接在本分支里逐条发出了。

---

#### 2.3.3 路径 B：首次连接 / 刷新页面（`reconnect` 为 false / 未设置）

代码位置：[PadMessageHandler.ts#L1259-L1413](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1259-L1413)

这是普通首次打开页面或刷新页面的场景。客户端内存里没有任何 pad 状态，必须从零构建。

```typescript
else {
  // ① 原子快照：同时读取 pad.head 和 atext，避免两者不一致
  headRev = pad.getHeadRevisionNumber();
  atext = cloneAText(pad.atext);

  // ② 组装庞大的 clientVars 对象

  // ③ 调用 clientVars 插件 hook（可能有异步操作）
  await hooks.aCallAll('clientVars', {clientVars, pad, socket});

  // ④ 加入房间
  socket.join(sessionInfo.padId);

  // ⑤ 发送 CLIENT_VARS
  socket.emit('message', {type: 'CLIENT_VARS', data: clientVars});

  // ⑥ 记录当前已同步到客户端的 rev 和 time
  sessionInfo.rev = headRev;
  sessionInfo.time = await pad.getRevisionDate(headRev);

  // ⑦ ★ 补发：推送在 ③④ 期间新增的 revisions
  await exports.updatePadClients(pad);
}
```

**为什么 `socket.join` 之后还要再次 `updatePadClients`？**

关键在于步骤 ③ 的 `hooks.aCallAll('clientVars', ...)` 是**异步**的，可能耗时很长（插件做 DB 查询、HTTP 请求等）。同时在 ① 到 ④ 之间，也可能有其他客户端在提交编辑。

这段时间窗口内发生的事：
- 步骤 ① 拍快照时 `pad.head = N`
- 步骤 ③ 插件 hook 执行期间，可能已有其他用户提交了编辑，`pad.head` 推进到了 `N + k`
- 这些新 revision 在步骤 ③ 之前就通过 `updatePadClients` 向房间内已有的 socket 做了广播
- **但本 socket 在步骤 ④ 才 `socket.join(padId)`，之前广播的 NEW_CHANGES 它一条也收不到**
- 步骤 ⑤ 发出去的 `CLIENT_VARS` 只包含了 snapshot at rev `N`，缺了 `N+1` ~ `N+k`

所以步骤 ⑦ 的 `await exports.updatePadClients(pad)` 就是专门用来填补这个窗口的。它遍历 `roomSockets`，发现 `sessionInfo.rev (N) < pad.head (N+k)`，就把 `N+1` ~ `N+k` 的 `NEW_CHANGES` 逐条推送给本 socket。

代码里的注释也明确说明了这一点：[PadMessageHandler.ts#L1409-L1412](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1409-L1412)

> *Flush any revisions that may have been appended while we were awaiting the clientVars hook (before socket.join). Those revisions were broadcast to existing room members but this socket hadn't joined yet so it missed them.*

**首次连接分支向客户端发送的消息：**

| 消息类型 | 内容概览 | 发送时机 |
|----------|----------|----------|
| `CLIENT_VARS` | 完整初始状态：`initialAttributedText`（atext + apool at headRev）、`historicalAuthorData`、`collab_client_vars.rev/time`、`chatHead`、`savedRevisions`、插件列表、用户信息、pad 设置、各种开关与 UI 配置、删除 token 等 | ⑤ `socket.emit` 单播 |
| `NEW_CHANGES`（若干条） | 每条对应一个缺失的 revision：`newRev`、`changeset`、`apool`、`author`、`currentTime`、`timeDelta` | ⑦ `updatePadClients` 内逐条单播 |

另外在两条分支**之后**（函数末尾），还会向客户端发送：

| 消息 | 方向 | 说明 |
|------|------|------|
| `USER_NEWINFO`（其他用户给新用户） | 其他用户 → 新用户（单播） | 遍历房间内其他 socket，把每个在线用户的颜色/名称/userId 发给新用户 |
| `USER_NEWINFO`（新用户给其他人） | 新用户 → 房间内其他人（broadcast） | `socket.broadcast.to(padId)` 通知其他用户有新成员加入 |

#### 2.3.4 两条路径消息对比

| 维度 | 重连恢复（reconnect=true） | 首次连接 / 刷新 |
|------|---------------------------|------------------|
| 发送 `CLIENT_VARS`？ | ❌ 不发送（客户端保留内存状态） | ✅ 发送（从零构建完整文档与 UI 状态） |
| 基线 revision 来源 | 客户端上报 `message.client_rev` | 服务端拍快照时的 `pad.getHeadRevisionNumber()` |
| 缺失 revision 消息类型 | `CLIENT_RECONNECT` | `NEW_CHANGES` |
| 缺失 revision 发送方式 | 在本分支内直接 for 循环 `socket.emit` | 调用通用的 `updatePadClients(pad)` |
| 是否设置 `sessionInfo.time` | ❌ 不设置 | ✅ 用 headRev 的时间戳初始化（否则 timeDelta=NaN） |
| 是否触发 `clientVars` 插件 hook | ❌ 不触发 | ✅ 触发（允许插件注入初始变量） |
| 原子快照（atext + headRev） | ❌ 不需要 | ✅ 需要（避免文档与 revision 号不一致，见 issue #4040） |

---

#### 2.3.5 重连恢复的完整状态机：服务端 sessionInfo 维护 + 客户端区分自身提交与远端变更

上一节对比了两条路径的宏观差异，这一节深入重连恢复的细节，回答三个问题：
1. 服务端发送完 `CLIENT_RECONNECT` 之后，`sessionInfo.rev` 和 `sessionInfo.time` 是何时追上 `pad.head` 的？
2. 客户端收到 `CLIENT_RECONNECT` 时如何区分"这是我自己离线期间提交的编辑"和"这是别人写的"？
3. `isPendingRevision` 状态从 `true` 何时回到 `false`，回到正常后做了什么？

##### A. 客户端侧：从断开 → 重连尝试 → pending 状态建立

断开触发入口在 [pad.ts#L390-L396](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/pad.ts#L390-L396)：

```javascript
socket.on('disconnect', (reason) => {
  socketReconnecting();
});
socket.io.on('reconnect_attempt', socketReconnecting); // 额外兜底
socket.on('error', (error) => {                         // socket.io 层报错
  pad.collabClient.setStateIdle();
  pad.collabClient.setIsPendingRevision(true);
});
```

`setChannelState('RECONNECTING')` 之前会调用 `socketReconnecting()`（[pad.ts#L381-L388](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/pad.ts#L381-L388)）：

```javascript
const socketReconnecting = () => {
  pad.collabClient.setStateIdle();        // ① committing=false，中断当前 commit 状态机
  pad.collabClient.setIsPendingRevision(true); // ② 进入 pending 状态
  pad.collabClient.setChannelState('RECONNECTING'); // ③ 标记 UI 上的重连提示
};
```

`setIsPendingRevision(true)` 的核心作用：在 collab_client 的 `handleUserChanges` 中（[collab_client.ts#L132-L152](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/collab_client.ts#L132-L152)），只有 `!isPendingRevision` 时才允许提交新编辑：

```javascript
if (!isPendingRevision) {
  // 可以提交 USER_CHANGES
  const userChangesData = prepareUserChangeset();
  if (userChangesData.changeset) { ... sendMessage(stateMessage); ... }
} else {
  // pending 期间，本地编辑不发，只是 setTimeout 3s 后再检查
  setTimeout(handleUserChanges, 3000);
}
```

**注意**：pending 期间用户输入不会丢失——编辑器仍然正常打字，只是 changeset 会累积在本地，等 pending 恢复后再一次性提交。

断开时还调用了 `setStateIdle()`，把 `committing = false`、清空 `stateMessage`。这里的语义是：
- 假设正在进行中的 commit（USER_CHANGES 已发出、尚未收到 ACCEPT_COMMIT）**可能成功也可能失败**（取决于断开时刻消息是否已到达服务端并落库）；
- 客户端不再等待这条 commit 的回执，而是把决定权交给重连恢复流程——如果该 commit 已落库，重连恢复阶段会通过 CLIENT_RECONNECT 中的 `author === pad.getUserId()` 识别并走 `acceptCommit()`；如果没落库，则该 changeset 仍然保留在本地编辑器的 user changes 中，pending 恢复后会被重新提交。

##### B. 服务端侧：CLIENT_RECONNECT 期间 sessionInfo.rev/time 的维护

[PadMessageHandler.ts#L1197-L1258](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1197-L1258)

重连分支的执行顺序是：

```
① socket.join(sessionInfo.padId)                            ← 重新进入房间
② sessionInfo.rev = message.client_rev                     ← 以客户端上报的 rev 为基线
③ 构建 revisionsNeeded = [client_rev+1, client_rev+2, ..., pad.head]
④ Promise.all 加载每条 rev 的 changeset/author/timestamp  ← 此时 sessionInfo.rev 不变
⑤ for (r of revisionsNeeded) { socket.emit(CLIENT_RECONNECT) }
                                                            ← 逐条发送，但 sessionInfo.rev 仍停留在 client_rev！
⑥ if (noChanges) { socket.emit(CLIENT_RECONNECT, noChanges:true) }
```

**关键点：重连分支内部没有递增 `sessionInfo.rev`，也没有设置 `sessionInfo.time`。** 与首次连接分支对比：

| 操作 | 首次连接（else 分支） | 重连恢复（if 分支） |
|------|----------------------|---------------------|
| 设置 `sessionInfo.rev` | ✅ 显式赋值 `sessionInfo.rev = headRev` | ⚠️ 只在开头赋值 `client_rev`，**不推进到 headRev** |
| 设置 `sessionInfo.time` | ✅ 用 `pad.getRevisionDate(headRev)` 或 `Date.now()` 初始化 | ❌ 完全不设置 |
| 推送缺失 revisions 后更新 rev | ✅ `updatePadClients` while 循环内每推进一条就 `sessioninfo.rev = r` 和 `sessioninfo.time = currentTime` | ❌ 分支内只 `socket.emit` 不落回写 sessionInfo |

**那么 sessionInfo.rev 何时追上 pad.head？**

答案是：**依赖后续的 `updatePadClients(pad)` 调用被动补齐。**

重连分支虽然没有显式调用 `updatePadClients`，但重连的 socket 在步骤 ① 已经 `socket.join(padId)` 进入了房间。只要在本分支执行完成后，房间内任意其他用户提交了一次新的编辑，`handleUserChanges` 末尾必然调用 `updatePadClients(pad)`：

```
exports.updatePadClients = async (pad) => {
  ...
  await Promise.all(roomSockets.map(async (socket) => {
    const sessioninfo = sessioninfos[socket.id];
    while (sessioninfo.rev < pad.getHeadRevisionNumber()) {
      // 逐条推送 NEW_CHANGES
      ...
      socket.emit('message', msg);
      sessioninfo.time = currentTime;   // ★ 补齐 sessionInfo.time
      sessioninfo.rev = r;              // ★ 逐步推进 sessionInfo.rev 到 head
    }
  }));
};
```

由于此时 `sessioninfo.rev` 仍停留在 `client_rev`（可能远小于 pad.head），第一次后续的 `updatePadClients` 会把 `CLIENT_RECONNECT` 已经发过的 revisions **再次以 NEW_CHANGES 的形式推给客户端**。这看起来像是重复推送，但客户端有防御：收到 `NEW_CHANGES` 时（[collab_client.ts#L209-L227](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/collab_client.ts#L209-L227)）：

```javascript
if (newRev !== (rev + 1)) {
  window.console.warn(`bad message revision on NEW_CHANGES: ${newRev} not ${rev + 1}`);
  return; // 直接忽略，不报错、不断开
}
```

因为客户端的本地 `rev` 已经在处理 CLIENT_RECONNECT 时被推进到了 head，所以这些 NEW_CHANGES 的 `newRev` 不满足 `newRev === rev+1`，会被警告后**安全丢弃**。副作用是：这次被动补齐虽然推送消息是"无效"的，但顺带把服务端的 `sessioninfo.rev` 和 `sessioninfo.time` 同步到了最新值——之后的推送就恢复正常了。

**潜在边界情况**：如果重连后很长时间没人编辑，pad.head 保持不变，那么直到下一次 `updatePadClients` 被触发前，服务端的 `sessionInfo.rev` 和 `sessionInfo.time` 会一直停留在旧值。这不会影响功能，但日志/统计中可能出现不一致。

##### C. 客户端侧：区分自身离线提交 vs 他人远端变更

[collab_client.ts#L242-L267](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/collab_client.ts#L242-L267)

```javascript
} else if (msg.type === 'CLIENT_RECONNECT') {
  serverMessageTaskQueue.enqueue(() => {
    if (msg.noChanges) {
      setIsPendingRevision(false);
      return;
    }
    const {headRev, newRev, changeset, author = '', apool} = msg;
    if (newRev !== (rev + 1)) {
      window.console.warn(`bad message revision on CLIENT_RECONNECT: ...`);
      return;
    }
    rev = newRev;          // 推进本地 rev 号
    if (author === pad.getUserId()) {
      // 这是客户端自己离线前提交的编辑（服务端已落库）
      acceptCommit();
    } else {
      // 这是其他用户在客户端离线期间的编辑
      editor.applyChangesToBase(changeset, author, apool);
    }
    if (newRev === headRev) {
      // 所有 pending revisions 都处理完了
      setIsPendingRevision(false);
    }
  });
}
```

**区分逻辑的核心只有一行**：`if (author === pad.getUserId())`。每条 revision 都带有产生它的 authorID，客户端把它与自己的 userId 比较：

- **匹配（自己的编辑）** → 调用 `acceptCommit()`：相当于把它当作一条晚到的 `ACCEPT_COMMIT` 回执，做：
  - `editor.applyPreparedChangesetToBase()`：把本地"已提交但未确认"的 changeset 正式合入基线
  - `stateMessage = null`：清空提交中的状态
  - `committing = false`：解除 commit 锁
  - `setStateIdle()`：安排 idle 回调
  - `callbacks.onInternalAction('commitAcceptedByServer')`
  - `callbacks.onConnectionTrouble('OK')`
  - `handleUserChanges()`：立刻检查是否有下一批要提交的编辑
- **不匹配（他人的编辑）** → 调用 `editor.applyChangesToBase(changeset, author, apool)`：像处理普通 NEW_CHANGES 一样，把远程变更合入本地编辑器的基线文本，并显示其他作者的颜色/高亮。

**为什么能这样区分？** 因为服务端保存每条 revision 时都把 authorID 写入了 `revision.meta.author`（`pad.appendRevision(changeset, author)`）。即使客户端断开期间有多个用户同时编辑，每条 revision 的作者归属都是精确可追溯的。

##### D. pending 状态何时恢复正常

`isPendingRevision` 设为 `true` 的触发点有三个（见上一节 A）：
1. `disconnect` 事件 → `socketReconnecting()`
2. `reconnect_attempt` → `socketReconnecting()`（兜底：如果第一次重连尝试发生在 disconnect 之前）
3. `error` 事件

`isPendingRevision` 设回 `false` 的触发点**只有一个**：在 CLIENT_RECONNECT 处理逻辑里：

| 情况 | 触发位置 | 条件 |
|------|----------|------|
| 没有缺失 revisions | [collab_client.ts#L246-L249](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/collab_client.ts#L246-L249) | `msg.noChanges === true`（客户端上报的 rev 已经等于 pad.head） |
| 处理完最后一条缺失 revision | [collab_client.ts#L263-L266](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/collab_client.ts#L263-L266) | `newRev === headRev`（刚处理的这条就是最新 revision） |

**恢复正常后做什么？**

`setIsPendingRevision(false)` 的实现（[collab_client.ts#L451-L461](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/collab_client.ts#L451-L461)）：

```javascript
const setIsPendingRevision = (value) => {
  const wasPending = isPendingRevision;
  isPendingRevision = value;
  if (wasPending && !value) {
    handleUserChanges();   // ★ 关键：立即触发一次提交检查
  }
};
```

`wasPending && !value` 这个条件保证只在"**从 true 跳变到 false**"时触发。触发后调用 `handleUserChanges()`，这是因为：
- 重连期间用户一直在打字，编辑器里已经累积了本地 changeset；
- pending 期间 `handleUserChanges()` 看到 `isPendingRevision === true` 就什么都不做，只 setTimeout 延期；
- 现在 pending 解除了，必须**立刻**检查是否有待提交的编辑，否则用户要等下次打字或下次 3 秒定时器到期才能同步。

注意：重连成功时 `socket.io.on('reconnect')` 也会调用 `pad.collabClient.setChannelState('CONNECTED')`（[pad.ts#L373-L379](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/pad.ts#L373-L379)），但此时 `isPendingRevision` 还是 true（CLIENT_RECONNECT 尚未到达），所以 `setChannelState('CONNECTED')` → `setUpSocket()` → `doDeferredActions()` 只会触发 deferred actions，不会解锁用户编辑提交——真正解锁是 `setIsPendingRevision(false)` 的那一步。

##### E. 重连状态机完整时序

```
  [正常协作]
    │  用户打字 → handleUserChanges()
    │  committing=true  →  发送 USER_CHANGES
    │  等待 ACCEPT_COMMIT  或  接收 NEW_CHANGES
    │
    ▼  (网络抖动)
  socket.io 'disconnect' 事件
    │
    ├─► socketReconnecting()
    │    ├─ setStateIdle()   → committing=false, stateMessage=null
    │    ├─ setIsPendingRevision(true)
    │    │   → 后续 handleUserChanges 调用都会 return（只延期不提交）
    │    └─ setChannelState('RECONNECTING') → UI 显示重连提示
    │
    ▼
  socket.io 自动重连尝试（最多 5 次，1s~5s 退避）
    │
    ▼
  socket.io 'reconnect' 事件
    │
    ├─► setChannelState('CONNECTED') → setUpSocket() → doDeferredActions()
    └─► sendClientReady(true)
          ├─ msg.reconnect = true
          ├─ msg.client_rev = collabClient.getCurrentRevisionNumber()  ← 本地 rev 号
          └─ emit('message', CLIENT_READY)
                │
                ▼
  服务端 handleClientReady reconnect 分支
    ├─ socket.join(padId)
    ├─ sessionInfo.rev = client_rev
    ├─ 加载 [client_rev+1, pad.head] 的所有 revisions
    └─ 逐条 socket.emit CLIENT_RECONNECT
          │   每条消息：headRev, newRev, changeset, apool, author, timestamp
          │
          ▼
  客户端 handleMessageFromServer → CLIENT_RECONNECT 分支
    │
    ├─ serverMessageTaskQueue.enqueue()  ← 保证串行处理
    │
    ▼  处理每条 CLIENT_RECONNECT（按 newRev 递增顺序）：
    │
    │  newRev !== rev+1 ? → warn 后 return 丢
    │  rev = newRev
    │  author === userId ?
    │    ├─ 是 → acceptCommit()        // 自己已落库的编辑
    │    │        └─ committing=false, stateMessage=null
    │    └─ 否 → applyChangesToBase()  // 他人编辑，合入本地基线
    │
    │  newRev === headRev ?
    │    └─ 是 → setIsPendingRevision(false)
    │              └─ handleUserChanges()  // ★ 立即提交 pending 期间的本地编辑
    │
    ▼
  恢复正常协作状态
```

---

#### 2.3.6 重连后首次编辑的"客户端-服务端状态不一致"与断言冲突

上一节描述了正常的重连恢复流程。但这一节揭示一个隐藏的**竞态条件/一致性问题**：客户端已经恢复了提交能力，但服务端的 `sessionInfo.rev` 和 `sessionInfo.time` 可能还停留在旧值，这与 `handleUserChanges` 中的关键断言之间存在冲突。

##### A. 问题根源：客户端状态前进一步，服务端状态原地踏步

重连恢复流程结束后：

| 状态变量 | 客户端 | 服务端 |
|----------|--------|--------|
| `rev` | 已推进到 `pad.head`（处理 CLIENT_RECONNECT 时逐条 `rev = newRev`） | **停留在 `client_rev`**（只在 reconnect 分支开头赋值一次 `sessionInfo.rev = message.client_rev`，发送 CLIENT_RECONNECT 时不更新） |
| `time`（时间戳基线） | 客户端本地无需维护这个变量（每条消息自带时间戳） | **未初始化**（重连分支完全没碰 `sessionInfo.time`） |
| 提交能力 | ✅ 可以提交（`isPendingRevision = false`，`committing = false`） | —— |
| 接收广播能力 | ✅ 已 join 房间，可以接收 NEW_CHANGES | —— |

**客户端为何能提交**：
- `setIsPendingRevision(false)` 解锁了 `handleUserChanges()` 的提交闸门（[collab_client.ts#L132-L152](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/static/js/collab_client.ts#L132-L152)）
- `committing = false`，没有正在进行的 commit 占用通道
- 客户端本地 `rev` 已经是最新值，可以构造正确的 `baseRev`

**服务端 `sessionInfo.rev` 为何没前进**：
- 重连分支（[PadMessageHandler.ts#L1197-L1258](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1197-L1258)）内部只有一次 `sessionInfo.rev = message.client_rev` 赋值
- 发送 CLIENT_RECONNECT 时，只 `socket.emit` 消息，**完全不更新 `sessionInfo.rev`**
- 重连分支既不调用 `updatePadClients(pad)`，也不在末尾手动同步 `sessionInfo.rev = pad.head`

对比首次连接分支的做法（[PadMessageHandler.ts#L1396-L1413](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1396-L1413)）：
```typescript
sessionInfo.rev = headRev;                   // ✅ 立即同步到最新
sessionInfo.time = await pad.getRevisionDate(headRev);  // ✅ 初始化时间戳
await exports.updatePadClients(pad);         // ✅ 还调用 updatePadClients 补发
```

重连分支相当于"只做了一半"——把消息推给客户端了，但服务端自己的状态没更新。

##### B. 关键断言：`assert.equal(thisSession.rev, r)`

在 `handleUserChanges` 函数中（[PadMessageHandler.ts#L976-L978](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L976-L978)）：

```typescript
// The client assumes that ACCEPT_COMMIT and NEW_CHANGES messages arrive in order. Make sure we
// have already sent any previous ACCEPT_COMMIT and NEW_CHANGES messages.
assert.equal(thisSession.rev, r);
```

这个断言的语义：**服务端记录的"该客户端已确认收到的 revision"必须等于 rebase 完成后到达的 revision 号**。

变量含义：
- `thisSession.rev`：服务端认为客户端已同步到的 revision
- `r`：从客户端上报的 `baseRev` 开始，经过 while 循环逐个 follow 追上 `pad.head` 后最终到达的 revision 号

断言的目的是确保消息顺序性——在给这个客户端发 `ACCEPT_COMMIT(newRev)` 之前，所有该客户端应该收到的 `NEW_CHANGES` 都已经发过了，且 `thisSession.rev` 正确反映了客户端的同步状态。

##### C. 冲突场景：客户端可提交 + 服务端 rev 旧值 = 断言失败

重现步骤（精确时序）：

```
时间轴：

T0  客户端正常协作，本地 rev = N，提交一条 USER_CHANGES（baseRev=N）
T1  网络断开，客户端没收到 ACCEPT_COMMIT
    socket.on('disconnect') → socketReconnecting()
      → setStateIdle()       // committing=false, stateMessage=null
      → setIsPendingRevision(true)
      → setChannelState('RECONNECTING')
T2  服务端收到并落库该提交，pad.head = N+1
    向房间广播 NEW_CHANGES(N+1)，但客户端已断线收不到
T3  另一客户端提交编辑，pad.head = N+2
    广播 NEW_CHANGES(N+2)，客户端也收不到
T4  socket.io 自动重连成功
    socket.io.on('reconnect') → sendClientReady(true)
      msg.reconnect = true
      msg.client_rev = getCurrentRevisionNumber() = N  (客户端本地 rev 还在 N)
T5  服务端 handleClientReady reconnect 分支:
      socket.join(padId)
      sessionInfo.rev = client_rev = N   ← ★ 只赋值一次
      加载 revisionsNeeded = [N+1, N+2]
      逐条发送 CLIENT_RECONNECT:
        CLIENT_RECONNECT(N+1, author=该客户端)
        CLIENT_RECONNECT(N+2, author=另一客户端)
      (发送过程中 sessionInfo.rev 仍为 N，不更新)
T6  客户端处理 CLIENT_RECONNECT:
      处理 N+1: author === userId → acceptCommit() → rev = N+1
      处理 N+2: author !== userId → applyChangesToBase() → rev = N+2
      newRev === headRev → setIsPendingRevision(false)
        → wasPending && !value → handleUserChanges()  ★ 立即触发提交
T7  客户端构造新的 USER_CHANGES，baseRev = N+2（本地最新 rev）
    emit('message', USER_CHANGES)
T8  服务端 handleUserChanges 处理该消息:
      const thisSession = sessioninfos[socket.id]
      const {baseRev=N+2, apool, changeset} = message
      ... 各种校验通过 ...
      let r = baseRev = N+2
      while (r < pad.head) {   // 假设 pad.head 仍为 N+2
        // 循环体不执行
      }
      const newRev = await pad.appendRevision(rebasedChangeset, author)
      assert.equal(thisSession.rev, r)  ← ★ assert N === N+2
                                          ← 💥 ASSERTION FAILED!
```

**断言失败触发的后果**（[PadMessageHandler.ts#L987-L991](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L987-L991)）：
```typescript
catch (err:any) {
  socket.emit('message', {disconnect: 'badChangeset'});
  stats.meter('failedChangesets').mark();
  messageLogger.warn(`Failed to apply USER_CHANGES from author ...`);
}
```
客户端被强制断开，显示"badChangeset"，但它的 changeset 其实是完全合法的——问题出在服务端状态不一致，而非客户端数据有误。

##### D. 何时能侥幸通过？

只有一种情况断言不会失败：**在 T6 和 T7 之间，有另一个客户端提交了编辑**，触发 `updatePadClients(pad)`：

```
T6.5  另一客户端提交编辑 → handleUserChanges → updatePadClients(pad)
        遍历 roomSockets，包括刚重连的 socket:
          sessioninfo = sessioninfos[socket.id]  // rev = N
          while (N < pad.head) {  // pad.head 现在可能是 N+3
            推送 NEW_CHANGES(N+1) → client 端 newRev !== rev+1 → warn 后丢弃
            sessioninfo.time = timestamp_N+1
            sessioninfo.rev = N+1
            推送 NEW_CHANGES(N+2) → 同样被丢弃
            sessioninfo.time = timestamp_N+2
            sessioninfo.rev = N+2
            推送 NEW_CHANGES(N+3) → 正常接收
            sessioninfo.time = timestamp_N+3
            sessioninfo.rev = N+3
          }
T7    原客户端提交 USER_CHANGES(baseRev=N+2)
        ...
        assert.equal(thisSession.rev, r)  // assert N+3 === N+3 → ✅ PASS
```

虽然断言通过了，但中间推送的 N+1、N+2 两条 NEW_CHANGES 对客户端来说是重复消息，被 `newRev !== rev+1` 检查丢弃——这是必要的防御性编程，但也暴露了服务端状态不同步的问题。

##### E. 与重传检查的交互

还有一个更隐蔽的交互：`handleUserChanges` 中的**重传检测**（[PadMessageHandler.ts#L931-L934](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L931-L934)）：

```typescript
if (canonicalCs === c && thisSession.author === authorId) {
  // Assume this is a retransmission of an already applied changeset.
  rebasedChangeset = identity(unpack(canonicalCs).oldLen);
}
```

如果客户端在断线前提交的那条编辑（T0 时刻）没收到 ACCEPT_COMMIT，断线期间服务端已经落库了，重连后客户端可能因为本地 editor 状态还保留着那个 changeset 而重传。此时：

- 客户端 `baseRev = N`（断线前的 rev）
- `canonicalCs` 与 revision N+1 的 changeset 完全匹配，`authorId` 也匹配
- 被判定为重传，`rebasedChangeset = identity(...)`（无净变化的空 changeset）
- while 循环还会继续推进 r 到 pad.head（例如 N+2）
- 最终 `assert.equal(thisSession.rev, r)` → `assert N === N+2` → **仍然失败！**

重传检测只是把 changeset 变成了空操作，但没有更新 `thisSession.rev`，所以断言还是失败。这意味着即使客户端只是"重新提交一个已经落库的编辑"，也会被断言错误地判定为 `badChangeset` 而踢下线。

##### F. sessionInfo.time 的连锁问题

`sessionInfo.time` 在重连分支中完全未初始化。在 `updatePadClients` 中（[PadMessageHandler.ts#L1041](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1041)）：

```typescript
timeDelta: currentTime - sessioninfo.time,
```

- 如果 `sessioninfo.time` 是 `undefined`，`timeDelta = NaN`
- 客户端收到 `timeDelta=NaN` 会导致广播和时间轴的时间显示异常
- 但推送第一条 NEW_CHANGES 后，`sessioninfo.time = currentTime` 会被赋值，后续就正常了

而在 `handleUserChanges` 中（[PadMessageHandler.ts#L985](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L985)）：

```typescript
if (newRev !== r) thisSession.time = await pad.getRevisionDate(newRev);
```

- 如果断言能通过（侥幸场景），`thisSession.time` 会被正确设置
- 如果断言失败，这个赋值永远执行不到

##### G. 问题总结

这是 Etherpad 代码中一个真实存在的一致性缺陷，根因在于：

1. **重连分支只同步客户端，不同步服务端自己的状态**——CLIENT_RECONNECT 发出去了，但 `sessionInfo.rev` 和 `sessionInfo.time` 没更新
2. **断言假设服务端状态始终与客户端同步**——`assert.equal(thisSession.rev, r)` 隐含假设 `sessionInfo.rev` 正确反映了客户端的同步进度
3. **两条路径实现不对称**——首次连接分支显式设置了 `sessionInfo.rev` 和 `sessionInfo.time`，还调用 `updatePadClients`，但重连分支全都省了

正确的修复应该是在重连分支末尾（发送完所有 CLIENT_RECONNECT 之后）同步服务端状态：

```typescript
// 重连分支末尾应补充：
sessionInfo.rev = pad.getHeadRevisionNumber();
sessionInfo.time = await pad.getRevisionDate(pad.getHeadRevisionNumber());
```

但目前代码里没有这两行，导致了本节描述的竞态条件。

### 2.4 重复作者检测（Stale Tab 踢下线）

[PadMessageHandler.ts#L1169-L1186](file:///d:/fz/0601-1/solo-dogfeeding/code/3-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1169-L1186)

对**未登录的匿名用户**（`req.session.user == null` 且非 embed 模式），遍历房间内现有 socket，若发现相同 authorID：
- 旧 socket 被踢：`otherSocket.emit('message', {disconnect: 'userdup'})` + `otherSocket.leave(padId)`
- `sessioninfos[otherSocket.id] = {}` 清空旧会话

已认证用户（SSO / basic auth）和 embed 模式不受此限制。

### 2.5 断开清理

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
