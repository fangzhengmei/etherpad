# Etherpad Client 重连与状态恢复代码路径分析

本文档分析 Etherpad 客户端断线检测、自动重连以及编辑状态恢复的完整代码路径。

---

## 一、核心文件索引

| 文件 | 职责 |
|------|------|
| [socketio.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/socketio.ts) | Socket.IO 连接创建与基础配置 |
| [pad.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts) | 主控层：handshake() 建立连接、注册 socket 事件、状态路由 |
| [collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts) | 协作客户端：通道状态机、提交/接受 changeset、断线超时检测、状态恢复核心逻辑 |
| [pad_connectionstatus.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_connectionstatus.ts) | UI 连接状态：connecting / connected / reconnecting / disconnected |
| [pad_modals.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_modals.ts) | 模态框显示层，驱动 [pad_automatic_reconnect.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_automatic_reconnect.ts) |
| [pad_automatic_reconnect.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_automatic_reconnect.ts) | 自动重连倒计时 UI、指数退避计时器、手动重连触发 |
| [changesettracker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/changesettracker.ts) | 三层文本状态模型：baseAText / submittedChangeset / userChangeset |
| [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts) | 服务端：handleClientReady() 处理 CLIENT_READY（含 reconnect 分支），下发 CLIENT_RECONNECT |

---

## 二、断线检测的两条路径

断线由**两条独立机制**协同检测，任一条触发都会进入重连流程：

### 2.1 路径 A：Socket.IO 层事件（传输层）

入口在 [pad.ts:353-508](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L353-L508) `handshake()` 函数中：

```
socket.io 配置参数（pad.ts:361-367）：
  reconnection: true              允许自动重连
  reconnectionAttempts: 5         最多尝试 5 次
  reconnectionDelay: 1000         首次重连延迟 1s
  reconnectionDelayMax: 5000      最大延迟 5s
```

触发的事件流：

| Socket.IO 事件 | 处理函数 | 效果 |
|----------------|----------|------|
| `disconnect` | [pad.ts:390-396](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L390-L396) | 调用 `socketReconnecting()` 进入 RECONNECTING |
| `reconnect_attempt` | [pad.ts:381-388 + 425](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L381-L388) | 同上，每次尝试都会标记 |
| `reconnect_failed` | [pad.ts:427-434](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L427-L434) | 5 次尝试耗尽 → DISCONNECTED `reconnect_timeout` |
| `error` | [pad.ts:437-446](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L437-L446) | 不抛异常，仅 setStateIdle + setIsPendingRevision(true) |
| 消息中带 `obj.disconnect` | [pad.ts:478-487](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L478-L487) | 服务端主动踢人：`padconnectionstatus.disconnected(msg)`，禁用编辑器 |

`socketReconnecting()` 函数 [pad.ts:381-388](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L381-L388) 是统一过渡动作：
```
1. collabClient.setStateIdle()          取消正在进行的提交标记（committing = false）
2. collabClient.setIsPendingRevision(true)  打开“等待服务器补回历史 revisions”闸门
3. collabClient.setChannelState('RECONNECTING')  广播状态变更 → UI 弹 reconnecting 模态框
```

### 2.2 路径 B：collab_client 应用层超时（业务层）

在 [collab_client.ts:95-158](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L95-L158) `handleUserChanges()` 中，每次循环都做超时检查：

```
handleUserChanges() 循环（递归 setTimeout 调度）：

① 初始连接超时 [L102-104]
   条件：channelState === 'CONNECTING' 且 now - initialStartConnectTime > 20s
   结果：setChannelState('DISCONNECTED', 'initsocketfail')

② 提交响应超时 [L112-122]
   条件：committing === true
     · > 20s  → setChannelState('DISCONNECTED', 'slowcommit')
     · > 5s   → callbacks.onConnectionTrouble('SLOW')  （仅 UI 提示“同步慢”）
     · 否则    → 3s 后再次检查
   
③ 正常节流 [L125-129]
   距离上次提交 < commitDelay(默认 500ms) → 延时后再试
```

路径 B 的超时不会触发 Socket.IO 的自动重连机制（因为传输层是通的，只是服务端不回包），而是直接进入 `DISCONNECTED`，依赖用户点击 “Reconnect” 按钮或 `pad_automatic_reconnect` 的倒计时触发 `location.reload()` 全页刷新。

---

## 三、重连流程详解

重连分为**温和重连**（Socket.IO 自己建立了新连接）和**强制重连**（手动 / 超时触发页面重载）两条路径。

### 3.1 温和重连：Socket.IO reconnect

事件入口在 [pad.ts:373-379](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L373-L379)：

```
socket.io.on('reconnect', () => {
  pad.collabClient?.setChannelState('CONNECTED');   // ① 先标记通道正常
  sendClientReady(receivedClientVars);              // ② 发送带 client_rev 的 CLIENT_READY
});
```

紧接着 **①** 的 `setChannelState('CONNECTED')` 会触发 UI 状态流转，详见下文第五节。

#### 3.1.1 客户端发送 CLIENT_READY（reconnect 分支）

[sendClientReady()](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L298-L351) 的关键参数：

```
isReconnect = true 时附加：
  msg.client_rev = collabClient.getCurrentRevisionNumber()   // 客户端当前 rev
  msg.reconnect  = true                                      // 服务端识别位
```

`client_rev` 是客户端**最后确认并应用到本地 baseAText 的修订号**。它是服务端补回差异的起点。

#### 3.1.2 服务端处理 CLIENT_READY（reconnect 分支）

在 [PadMessageHandler.ts:1208-1269](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1208-L1269)：

```
1. socket.join(sessionInfo.padId)           重新加入 pad 房间
2. sessionInfo.rev = message.client_rev     服务端记录客户端当前 rev

3. 计算缺失范围：
   startNum = client_rev + 1
   endNum   = pad.getHeadRevisionNumber() + 1

4. 并行拉取 [startNum, endNum) 内每一条 revision：
   - changeset、author、timestamp

5. 逐条下发 CLIENT_RECONNECT 消息：
   每条消息 = {
     type: 'CLIENT_RECONNECT',
     headRev: 最新 revision 号,
     newRev: 本条 revision 号,
     changeset: 经 prepareForWire 转换后的 changeset,
     apool: 属性池快照,
     author, currentTime
   }

6. 若 startNum === endNum（无缺失）：
   单发 { type: 'CLIENT_RECONNECT', noChanges: true, newRev: headRev }
```

注意：**每条缺失 revision 对应一条 CLIENT_RECONNECT 消息**，而不是合并成一个大 changeset。这保证了作者归属、属性池等信息的精确性，但客户端必须按序通过 `serverMessageTaskQueue`（Promise 链）串行处理。

#### 3.1.3 客户端接收 CLIENT_RECONNECT

处理逻辑在 [collab_client.ts:242-267](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L242-L267)：

```
serverMessageTaskQueue.enqueue(() => {
  if (msg.noChanges) {                              无缺失 → 直接解封
    setIsPendingRevision(false);
    return;
  }

  校验 newRev === rev + 1                           保证单调递增
  rev = newRev

  if (author === pad.getUserId()) {                 这是“我”在断线前提交的改动
    acceptCommit();                                 本地合并提交到 base
  } else {                                          这是他人在我断线期间的改动
    editor.applyChangesToBase(changeset, author, apool);   作为外部变更应用
  }

  if (newRev === headRev) {                         全部补完 → 解封
    setIsPendingRevision(false);
  }
});
```

关键点 `setIsPendingRevision(false)` 的副作用 [collab_client.ts:451-461](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L451-L461)：
```
wasPending=true 且 value=false 时 → 立即调用 handleUserChanges()
  这会把断线期间我本地累积但未发送的编辑（userChangeset）打包提交出去。
```

#### 3.1.4 pending revision 闸门

`isPendingRevision` 变量贯穿整个重连过程，防止客户端在“已连上 socket，但服务器历史修订还没补完”时乱发本地改动：

- 断线时置 `true`（路径 A）
- 发 USER_CHANGES 前检查 [collab_client.ts:134](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L134)：
  ```
  if (!isPendingRevision) { ... 才允许准备并发送 changeset ... }
  ```
- 否则每 3s 重试一次 [collab_client.ts:150-151](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L150-L151)

### 3.2 强制重连：倒计时 + reload

当温和重连失败（`reconnect_failed`）或业务层超时（`slowcommit` / `initsocketfail`），客户端进入 `DISCONNECTED` 状态。此时 `pad_connectionstatus.disconnected()` 触发 `padmodals.showModal(k)`，后者调用：

```
automaticReconnect.showCountDownTimerToReconnectOnModal($modal, pad)
  [pad_automatic_reconnect.ts:5-18]
```

#### 3.2.1 指数退避计时器

`reconnectionTries` 对象 [pad_automatic_reconnect.ts:113-123](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_automatic_reconnect.ts#L113-L123)：

```
counter: 0
nextTry() → 返回 2^counter，然后 counter++
```

每次倒计时持续时间 = `clientVars.automaticReconnectionTimeout * 2^counter`

#### 3.2.2 倒计时与网络错误分支

[createTimerForModal()](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_automatic_reconnect.ts#L56-L74)：

```
onExpire() 回调：
  if ($modal.is('.disconnected')) {       // 这是网络问题类断线
    waitUntilClientCanConnectToServerAndThen(
      () => forceReconnection($modal), pad
    )
    // waitUntilClientCanConnectToServerAndThen:
    //   1. counter === 1 时注册 socket.once('connect', callback)
    //   2. pad.socket.connect() 主动尝试建立连接
  } else {
    forceReconnection($modal);            // 其他情况直接触发
  }
```

`forceReconnection($modal)` [L100-102](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_automatic_reconnect.ts#L100-L102) → 触发 `#forcereconnect` 的 click → 最终执行：

```
padconnectionstatus.init() 中绑定 [pad_connectionstatus.ts:35-37]：
  $('button#forcereconnect').on('click', () => {
    window.location.reload();
  });
```

所以强制重连的本质是**全页面重载**，这会丢掉未被服务器接受的 changeset。为了避免丢数据，Etherpad 提供了 `getMissedChanges()` 机制，详见下节。

---

## 四、编辑状态恢复机制

重连过程中，编辑状态被三层 changeset 精确跟踪。

### 4.1 三层状态模型（changesettracker.ts）

在 [changesettracker.ts:31-48](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/changesettracker.ts#L31-L48) 定义了三层：

```
┌─────────────────────────────────────────────────────┐
│ baseAText                                           │  服务器已确认的官方文本
│   （已应用所有 ACCEPT_COMMIT 和 NEW_CHANGES）        │
├─────────────────────────────────────────────────────┤
│ submittedChangeset                                  │  已发送但未收到 ACCEPT_COMMIT
│   （prepareUserChangeset() 产出，正飞行中）          │
├─────────────────────────────────────────────────────┤
│ userChangeset                                       │  还未 prepare 的用户编辑
│   （用户在编辑器里敲字，composeUserChangeset 累积）  │
└─────────────────────────────────────────────────────┘
```

三者通过 `compose` / `follow`（操作变换 OT 函数）精确组合后反映到屏幕上。

### 4.2 提交周期

正常周期（handleUserChanges 触发）：

```
① prepareUserChangeset()  [changesettracker.ts:133-188]
   · 若 submittedChangeset 存在：compose(submitted, user) → toSubmit
   · 否则：清理 userChangeset 的作者归属（替换为当前用户）→ toSubmit
   · submittedChangeset = toSubmit
   · userChangeset = identity(newLen)
   · 返回 { changeset, apool } 的 wire 格式

② sendMessage({ type: 'USER_CHANGES', baseRev, changeset, apool })
   此时 committing = true，stateMessage 缓存那份 USER_CHANGES

③ 服务端回 ACCEPT_COMMIT  [collab_client.ts:228-241]
   校验 newRev ∈ {rev, rev+1} → rev = newRev → acceptCommit()

④ acceptCommit()  [collab_client.ts:160-168]
   editor.applyPreparedChangesetToBase()    →  baseAText += submittedChangeset
   stateMessage = null
   setStateIdle()                            →  committing = false
   再调度 handleUserChanges()
```

### 4.3 断线期间状态保留

温和重连期间，整个三层状态**完全保留在内存里**：
- submittedChangeset：断线前已发送未确认的那份，保留
- userChangeset：断线时用户继续打字继续累积

这就是为什么 `isPendingRevision` 闸门如此重要：不能在历史修订补完之前把本地 changeset 发出去，否则 baseRev 不匹配。

### 4.4 getMissedChanges()：强制重连前的“抢救”

如果被迫 `location.reload()`，内存会丢失。因此在表单提交 `forceReconnect()` 之前，`pad.forceReconnect()` [pad.ts:1065-1072](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L1065-L1072) 会把状态序列化进隐藏表单：

```
const missed = pad.collabClient.getMissedChanges()
  [collab_client.ts:428-443]：
    {
      userInfo,                           // 当前用户信息
      baseRev: rev,                       // 客户端已知 rev
      committedChangeset,                 // stateMessage.changeset（若有提交中）
      committedChangesetAPool,            // stateMessage.apool
      furtherChangeset,                   // userChangeset（若有未 prepare 的）
      furtherChangesetAPool
    }
```

注意：若存在 `committing && stateMessage`，在提取前先 `editor.applyPreparedChangesetToBase()`，把 in-flight 的 changeset 也加入 base，以便后续 `furtherChangeset` 以正确的 oldLen 为起点。重载后服务端的 pad 页面加载逻辑会读取这些表单字段并重放用户的未保存编辑。

---

## 五、通道状态机与 UI 协同

### 5.1 状态流转图

```
                handshake() 开始
                      │
                      ▼
              CONNECTING ──────── 20s 超时 ──────► DISCONNECTED(initsocketfail)
                │    ▲
   socket.on('connect')  │        committing > 5s
                │    │        │        │
                ▼    │        ▼        │
            CONNECTED  └────── SLOW ────┘
                │
                │  socket disconnect / reconnect_attempt
                ▼
           RECONNECTING ◄────────────────────────┐
                │                                 │
                │  socket.io reconnect 成功       │  应用层超时 >20s
                ▼                                 │  reconnect_failed
         (回到 CONNECTED)                         │
                │                                 │
                └─────────────────────────────────┴────► DISCONNECTED(xxx)
                                                              │
                                                    pad_automatic_reconnect 倒计时
                                                              │
                                                              ▼
                                                    location.reload()
```

### 5.2 状态变更回调链

`collabClient.setChannelState(state, info)` → [collab_client.ts:374-379](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L374-L379) 调用 `callbacks.onChannelStateChange`，后者绑定到：

```
pad.handleChannelStateChange(newState, message)  [pad.ts:985-1028]
  ├─ newState === CONNECTED:
  │    padeditor.enable()
  │    padeditbar.enable()
  │    padimpexp.enable()
  │    padconnectionstatus.connected()
  │      └─ showModal('connected') + hideOverlay()
  │
  ├─ newState === RECONNECTING:
  │    padeditor.disable()
  │    padeditbar.disable()
  │    padimpexp.disable()
  │    padconnectionstatus.reconnecting()
  │      └─ showModal('reconnecting') + showOverlay()
  │
  └─ newState === DISCONNECTED:
       收集 diagnosticInfo → POST /ep/pad/connection-diagnostic-info
       padeditor / padeditbar / padimpexp.disable()
       padconnectionstatus.disconnected(message)
         └─ 对 message 做白名单（badChangeset/corruptPad/deleted/...）
            showModal(k) + showOverlay()
       若 hasUnacceptedCommit() → 弹 gritter “未保存更改”警告
```

同时 `handleIsFullyConnected(newFullyConnected)` 在首次连通时刷新控件可见性、关闭下拉菜单。

---

## 六、消息队列：重连期间避免乱序

服务端发回的 `NEW_CHANGES` / `ACCEPT_COMMIT` / `CLIENT_RECONNECT` 都进入同一个串行 Promise 队列 [collab_client.ts:187-200](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L187-L200) `serverMessageTaskQueue`：

```
class {
  _promiseChain = Promise.resolve()
  enqueue(fn) {
    taskPromise = _promiseChain.then(fn)
    _promiseChain = taskPromise.catch(() => {})      // 单条失败不中断后续
    return await taskPromise
  }
}
```

配合 `await editor.getInInternationalComposition()`（NEW_CHANGES / CLIENT_RECONNECT 分支），保证用户在输入法合成态时不会被外部 changeset 打断 DOM 更新。

此外，在 `collabClient` 尚未创建之前，`pad._messageQ`（[MessageQueue 类 pad.ts:511-530](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L511-L530)）会暂存所有 socket 消息，collabClient 创建完成后立即 flush。这保护了**首屏加载过程**中 CLIENT_VARS 之后到达的消息不被丢失。

---

## 七、小结：完整链路一览

**用户正在编辑 → Wi-Fi 短暂断开 → 自动恢复** 的典型时序：

```
时间轴  客户端                                                          服务端
 │     1. 用户敲字 → composeUserChangeset() 累积到 userChangeset
 │     2. handleUserChanges() 检测 committing + 5s → onConnectionTrouble(SLOW)
 │     3. socket 发现心跳超时 → disconnect 事件
 │     4. socketReconnecting():
 │        setStateIdle() + setIsPendingRevision(true) + setChannelState(RECONNECTING)
 │     5. UI 显示“重新连接中…”，编辑器禁用
 │     6. Socket.IO 底层按指数退避尝试重连（1s, 2s, ...）
 │                                                                 7. 接受新连接，socket.io.on('reconnect') 触发
 │     8. setChannelState(CONNECTED) → UI 进入“已连接”
 │     9. sendClientReady(isReconnect=true):
 │          { client_rev: N, reconnect: true } ──────────────────────►
 │                                                                  10. handleClientReady():
 │                                                                       ① join(padId)
 │                                                                       ② startNum = N+1, endNum = head+1
 │                                                                       ③ 拉取每条 revision 的 cs/author/ts
 │                                                                       ④ 逐条下发 CLIENT_RECONNECT
 │        ◄─────────────── {CLIENT_RECONNECT, newRev:N+1, changeset...}
 │     11. serverMessageTaskQueue 串行处理每条：
 │           · newRev !== rev+1 警告跳过
 │           · rev++
 │           · 自己的 changeset → acceptCommit() 合并到 base
 │           · 别人的 changeset → applyChangesToBase() 应用到 base（OT）
 │           · 到达 headRev → setIsPendingRevision(false)
 │
 │     12. setIsPendingRevision(false) 副作用：handleUserChanges()
 │           · 把用户在断线期间新敲的字 prepare 成 changeset
 │           · 构造 USER_CHANGES(baseRev=head) 发送 ──────────────►
 │                                                                  13. 接受并广播：ACCEPT_COMMIT + NEW_CHANGES
 │        ◄──────────────────────────────────────── ACCEPT_COMMIT(newRev)
 │     14. acceptCommit() 合并到 base，committing=false
 │        编辑器重新启用，恢复正常协作
 ▼
```

这条链路中任何一个环节出问题（比如 CLIENT_RECONNECT 校验 newRev 失败），系统都不会强抛异常——仅打 `console.warn`，留待后续 `slowcommit` 超时或用户手动刷新兜底，最大程度保证“页面始终可操作、可手动重连”。
