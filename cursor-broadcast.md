# Etherpad 多端光标与选区广播机制分析

## 概述

Etherpad 的"多端光标与选区"并非传统意义上的实时光标位置同步，而是通过**作者属性标记** + **用户信息广播** + **变更集同步**三层机制实现的协作感知：

1. **作者颜色标记**：每个用户输入的文本携带 `author` 属性，通过背景色直观展示谁在何处编辑
2. **用户信息同步**：通过 `USER_NEWINFO` / `USERINFO_UPDATE` 消息广播用户的名称、颜色等元数据
3. **变更集（Changeset）同步**：文本编辑操作以 Changeset 形式在客户端间同步，变更中内嵌作者信息

## 核心消息类型

| 消息类型 | 方向 | 用途 |
|---------|------|------|
| `CLIENT_READY` | 客户端 → 服务端 | 进入会话（首次连接 / 重连） |
| `CLIENT_VARS` | 服务端 → 客户端 | 初始会话数据（文本、属性池、历史作者） |
| `USER_NEWINFO` | 服务端 → 客户端 | 用户加入 / 用户信息更新通知 |
| `USERINFO_UPDATE` | 客户端 → 服务端 | 用户信息变更请求（改名、改色） |
| `USER_CHANGES` | 客户端 → 服务端 | 提交文本变更 |
| `NEW_CHANGES` | 服务端 → 客户端 | 广播其他用户的文本变更 |
| `ACCEPT_COMMIT` | 服务端 → 客户端 | 确认提交成功 |
| `CLIENT_RECONNECT` | 服务端 → 客户端 | 重连时补发缺失的变更 |
| `USER_LEAVE` | 服务端 → 客户端 | 用户离开通知 |

---

## 一、进入会话时点处理

### 1.1 客户端流程

**入口文件**：[pad.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/pad.ts)

客户端建立 Socket.io 连接后，发送 `CLIENT_READY` 消息：

```typescript
// pad.ts L298-L351
const sendClientReady = (isReconnect) => {
  const msg = {
    component: 'pad',
    type: 'CLIENT_READY',
    padId,
    userInfo: { colorId, name },
  };
  if (isReconnect) {
    msg.client_rev = pad.collabClient.getCurrentRevisionNumber();
    msg.reconnect = true;
  }
  socket.emit("message", msg);
};
```

**关键初始化顺序**：
1. Socket.io 连接建立 → `sendClientReady(false)`
2. 收到 `CLIENT_VARS` → 初始化 collabClient
3. collabClient 初始化 Ace 编辑器
4. 设置变更通知回调：`editor.setUserChangeNotificationCallback(handleUserChanges)`

### 1.2 服务端流程

**入口函数**：`handleClientReady` in [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1118-L1509)

#### 处理步骤：

1. **身份验证与作者信息加载**
   - 从 cookie / token 解析作者 ID
   - 加载或创建作者（姓名、颜色）

2. **Pad 加载**
   - 加载或创建 Pad 对象
   - 获取所有历史作者数据

3. **构建初始数据（clientVars）**
   - `initialAttributedText`：带属性的初始文本
   - `apool`：属性池（Attribute Pool）
   - `historicalAuthorData`：历史作者信息（姓名、颜色）
   - `rev`：当前版本号
   - `colorPalette`：颜色调色板

4. **加入房间**
   - `socket.join(sessionInfo.padId)`
   - 发送 `CLIENT_VARS` 给客户端

5. **双向用户通知**
   - **广播给其他用户**：新用户的 `USER_NEWINFO`
   - **通知新用户**：所有在线用户的 `USER_NEWINFO`

6. **触发 userJoin 钩子**

#### 关键代码片段：

```typescript
// 通知其他用户有新用户加入
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

// 向新用户发送所有在线用户信息
await Promise.all(_getRoomSockets(pad.id).map(async (roomSocket) => {
  // ... 获取每个用户的 authorInfo
  socket.emit('message', {
    type: 'COLLABROOM',
    data: {
      type: 'USER_NEWINFO',
      userInfo: { colorId, name, userId: authorId },
    },
  });
}));
```

### 1.3 客户端接收 USER_NEWINFO

**处理位置**：[collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/collab_client.ts#L268-L278)

```typescript
// collab_client.ts L268-L278
if (msg.type === 'USER_NEWINFO') {
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

**tellAceActiveAuthorInfo** → `editor.setAuthorInfo(userId, { bgcolor })` → 在编辑器中设置作者颜色样式

---

## 二、变更广播与去抖机制

### 2.1 客户端去抖策略

Etherpad 的去抖设计在**客户端**和**服务端**各有一层：

#### 第一层：ChangesetTracker 的微任务合并

**文件**：[changesettracker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/changesettracker.ts)

```typescript
// changesettracker.ts L48-L65
let changeCallbackTimeout = null;

const setChangeCallbackTimeout = () => {
  // 同一调用栈内多次调用只调度一次回调
  if (changeCallback && changeCallbackTimeout == null) {
    changeCallbackTimeout = scheduler.setTimeout(() => {
      try {
        changeCallback();
      } catch (pseudoError) {
        // as empty as my soul
      } finally {
        changeCallbackTimeout = null;
      }
    }, 0); // setTimeout(0) —— 微任务级合并
  }
};
```

**作用**：同一事件循环内的多次 DOM 变更合并为一次变更通知，避免高频触发。

#### 第二层：CollabClient 的提交延迟（commitDelay）+ 六段式状态门禁

**文件**：[collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/collab_client.ts#L49)

```typescript
let commitDelay = 500; // 默认 500ms 提交延迟
```

**核心逻辑**：`handleUserChanges` 函数——**六段式提前终止**

`handleUserChanges` 是客户端推送变更的唯一入口，进入后按顺序做 6 个门禁检查，每个检查不通过就**提前 return**，并根据情况安排下次重试。完整代码在 [collab_client.ts L95-L158](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/collab_client.ts#L95-L158)。

---

**门禁 1：输入法合成中（InInternationalComposition）**

```typescript
if (editor.getInInternationalComposition()) {
  // handleUserChanges() will be called again once composition ends
  // so there's no need to set up a future call before returning.
  return;
}
```

| 项目 | 说明 |
|------|------|
| **触发条件** | 用户正在使用中文/日文等输入法输入复合字符（compositionstart 已触发但 compositionend 尚未触发） |
| **后续处理** | **不设 setTimeout**——直接 return。因为 compositionend 事件触发后会重新走 DOM 变更检测流程，自动再次调用 `handleUserChanges` |
| **深层机制** | 输入法合成期间，浏览器 DOM 处于不稳定状态（中间态字符、下划线标记等）。如果此时提交 Changeset，会把合成的中间态（如拼音字母）也发送出去，造成文档污染 |
| **实现细节** | `inInternationalComposition` 是一个 **Promise** 而不是布尔值。见 [ace2_inner.ts L3589-L3597](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/ace2_inner.ts#L3589-L3597)，`compositionstart` 时创建 Promise，`compositionend` 时 resolve。这样既可以用 `if(x)` 判断状态，又能用 `await x` 等待合成结束 |

```typescript
// ace2_inner.ts L3391-L3597
let inInternationalComposition = null;

$(targetDoc.documentElement).on('compositionstart', () => {
  if (inInternationalComposition) return;
  inInternationalComposition = new Promise((resolve) => {
    $(targetDoc.documentElement).one('compositionend', () => {
      inInternationalComposition = null;
      resolve();
    });
  });
});
```

---

**门禁 2：连接未建立或正在连接中**

```typescript
const now = Date.now();
if ((!getSocket()) || channelState === 'CONNECTING') {
  if (channelState === 'CONNECTING' && (now - initialStartConnectTime) > 20000) {
    setChannelState('DISCONNECTED', 'initsocketfail');
  } else {
    setTimeout(handleUserChanges, 1000); // 1秒后重试
  }
  return;
}
```

| 项目 | 说明 |
|------|------|
| **触发条件** | Socket 对象不存在，或连接状态是 `CONNECTING`（握手尚未完成） |
| **超时判定** | `initialStartConnectTime` 在 `setUpSocket()` 中设置（见 L175），如果连接过程超过 **20 秒**，判定为网络故障，进入 `DISCONNECTED` 状态 |
| **重试策略** | 未超时的话，**每 1000ms** 重试一次 `handleUserChanges`。这是一个相对宽松的轮询周期，因为连接建立本身是异步事件驱动的，轮询只是兜底 |
| **用户体验** | 此期间用户可以继续在本地输入，DOM 变更会被 ChangesetTracker 正常捕获，只是不会提交到服务端 |

---

**门禁 3：提交过慢（上一次提交长期未确认）**

```typescript
if (committing) {
  if (now - lastCommitTime > 20000) {
    setChannelState('DISCONNECTED', 'slowcommit');
  } else if (now - lastCommitTime > 5000) {
    callbacks.onConnectionTrouble('SLOW'); // 触发 UI 提示"连接缓慢"
  } else {
    setTimeout(handleUserChanges, 3000); // 3秒后重试
  }
  return;
}
```

| 项目 | 说明 |
|------|------|
| **触发条件** | `committing = true`（上一次 `USER_CHANGES` 已发送，但尚未收到 `ACCEPT_COMMIT` 响应） |
| **三级判定** |<ul><li>**< 5 秒**：正常情况，3 秒后再查一次（看是否收到确认）</li><li>**5~20 秒**：触发 `onConnectionTrouble('SLOW')`，UI 显示"网络缓慢"提示</li><li>**> 20 秒**：判定为提交丢失或服务端故障，进入 `DISCONNECTED` 状态，由 Socket.io 自动重连机制接手</li></ul> |
| **commit 期间的本地变更** | 用户继续输入会被 ChangesetTracker 累积到 `userChangeset`（和上一次提交的变更自动 compose 合并），等 `acceptCommit()` 被调用后会一并提交 |
| **提交确认的作用链** | `ACCEPT_COMMIT` → `acceptCommit()` → `setStateIdle()`（`committing = false`） → 立即调用 `handleUserChanges()` 提交累积的新变更 |

---

**门禁 4：去抖窗口内（距离上次提交不足 500ms）**

```typescript
const earliestCommit = lastCommitTime + commitDelay;
if (now < earliestCommit) {
  setTimeout(handleUserChanges, earliestCommit - now);
  return;
}
```

| 项目 | 说明 |
|------|------|
| **触发条件** | `now < lastCommitTime + 500ms` —— 距离上次成功提交还不到 500ms |
| **精确调度** | 不是固定延迟，而是精确计算**还需要等多少毫秒**才到最早提交时间（`earliestCommit - now`），到期立刻执行 |
| **设计考量** |<ul><li>500ms 是人眼感知"实时"的心理阈值</li><li>合并快速连续输入（比如打字速度 400 字符/分钟，500ms 内会有 3~4 个按键事件）</li><li>减少服务端处理压力和网络带宽</li></ul> |
| **与 committing 的配合** | 注意：这个门禁在 `committing` 门禁**之后**。也就是说，即使 500ms 到了，如果上一次提交还没确认，也不会触发新的提交。必须同时满足"committing = false"且"距上次提交 > 500ms" |

---

**门禁 5：有待处理的服务端版本（isPendingRevision）**

```typescript
let sentMessage = false;
if (!isPendingRevision) {
  const userChangesData = prepareUserChangeset();
  if (userChangesData.changeset) {
    // ... 真正执行提交 ...
    sendMessage(stateMessage);
    sentMessage = true;
  }
} else {
  setTimeout(handleUserChanges, 3000); // 3秒后重试
}
```

| 项目 | 说明 |
|------|------|
| **触发条件** | `isPendingRevision = true`——正在接收服务端补发的批量变更（断线重连时），或有 NEW_CHANGES 消息队列待处理 |
| **为什么必须拦截** | 如果在服务端版本尚未全部补齐的情况下提交本地变更，会导致 `baseRev` 不正确，服务端 Rebase 时出现版本错乱（Changeset 的偏移计算会错位） |
| **恢复时机** | 见下方「断线后本地新输入拦截与恢复」一节的详细分析 |
| **重试周期** | 3 秒一次，周期较长，因为重连补版本通常是秒级完成的，这个轮询只是兜底（正常情况下是由 `setIsPendingRevision(false)` 主动触发提交） |

---

**门禁 6：无实际变更（userChangesData.changeset 为空）**

这不是显式的 if-return，而是隐含在流程中。当 `prepareUserChangeset()` 返回的 `changeset` 为空（或为恒等 Changeset）时，不会执行 `sendMessage`，`sentMessage` 保持为 `false`，后面也就不会安排 3 秒后的超时检测。

**六个门禁之后的正常提交流程**：

```typescript
lastCommitTime = now;        // 记录提交时间（用于门禁4的去抖计算）
committing = true;           // 设置提交中标志（门禁3的判断条件）
stateMessage = { ... };      // 保存提交内容（用于断线后 getMissedChanges 重放）
sendMessage(stateMessage);   // 实际发送 USER_CHANGES
sentMessage = true;
// ...
if (sentMessage) {
  setTimeout(handleUserChanges, 3000); // 3秒后再次检查（用于超时检测，见门禁3）
}
```

**去抖效果总结**：
- 连续输入时，每 500ms 才提交一次 Changeset
- 提交期间（`committing = true`）的新变更会累积，等待下一次提交
- 减少网络请求次数，合并多次小变更

#### 第三层：服务端速率限制

**文件**：[PadMessageHandler.ts](file:///d:/fz/0601-2\solo-dogfeeding\code\8-etherpad-lite\src\node\handler\PadMessageHandler.ts#L78)

```typescript
rateLimiter = new RateLimiterMemory(settings.commitRateLimiting);
```

使用 `rate-limiter-flexible` 库限制每个客户端的提交频率，防止恶意刷屏。

### 2.2 服务端变更处理与广播

**入口函数**：`handleUserChanges` in [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/node/handler/PadMessageHandler.ts#L819-L1006)

#### 处理流程：

1. **验证 Changeset**
   - 语法校验：`checkRep(changeset)`
   - 作者属性校验：确保插入的文本作者是当前用户

2. **Rebase（变基）**
   - 客户端提交的是基于 `baseRev` 的变更
   - 服务端可能已有更新的版本
   - 使用 `follow()` 函数将变更变基到最新版本

3. **追加新版本**
   - `pad.appendRevision(rebasedChangeset, author)`
   - 版本号 +1

4. **确认提交 + 预更新提交者 session.rev**
   - 向提交者发送 `ACCEPT_COMMIT` 消息
   - **关键**：在调用 `updatePadClients` 之前，先把提交者的 `session.rev` 更新为新版本号
   - 这是让提交者本人不被重复推送的核心技巧

5. **广播给所有客户端**
   - 调用 `updatePadClients(pad)`
   - 遍历所有在线用户（包括提交者本人），根据 `session.rev` 判断是否需要推送

---

#### 关键机制：提交者本人为何不被重复推送 NEW_CHANGES

**完整时序**（[PadMessageHandler.ts L977-L997](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/node/handler/PadMessageHandler.ts#L977-L997)）：

```typescript
// 第 1 步：追加新版本到 Pad（假设从 rev=42 升到 rev=43）
const newRev = await pad.appendRevision(rebasedChangeset, thisSession.author);
// 此时：pad.getHeadRevisionNumber() = 43
// 但：thisSession.rev 仍然是 42（提交者自己的旧版本号）

// 第 2 步：断言确认——必须保证提交者的 rev 还是原来的 r（=42）
assert.equal(thisSession.rev, r);

// 第 3 步：向提交者发送 ACCEPT_COMMIT（确认消息）
socket.emit('message', {type: 'COLLABROOM', data: {type: 'ACCEPT_COMMIT', newRev}});

// 第 4 步：★★★ 关键操作——在广播前，先把提交者的 rev 更新！★★★
thisSession.rev = newRev;  // 从 42 改成 43（注意：pad 的 head 已经是 43）
if (newRev !== r) thisSession.time = await pad.getRevisionDate(newRev);

// 第 5 步：调用 updatePadClients，遍历所有在线用户（包括提交者本人）
await exports.updatePadClients(pad);
```

**`_getRoomSockets` 的工作方式**（[PadMessageHandler.ts L1670-L1682](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1670-L1682)）：

```typescript
const _getRoomSockets = (padID) => {
  const ns = socketio.sockets;
  const room = ns.adapter.rooms?.get(padID);
  if (!room) return [];
  // 直接从 Socket.io 房间适配器取所有 socket ID，包括提交者本人
  return Array.from(room)
    .map(socketId => ns.sockets.get(socketId))
    .filter(socket => socket);
};
```

**`updatePadClients` 中的跳过逻辑**（[PadMessageHandler.ts L1030-L1064](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1030-L1064)）：

```typescript
exports.updatePadClients = async (pad) => {
  const roomSockets = _getRoomSockets(pad.id);
  // 包括提交者本人的 socket，因为他也在房间里

  const revCache = {}; // 缓存已查询的版本，避免重复读数据库

  await Promise.all(roomSockets.map(async (socket) => {
    const sessioninfo = sessioninfos[socket.id];
    if (sessioninfo == null) return;

    // ★★ while 循环条件：sessioninfo.rev < pad.getHeadRevisionNumber() ★★
    //
    // 对于提交者（提交前 sessioninfo.rev = 42）：
    //   因为第 4 步已经执行 thisSession.rev = newRev = 43
    //   而 pad.getHeadRevisionNumber() = 43
    //   所以：43 < 43 = false → while 循环体一次都不执行！
    //
    // 对于其他用户（sessioninfo.rev = 42）：
    //   42 < 43 = true → 进入循环，发送 NEW_CHANGES rev=43
    //   更新 sessioninfo.rev = 43
    //   循环条件再次判断：43 < 43 = false → 退出

    while (sessioninfo.rev < pad.getHeadRevisionNumber()) {
      const r = sessioninfo.rev + 1;
      let revision = revCache[r];
      if (!revision) {
        revision = await pad.getRevision(r);
        revCache[r] = revision;
      }

      const forWire = prepareForWire(revision.changeset, pad.pool);
      const msg = {
        type: 'COLLABROOM',
        data: {
          type: 'NEW_CHANGES',
          newRev: r,
          changeset: forWire.translated,
          apool: forWire.pool,
          author: revision.meta.author,
          currentTime: revision.meta.timestamp,
          timeDelta: revision.meta.timestamp - sessioninfo.time,
        },
      };
      socket.emit('message', msg);
      sessioninfo.time = revision.meta.timestamp;
      sessioninfo.rev = r;
    }
  }));
};
```

**设计要点总结**：

| 设计选择 | 为什么这样做 |
|---------|------------|
| **不用 `socket.broadcast` 排除提交者** | `socket.broadcast` 只适合单条消息。`updatePadClients` 是通用函数，要处理**批量补发多个缺失版本**的场景，必须用 while 循环逐个推 |
| **先更新 session.rev，再调用通用函数** | 提交者的 `session.rev` 先被更新为 newRev，进入 `while` 判断时刚好等于 head，自然跳过。这是"无侵入式排除"——同一个函数对所有用户逻辑完全一致，只是内部状态不同导致行为不同 |
| **`revCache` 缓存优化** | 多个客户端同时缺失版本时，只从数据库读一次 |
| **`thisSession.time` 更新** | `timeDelta` 用于客户端计算"这条变更是多久前发生的"，必须在推送前更新为新版本的时间戳 |

**关于顺序保证的注释**（[PadMessageHandler.ts L987-L989](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/node/handler/PadMessageHandler.ts#L987-L989)）：

```typescript
// The client assumes that ACCEPT_COMMIT and NEW_CHANGES messages arrive in order.
assert.equal(thisSession.rev, r);
```

单 socket 上的消息按发送顺序排队，客户端会先收到 `ACCEPT_COMMIT`，再收到其他用户的 `NEW_CHANGES`。

### 2.3 客户端接收 NEW_CHANGES

**处理位置**：[collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/collab_client.ts#L209-L227)

```typescript
if (msg.type === 'NEW_CHANGES') {
  serverMessageTaskQueue.enqueue(async () => {
    await editor.getInInternationalComposition(); // 输入法合成中则等待
    rev = newRev;
    editor.applyChangesToBase(changeset, author, apool);
  });
}
```

**任务队列（serverMessageTaskQueue）**：
- 保证服务端消息按顺序处理
- 避免并发修改导致的状态不一致

---

## 三、断线恢复时点处理

### 3.1 断线检测

**客户端检测**：
- Socket.io 内置心跳检测
- `disconnect` 事件触发

**连接状态**：
```typescript
// collab_client.ts
const channelState = 'CONNECTING'; // CONNECTING | CONNECTED | RECONNECTING | DISCONNECTED
```

**状态切换**：
```typescript
// pad.ts L390-L396
socket.on('disconnect', (reason) => {
  socketReconnecting(); // 设置为 RECONNECTING
});

socket.io.on('reconnect_attempt', socketReconnecting);
```

### 3.2 重连流程

#### 客户端重连

Socket.io 自动重连（默认 5 次，指数退避 1s~5s）：

```typescript
// pad.ts L361-L367
socket = socketio.connect(exports.baseURL, '/', {
  reconnectionAttempts: 5,
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
});
```

重连成功后发送 `CLIENT_READY` + `reconnect: true`：

```typescript
// pad.ts L373-L379
socket.io.on('reconnect', () => {
  pad.collabClient.setChannelState('CONNECTED');
  sendClientReady(receivedClientVars); // 带上 reconnect 标志
});
```

#### 服务端重连处理

**处理位置**：`handleClientReady` 中的 `message.reconnect` 分支 in [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1208-L1269)

```typescript
if (message.reconnect) {
  socket.join(sessionInfo.padId);
  sessionInfo.rev = message.client_rev;
  
  // 计算缺失的版本范围
  let startNum = message.client_rev + 1;
  let endNum = pad.getHeadRevisionNumber() + 1;
  
  // 逐个补发 CLIENT_RECONNECT 消息
  for (const r of revisionsNeeded) {
    socket.emit('message', {
      type: 'COLLABROOM',
      data: {
        type: 'CLIENT_RECONNECT',
        headRev: pad.getHeadRevisionNumber(),
        newRev: r,
        changeset: forWire.translated,
        apool: forWire.pool,
        author: changesets[r].author,
      },
    });
  }
}
```

### 3.3 客户端接收 CLIENT_RECONNECT

**处理位置**：[collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/collab_client.ts#L242-L267)

```typescript
if (msg.type === 'CLIENT_RECONNECT') {
  serverMessageTaskQueue.enqueue(() => {
    if (msg.noChanges) {
      setIsPendingRevision(false);
      return;
    }
    
    rev = newRev;
    if (author === pad.getUserId()) {
      acceptCommit(); // 是自己的提交，直接确认
    } else {
      editor.applyChangesToBase(changeset, author, apool);
    }
    
    // 全部补发完成后恢复正常状态
    if (newRev === headRev) {
      setIsPendingRevision(false);
    }
  });
}
```

### 3.4 本地输入拦截机制（isPendingRevision 的完整作用链）

断线→重连期间，用户可以继续在本地输入，但**提交动作会被 `isPendingRevision` 标志拦截**，直到服务端补发的所有变更都应用完成。这个机制的核心不是"阻止用户输入"，而是"阻止在版本不一致的情况下提交"。

---

#### 拦截点：`handleUserChanges` 门禁 5

回看上文的"六段式门禁"，**门禁 5** 是拦截提交的关键：

```typescript
// collab_client.ts L134-L152
let sentMessage = false;
if (!isPendingRevision) {          // ★ 只有 isPendingRevision = false 才允许提交
  const userChangesData = prepareUserChangeset();
  if (userChangesData.changeset) {
    // ... 真正执行提交 ...
    sendMessage(stateMessage);
    sentMessage = true;
  }
} else {
  setTimeout(handleUserChanges, 3000); // 3秒后重试（兜底用，见下文）
}
```

**重要**：`isPendingRevision` 只拦截**提交到服务端**的动作，不拦截**本地 DOM 输入**和**本地 Changeset 累积**。用户输入的每一个字符仍然会：
1. 正常渲染在浏览器 DOM 上
2. 被 ChangesetTracker 的 `composeUserChangeset` 捕获并累积到 `userChangeset` 变量
3. 被 `setChangeCallbackTimeout` 调度回调 → 触发 `handleUserChanges`（但走到门禁 5 就 return）

这确保了用户体验的连续性——本地输入不受任何影响，只是暂时不上传。

---

#### isPendingRevision 的置位（true）与复位（false）时机

**⚠️ 重要修正**：之前的代码是错误的。`setChannelState` 函数体内**根本没有**置位逻辑！实际的 `setChannelState` 实现非常简单（[collab_client.ts L374-L379](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/collab_client.ts#L374-L379)）：

```typescript
// collab_client.ts L374-L379 —— 真实的 setChannelState，没有置位逻辑！
const setChannelState = (newChannelState, moreInfo) => {
  if (newChannelState !== channelState) {
    channelState = newChannelState;
    callbacks.onChannelStateChange(channelState, moreInfo);
  }
};
```

**置位为 true 的真实时机**：

`isPendingRevision` 被设为 true，**不是在 `setChannelState` 内部**，而是在 **pad.ts** 的三个事件处理函数中**显式调用**的。

---

#### 完整事件绑定与调用链（pad.ts）

**事件绑定**（都在 [pad.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/pad.ts)）：

```
Socket.io 事件 ──→ 处理函数
──────────────────────────────────────
1. socket.on('disconnect')        ──→ socketReconnecting()
2. socket.io.on('reconnect_attempt') ──→ socketReconnecting（直接绑定函数）
3. socket.io.on('reconnect')      ──→ setChannelState('CONNECTED') + sendClientReady(true)
4. socket.io.on('reconnect_failed') ──→ setChannelState('DISCONNECTED') ★ 终态
5. socket.on('error')             ──→ setStateIdle + setIsPendingRevision(true)
6. socket.once('connect')         ──→ sendClientReady(false)
```

**事件触发顺序**（从断线到重连成功/失败的完整生命周期）：

```
disconnect  ← 连接刚断，立即触发
     ↓
reconnect_attempt  ← 第 1 次重连尝试（1s 后）
     ↓  （失败）
reconnect_attempt  ← 第 2 次重连尝试（2s 后）
     ↓  （失败）
     ...
     ↓
reconnect_attempt  ← 第 5 次重连尝试（5s 后）
     ↓  （失败）
reconnect_failed  ← 5 次全部失败，进入终态
     ↓
（只能靠用户手动点"重新连接"按钮，或自动重连定时器触发整页重载）
```

**Socket.io 重连参数**（[pad.ts L361-L367](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/pad.ts#L361-L367)）：

```typescript
socket = pad.socket = socketio.connect(exports.baseURL, '/', {
  query: {padId},
  reconnectionAttempts: 5,      // 最多 5 次重连尝试
  reconnection: true,           // 启用自动重连
  reconnectionDelay: 1000,      // 首次延迟 1000ms
  reconnectionDelayMax: 5000,   // 最大延迟 5000ms（指数退避的上限）
});
```

退避策略：第 n 次尝试的延迟 = `min(reconnectionDelay * 2^(n-1), reconnectionDelayMax)`
- 第 1 次：1000ms
- 第 2 次：2000ms
- 第 3 次：4000ms
- 第 4 次：5000ms（达到上限）
- 第 5 次：5000ms
- 5 次都失败 → `reconnect_failed`

**`socketReconnecting` 函数**（[pad.ts L381-L388](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/pad.ts#L381-L388)）：

```typescript
const socketReconnecting = () => {
  if (pad.collabClient != null) {
    // ★ 第 1 步：重置提交状态（把 committing 设为 false）
    pad.collabClient.setStateIdle();
    // ★ 第 2 步：置位拦截标志（门禁 5 开始拦截提交）
    pad.collabClient.setIsPendingRevision(true);
    // ★ 第 3 步：更新连接状态（UI 显示"正在重连"）
    pad.collabClient.setChannelState('RECONNECTING');
  }
};
```

**三步调用顺序的设计考量**：

| 顺序 | 调用 | 作用 |
|------|------|------|
| 1 | `setStateIdle()` | 重置 `committing = false`，清除可能残留的提交中状态。如果断线前恰好有一个 USER_CHANGES 发出去但还没收到 ACCEPT_COMMIT，这个状态就不对了，必须重置。同时执行 `idleFuncs` 队列中等待的函数 |
| 2 | `setIsPendingRevision(true)` | 核心拦截标志置位。这个必须在 `setChannelState` 之前，因为状态切换回调可能会触发一些检查 |
| 3 | `setChannelState('RECONNECTING')` | 更新 `channelState` 并触发 UI 回调（显示"连接中断，正在重连…"） |

---

**`error` 事件的特殊处理**（[pad.ts L437-L446](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/pad.ts#L437-L446)）：

```typescript
socket.on('error', (error) => {
  if (pad.collabClient != null) {
    pad.collabClient.setStateIdle();
    pad.collabClient.setIsPendingRevision(true);
    // ★ 注意：这里没有调用 setChannelState('RECONNECTING')！
  }
});
```

**为什么 error 事件不调用 setChannelState？**
- `error` 只是连接异常（如偶发的网络错误），不一定会断开
- 如果改变 `channelState`，UI 会频繁闪烁"正在重连"提示
- 保守起见仍然拦截提交（防止版本不一致），但不打扰用户

---

**`reconnect` 事件的处理**（[pad.ts L373-L379](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/pad.ts#L373-L379)）：

```typescript
socket.io.on('reconnect', () => {
  if (pad.collabClient != null) {
    pad.collabClient.setChannelState('CONNECTED');
  }
  sendClientReady(receivedClientVars);  // 发送 CLIENT_READY + reconnect: true
});
```

**关键点**：`setChannelState('CONNECTED')` 时 **不会** 自动把 `isPendingRevision` 设为 false。这是刻意的设计——因为服务端还没开始补发变更，此时如果允许提交，`baseRev` 还是断线前的旧版本，会造成版本错乱。

`setIsPendingRevision` 函数的注释（[collab_client.ts L454-L457](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/collab_client.ts#L454-L457)）明确说明了这一点：

```typescript
// After reconnect, once all pending revisions from the server have been applied
// (isPendingRevision transitions from true to false), flush any unsent local changes
// that were queued while disconnected. The handleUserChanges() call in setChannelState
// (CONNECTED) is not sufficient because isPendingRevision is still true at that point.
```

翻译：**重连后，必须等服务端补发的所有版本都应用完（isPendingRevision 从 true 变 false），才能提交本地累积的变更。在 setChannelState('CONNECTED') 时调用 handleUserChanges() 是不够的，因为此时 isPendingRevision 还是 true，会被门禁 5 拦住。**

---

**`setStateIdle` 的完整作用**（[collab_client.ts L445-L479](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/collab_client.ts#L445-L479)）：

```typescript
const setStateIdle = () => {
  committing = false;  // 重置提交中标志
  callbacks.onInternalAction('newlyIdle');
  schedulePerhapsCallIdleFuncs();  // 执行 idleFuncs 队列
};

const schedulePerhapsCallIdleFuncs = () => {
  setTimeout(() => {
    if (!committing) {
      while (idleFuncs.length > 0) {
        const f = idleFuncs.shift();
        f();  // 执行所有等待"不处于提交中"状态的函数
      }
    }
  }, 0);
};
```

`idleFuncs` 队列用于存放"必须等当前提交完成后才能执行"的操作（比如某些需要修改文档但又不能和提交冲突的插件操作）。断线时必须把这些函数执行掉，否则它们会一直等一个永远不会到来的 `acceptCommit`。

---

**完整置位时机总结**：

只要 Socket.io 出现以下任何一种情况，`isPendingRevision` 就被设为 true：
1. `disconnect` 事件（连接断开）
2. `reconnect_attempt` 事件（开始尝试重连）
3. `error` 事件（连接异常）

这是一个极其保守的策略——**宁可暂时不提交，也不在版本可能不一致的情况下提交**。

---

#### 复位为 false 的两个路径

**路径 A：服务端补发完成（正常路径）**

服务端逐个发送 `CLIENT_RECONNECT` 消息，每条消息携带 `headRev`（总版本数）和 `newRev`（当前这条消息对应的版本号）。客户端每接收一条就检查：

```typescript
// collab_client.ts L242-L267
if (msg.type === 'CLIENT_RECONNECT') {
  serverMessageTaskQueue.enqueue(() => {
    if (msg.noChanges) {
      setIsPendingRevision(false);  // ★ 如果没有任何变更需要补，直接恢复
      return;
    }
    const {headRev, newRev, changeset, author = '', apool} = msg;
    // ... 应用变更或 acceptCommit ...
    rev = newRev;
    if (author === pad.getUserId()) {
      acceptCommit();  // 自己的旧提交，直接确认
    } else {
      editor.applyChangesToBase(changeset, author, apool);  // 别人的变更，应用到文档
    }
    if (newRev === headRev) {
      // ★★★ 当收到的版本等于 headRev 时，说明补发完毕
      setIsPendingRevision(false);
    }
  });
}
```

**路径 B：没有缺失的变更**

如果断线期间恰好没有任何其他用户修改文档，服务端发送的第一条 `CLIENT_RECONNECT` 就会带 `noChanges: true`，客户端直接恢复：

```typescript
// collab_client.ts L246-L249
if (msg.noChanges) {
  setIsPendingRevision(false);  // 无变更，直接退出拦截模式
  return;
}
```

---

#### 复位后的自动提交流程（`setIsPendingRevision` 的副作用）

`setIsPendingRevision` 不是一个简单的 setter，它带有状态转换检测：

```typescript
// collab_client.ts L451-L461
const setIsPendingRevision = (value) => {
  const wasPending = isPendingRevision;  // 记录转换前的状态
  isPendingRevision = value;             // 执行赋值

  // ★ 关键：只有当状态从 true → false（即"等待中→恢复正常"的转换边沿）
  // 才会触发 handleUserChanges，立即提交本地累积的变更
  if (wasPending && !value) {
    handleUserChanges();
  }
};
```

这个**边沿触发**设计非常重要：
- `false → true`（进入等待）：不需要做任何事，因为下一次 `handleUserChanges` 自然会在门禁 5 被拦住
- `true → false`（恢复正常）：必须**立即**调用 `handleUserChanges`，否则就只能等 3 秒的兜底轮询（门禁 5 的 else 分支），用户会感知到明显的延迟

---

#### 兜底轮询机制

在 `isPendingRevision = true` 期间，门禁 5 的 else 分支设置了 3 秒一次的轮询：

```typescript
} else {
  setTimeout(handleUserChanges, 3000);  // 3秒后重试
}
```

这个轮询存在的意义是**防止 `setIsPendingRevision(false)` 因异常未被调用**（例如网络异常导致最后一条 `CLIENT_RECONNECT` 丢失）。如果一切正常，这个轮询永远不会触发到实际的提交——因为边沿触发会先一步调用 `handleUserChanges`。轮询只是双保险。

---

#### 完整状态机转换图（`isPendingRevision` + `channelState` 双状态协同）

```
                        ┌──────────────────────────────────────────────────────────────┐
                        │  正常连接状态（isPendingRevision=false, channelState=CONNECTED） │
                        │  - 用户输入 → handleUserChanges → 门禁5通过 → 提交成功       │
                        │  - 收到 NEW_CHANGES → 正常应用                          │
                        └──────────────┬───────────────────────────────────────────┘
                                       │
                                       ▼  Socket.io disconnect / error / reconnect_attempt
                        ┌──────────────────────────────────────────────────────────────┐
                        │  断线/重连状态（isPendingRevision=true, channelState=RECONNECTING） │
                        │  ★ pad.ts 显式调用 setIsPendingRevision(true)               │
                        │  - 用户输入 → DOM正常渲染 → ChangesetTracker 累积             │
                        │  - handleUserChanges → 门禁5拦截 → 3s兜底轮询              │
                        │  - 收到 NEW_CHANGES → 继续拦截（通过 serverMessageTaskQueue） │
                        └──────────────┬───────────────────────────────────────────┘
                                       │
                                       ▼  收到最后一条 CLIENT_RECONNECT
                                       │  (newRev === headRev) or (noChanges: true)
                        ┌──────────────────────────────────────────────────────────────┐
                        │  恢复转换中（isPendingRevision 从 true→false 的边沿瞬间）     │
                        │  ★ 仅在这个边沿，setIsPendingRevision 自动调用 handleUserChanges() │
                        │  - 把累积了整个断线期间的 userChangeset 整体合并提交               │
                        └──────────────┬───────────────────────────────────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────────────────────────────────────┐
                        │  恢复正常（isPendingRevision=false, channelState=CONNECTED） │
                        │  回到正常连接状态，继续正常协作                              │
                        └──────────────────────────────────────────────────────────────┘
```

**状态转换的关键不变量**（任何时刻必须满足）：
- `isPendingRevision=true` → `handleUserChanges` 门禁5 **一定** 拦截提交
- `isPendingRevision=false` → `handleUserChanges` 门禁5 **一定** 允许提交（其他门禁可能拦截）
- `isPendingRevision` 从 true→false 的 **边沿** → **自动触发** `handleUserChanges` 一次
- `isPendingRevision` 从 false→true 的 **边沿** → **不触发** 任何操作

**为什么要让 pad.ts 显式调用 setIsPendingRevision，而不是在 setChannelState 内部自动设置？**

这是一个重要的设计选择：
1. **关注点分离**：`collab_client.ts` 只负责协作逻辑（提交、去抖、版本管理），不直接监听 Socket.io 事件
2. **灵活控制**：`pad.ts` 作为上层协调者，根据不同的 Socket.io 事件（disconnect / reconnect_attempt / error）决定何时进入拦截模式
3. **避免误触发**：如果在 setChannelState 内部自动设置，可能会在正常连接建立时（如初次握手过程中）误拦截
4. **显式优于隐式**：调用点明确写在事件处理函数中，代码可读性更好

---

#### 重连恢复完整时序示例

假设用户 A 在编辑过程中断线 10 秒，期间输入了 3 个字符 "abc"，然后重连成功：

| 时间 | 事件 | 状态 | 操作 |
|------|------|------|------|
| T=0 | 用户正常输入 "h" | `isPendingRevision=false` | 500ms 后提交 rev=100 → ACCEPT_COMMIT → rev=100 |
| T=1 | 网络波动，Socket.io 触发 disconnect | `isPendingRevision=true`（pad.ts 调用 setIsPendingRevision(true)） | 进入拦截模式 |
| T=2 | 用户输入 "a" | `isPendingRevision=true` | DOM 渲染 "a"，ChangesetTracker 累积，handleUserChanges 被门禁5 拦截 |
| T=5 | 用户输入 "b" | `isPendingRevision=true` | DOM 渲染 "b"，累积到 userChangeset |
| T=10 | Socket.io 重连成功，发送 CLIENT_READY + reconnect=true | `isPendingRevision=true` | 服务端开始补发 CLIENT_RECONNECT |
| T=10.1 | 收到 CLIENT_RECONNECT rev=101（别人的变更） | `isPendingRevision=true` | 应用变更，newRev=101, headRev=102 → 继续等待 |
| T=10.2 | 收到 CLIENT_RECONNECT rev=102（别人的变更） | `isPendingRevision=true` | 应用变更，newRev=102, headRev=102 → ★ 调用 setIsPendingRevision(false) |
| T=10.2 | 边沿触发（true→false） | `isPendingRevision=false` | 自动调用 handleUserChanges，门禁5 通过 |
| T=10.2 | 提交 "abc" 到服务端 | `isPendingRevision=false` | 发送 USER_CHANGES baseRev=102 |
| T=10.3 | 收到 ACCEPT_COMMIT rev=103 | `isPendingRevision=false` | 确认成功，rev=103 |

---

### 3.5 本地变更保存与重放（ChangesetTracker 内部的三层状态机）

断线期间用户可能继续输入，这些本地变更需要在重连后重新提交。ChangesetTracker 内部维护了三层状态，用于区分"已提交未确认"和"未提交"的变更，确保重连后两者都能正确重放。

**ChangesetTracker 三层状态**（[changesettracker.ts L88-L148](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/changesettracker.ts#L88-L148)）：

| 变量 | 含义 |
|------|------|
| `baseAText` | **基准文本**——服务端确认的最后一个版本对应的带属性文本 |
| `submittedChangeset` | **已提交但未确认**的变更（即上一次 `USER_CHANGES` 发送出去但还没收到 `ACCEPT_COMMIT`） |
| `userChangeset` | **未提交**的变更（本地新输入，还没发出去） |

每次接收服务端变更（`applyChangesToBase`）时，这三者会通过 `follow()` 函数一起前进，确保本地用户的光标和输入位置始终正确跟随文档变化：

```typescript
// changesettracker.ts L110-L130
applyChangesToBase: (c, optAuthor, apoolJsonObj) => {
  baseAText = applyToAText(c, baseAText, apool);  // 基准前进

  if (submittedChangeset) {
    // 让"已提交未确认"的变更跟随服务端变更前进（变基）
    submittedChangeset = follow(c, oldSubmittedChangeset, false, apool);
    c2 = follow(oldSubmittedChangeset, c, true, apool);
  }

  // 让"未提交"的本地变更也跟随前进（保持用户输入的相对位置）
  userChangeset = follow(c2, oldUserChangeset, preferInsertingAfterUserChanges, apool);

  // 最终把合并后的变更实际应用到 DOM
  applyingNonUserChanges = true;
  try {
    callbacks.applyChangesetToDocument(postChange, preferInsertionAfterCaret);
  } finally {
    applyingNonUserChanges = false;  // 确保标志位一定被重置
  }
}
```

注意 `applyingNonUserChanges` 标志位的作用——在应用非用户变更的过程中，DOM 会发生变化（其他人的新文本插入），这些变化**不能**被当作用户新输入再次捕获，所以 `composeUserChangeset` 会直接 return：

```typescript
// changesettracker.ts L91-L98
composeUserChangeset: (c) => {
  if (!tracking) return;
  if (applyingNonUserChanges) return;  // ★ 应用服务端变更期间的 DOM 变化不捕获
  if (isIdentity(c)) return;
  userChangeset = compose(userChangeset, c, apool);
  setChangeCallbackTimeout();
}
```

---

**本地变更的收集（`getMissedChanges`）**：

在进入重连流程之前，客户端通过 `getMissedChanges()` 把三层状态拍平成两层（"已提交"和"未提交"），供后续可能的重连重放使用：

```typescript
// collab_client.ts L428-L443
const getMissedChanges = () => {
  const obj = {};
  obj.userInfo = userSet[userId];
  obj.baseRev = rev;
  
  // 第 1 层：已提交但未确认的变更
  if (committing && stateMessage) {
    obj.committedChangeset = stateMessage.changeset;
    obj.committedChangesetAPool = stateMessage.apool;
    // 把 submittedChangeset 合并进 baseAText（因为服务端可能已经接受了它）
    editor.applyPreparedChangesetToBase();
  }
  
  // 第 2 层：未提交的本地变更
  const userChangesData = prepareUserChangeset();
  if (userChangesData.changeset) {
    obj.furtherChangeset = userChangesData.changeset;
    obj.furtherChangesetAPool = userChangesData.apool;
  }
  return obj;
};
```

**重连后恢复的完整时序**：
1. Socket.io 断线 → `channelState` 变为 `DISCONNECTED` / `RECONNECTING`
2. `setIsPendingRevision(true)` → 门禁 5 开始拦截提交
3. 用户继续输入 → DOM 正常渲染 → ChangesetTracker 累积到 `userChangeset`
4. Socket.io 自动重连成功 → 发送 `CLIENT_READY` + `reconnect: true`
5. 服务端逐个补发 `CLIENT_RECONNECT` 消息
6. 客户端每条都应用（别人的变更）或确认（自己的变更）
7. 最后一条 `CLIENT_RECONNECT`（`newRev === headRev`）→ `setIsPendingRevision(false)`
8. **边沿触发**：`wasPending=true, value=false` → 立即调用 `handleUserChanges()`
9. 此时门禁 5 不再拦截 → 把累积了整个断线期间的 `userChangeset` 合并提交

---

## 四、光标/选区视觉呈现机制

### 4.1 作者颜色系统

**核心文件**：
- [ace2_inner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/ace2_inner.ts#L215-L313) —— 作者样式管理
- [linestylefilter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/linestylefilter.ts) —— 行样式过滤器

**工作原理**：
1. 每个字符携带 `author` 属性
2. 渲染时，每行根据作者属性生成 CSS 类名（如 `author-a-z122z-z98z`）
3. 通过动态 CSS 设置不同作者的背景色和文字颜色

**类名生成**：
```typescript
// ace2_inner.ts L289-L292
const getAuthorClassName = (author) => `author-${author.replace(/[^a-y0-9]/g, (c) => {
  if (c === '.') return '-';
  return `z${c.charCodeAt(0)}z`;
})}`;
```

**动态样式设置**：
```typescript
// ace2_inner.ts L219-L273
const setAuthorStyle = (author, info) => {
  const authorSelector = getAuthorColorClassSelector(getAuthorClassName(author));
  
  if (info.bgcolor) {
    // 褪色处理（非活跃用户）
    if (fadeInactiveAuthorColors && (typeof info.fade) === 'number') {
      bgcolor = fadeColor(bgcolor, info.fade);
    }
    // WCAG-AA 可读性保障
    bgcolor = colorutils.ensureReadableBackground(bgcolor, ...);
    // 设置背景色和文字颜色
    style.backgroundColor = bgcolor;
    style.color = textColor;
  }
};
```

### 4.2 用户离开后的褪色效果

用户离开时，调用 `fadeAceAuthorInfo` 将其颜色设置为 50% 透明度：

```typescript
// collab_client.ts L360-L362
const fadeAceAuthorInfo = (userInfo) => {
  tellAceAuthorInfo(userInfo.userId, userInfo.colorId, true);
};

// true → inactive → fade: 0.5
editor.setAuthorInfo(userId, { bgcolor: cssColor, fade: 0.5 });
```

### 4.3 选区变更钩子

编辑器内部选区变化时触发 `aceSelectionChanged` 钩子，插件可通过此钩子实现光标追踪：

```typescript
// ace2_inner.ts L2033-L2037
hooks.callAll('aceSelectionChanged', {
  rep,
  callstack: currentCallStack,
  documentAttributeManager,
});
```

---

## 五、关键设计决策

### 5.1 为什么用 500ms 延迟？

- **平衡实时性与性能**：500ms 是人眼可感知的"实时"阈值
- **减少网络开销**：合并多次小输入，降低服务端压力
- **避免冲突**：减少多人同时编辑同一位置的冲突概率

### 5.2 为什么 Changeset 而不是 OT？

Etherpad 使用的是 **Changeset + Follow（变基）** 模式，而非传统 OT：
- 服务端串行处理所有提交
- 客户端提交的变更先变基到最新版本再应用
- 实现更简单，保证强一致性

### 5.3 为什么需要 serverMessageTaskQueue？

- 保证消息按顺序处理（服务端发来的消息可能乱序到达）
- 避免输入法合成期间的 DOM 操作冲突
- 异步操作串行化，防止状态竞争

### 5.4 重连时为什么用 CLIENT_RECONNECT 而不是 NEW_CHANGES？

- 语义区分：`CLIENT_RECONNECT` 表示"补发断线期间的变更"
- 特殊处理：自己的提交直接确认（`acceptCommit`），无需重复应用
- 进度感知：`headRev` 字段让客户端知道还有多少变更待接收

---

## 六、核心文件索引

| 模块 | 文件 | 关键函数/变量 |
|------|------|-------------|
| 协作客户端 | [collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/collab_client.ts) | `handleUserChanges`, `commitDelay`, `getMissedChanges` |
| 变更跟踪器 | [changesettracker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/changesettracker.ts) | `setChangeCallbackTimeout`, `composeUserChangeset` |
| 编辑器内核 | [ace2_inner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/ace2_inner.ts) | `setAuthorInfo`, `setAuthorStyle`, `repSelectionChange` |
| Pad 主逻辑 | [pad.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/pad.ts) | `sendClientReady`, `handshake`, `handleChannelStateChange` |
| 服务端消息处理 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleClientReady`, `handleUserChanges`, `updatePadClients` |
| 消息类型定义 | [SocketIOMessage.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/types/SocketIOMessage.ts) | 所有 Socket 消息类型定义 |

---

## 七、扩展：实现真正的光标追踪

Etherpad 核心不包含光标位置实时同步，但可通过以下方式扩展：

1. **监听 `aceSelectionChanged` 钩子** 获取本地光标位置
2. **通过 `CLIENT_MESSAGE` 自定义消息** 广播光标位置
3. **服务端转发** 给其他用户
4. **其他用户接收后** 在编辑器上渲染光标指示器

参考插件：`ep_cursortracing`、`ep_cursormesh`
