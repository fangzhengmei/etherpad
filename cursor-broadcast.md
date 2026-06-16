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

#### 第二层：CollabClient 的提交延迟（commitDelay）

**文件**：[collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/8-etherpad-lite/src/static/js/collab_client.ts#L49)

```typescript
let commitDelay = 500; // 默认 500ms 提交延迟
```

**核心逻辑**：`handleUserChanges` 函数

```typescript
// collab_client.ts L95-L158
const handleUserChanges = () => {
  const now = Date.now();
  
  // ... 状态检查（连接中、提交中等）
  
  // 计算最早可提交时间
  const earliestCommit = lastCommitTime + commitDelay;
  if (now < earliestCommit) {
    setTimeout(handleUserChanges, earliestCommit - now);
    return; // 还没到提交时间，延后再试
  }

  // 没有待处理的服务端变更时才提交
  if (!isPendingRevision) {
    const userChangesData = prepareUserChangeset();
    if (userChangesData.changeset) {
      lastCommitTime = now;
      committing = true;
      stateMessage = {
        type: 'USER_CHANGES',
        baseRev: rev,
        changeset: userChangesData.changeset,
        apool: userChangesData.apool,
      };
      sendMessage(stateMessage); // 发送到服务端
    }
  }
};
```

**去抖效果**：
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

4. **确认提交**
   - 向提交者发送 `ACCEPT_COMMIT` 消息
   - 更新该客户端的 `session.rev`

5. **广播给所有客户端**
   - 调用 `updatePadClients(pad)`
   - 遍历所有在线用户，发送 `NEW_CHANGES` 消息

#### 关键代码：广播逻辑

```typescript
// PadMessageHandler.ts L1008-L1066
exports.updatePadClients = async (pad) => {
  const roomSockets = _getRoomSockets(pad.id);
  
  await Promise.all(roomSockets.map(async (socket) => {
    const sessioninfo = sessioninfos[socket.id];
    
    // 逐个补发缺失的版本
    while (sessioninfo.rev < pad.getHeadRevisionNumber()) {
      const r = sessioninfo.rev + 1;
      const revision = await pad.getRevision(r);
      
      const msg = {
        type: 'COLLABROOM',
        data: {
          type: 'NEW_CHANGES',
          newRev: r,
          changeset: forWire.translated,
          apool: forWire.pool,
          author,
          currentTime,
          timeDelta: currentTime - sessioninfo.time,
        },
      };
      socket.emit('message', msg);
      sessioninfo.rev = r;
    }
  }));
};
```

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

### 3.4 本地变更保存与重放

断线期间用户可能继续输入，这些本地变更需要在重连后重新提交。

**本地变更保存**：

```typescript
// collab_client.ts L428-L443
const getMissedChanges = () => {
  const obj = {};
  obj.userInfo = userSet[userId];
  obj.baseRev = rev;
  
  // 已提交但未确认的变更
  if (committing && stateMessage) {
    obj.committedChangeset = stateMessage.changeset;
    obj.committedChangesetAPool = stateMessage.apool;
    editor.applyPreparedChangesetToBase();
  }
  
  // 未提交的本地变更
  const userChangesData = prepareUserChangeset();
  if (userChangesData.changeset) {
    obj.furtherChangeset = userChangesData.changeset;
    obj.furtherChangesetAPool = userChangesData.apool;
  }
  return obj;
};
```

**重连后恢复**：
- `isPendingRevision = true` 期间，本地变更暂不提交
- 所有服务端补发的变更应用完成后，`isPendingRevision = false`
- 触发 `handleUserChanges()` 重新提交本地累积的变更

```typescript
// collab_client.ts L451-L461
const setIsPendingRevision = (value) => {
  const wasPending = isPendingRevision;
  isPendingRevision = value;
  
  // 待处理版本全部应用完后，刷新本地变更
  if (wasPending && !value) {
    handleUserChanges();
  }
};
```

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
