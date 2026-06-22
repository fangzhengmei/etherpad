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

这条链路中任何一个环节出问题（比如 CLIENT_RECONNECT 校验 newRev 失败），系统都不会强抛异常——仅打 `console.warn`，留待后续 `slowcommit` 超时或用户手动刷新兜底，最大程度保证"页面始终可操作、可手动重连"。

---

## 八、OT 对齐深度：compose/follow 如何对齐本地未发编辑与历史修订

重连场景最绕的部分就是：服务端补回的 `CLIENT_RECONNECT` 逐条 `applyChangesToBase()` 时，如何让 **submittedChangeset（断线前已发未确认）** 和 **userChangeset（断线期间继续累积的本地打字）** 都能正确前进，既不丢字也不重复。

### 8.1 compose 与 follow 的语义区别

两者入口都在 [Changeset.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/Changeset.ts)，语义截然不同：

| 函数 | 签名 | 适用场景 | 关键断言 |
|------|------|----------|----------|
| `compose(cs1, cs2, pool)` | [L755-784](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/Changeset.ts#L755-L784) | 串联：cs1 先发生，**紧接着** cs2 发生，`cs1.newLen === cs2.oldLen` | `assert(len2 === unpacked2.oldLen, 'mismatched composition')` |
| `follow(cs1, cs2, reverseInsertOrder, pool)` | [L1446-1585](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/Changeset.ts#L1446-L1585) | **并发（OT 变换）**：cs1、cs2 都基于同一旧文档，要把它们"投影"到同一新文档上 | `assert(len1 === len2, 'mismatched follow - cannot transform cs1 on top of cs2')` |

通俗理解：
- `compose(A, B)` = "先做 A 再做 B，合并成一步"
- `follow(A, B, true/false)` = "A 和 B 同时发生，把 B 变换到 A 执行后的新坐标"；reverseInsertOrder 解决同位置并发插入的对称性破缺。

### 8.2 applyChangesToBase 里的六步变换链

在 [changesettracker.ts:99-132](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/changesettracker.ts#L99-L132)，每一条服务端补回的 changeset `c`（不管是 NEW_CHANGES 还是 CLIENT_RECONNECT）都会经历这 6 步：

```
变量名约定：
  c                     = 当前外部 changeset（服务端来的，基于 baseAText）
  oldSubmittedChangeset = 断线前已发送给服务器但还没被 ACCEPT 的那份本地提交
  oldUserChangeset      = 断线期间用户继续打字积累、还没 prepare 的本地编辑
```

```
步骤 1  [L108]
  baseAText = applyToAText(c, baseAText, apool)
  意义：服务器的权威变更先落地到 baseAText，baseAText 永远前进到 c 执行后的新状态。

步骤 2  [L111-114]（仅 submittedChangeset 存在时执行）
  submittedChangeset = follow(c, oldSubmittedChangeset, false, apool)
  意义：把"我刚发出去但没确认的那份提交"变换到『c 已经执行后的新坐标』上。
        reverseInsertOrder=false → 同位置若并发插入，外部 c 的插入排在前面，我的靠后。

步骤 3  [L110,114]（仅 submittedChangeset 存在时执行）
  c2 = follow(oldSubmittedChangeset, c, true, apool)
  意义：算出『如果世界线是先执行我那份旧提交、再执行外部 c』等价的外部 c。
        下一步 userChangeset 要基于这个 c2 来变换——因为 userChangeset 的位置基准本来就是
        『oldSubmittedChangeset 之后』（oldSubmitted 是 prepareUserChangeset 时从 userChangeset
        里抽走前进到 submitted 的，userChangeset 被重置为 identity(newLen)）。

步骤 4  [L117-120]
  userChangeset = follow(c2, oldUserChangeset, true, apool)
  意义：把断线期间本地打字的 userChangeset 投影到『外部变更已经发生后的坐标』。
        preferInsertingAfterUserChanges=true → 反向插入顺序，我的本地插入排在前面，
        光标不会跳走。

步骤 5  [L121-122]
  postChange = follow(oldUserChangeset, c2, false, apool)
  意义：算出『屏幕上最终应该直接应用到 DOM 的 changeset』——这是真正反映到用户眼前的东西，
        经过相反的插入顺序偏好把外部 c 投射到"用户还以为自己在旧文档打字"的视图上。

步骤 6  [L127]
  callbacks.applyChangesetToDocument(postChange, preferInsertionAfterCaret=true)
  意义：把 postChange 打到编辑器 DOM，同时触发 applyingNonUserChanges 标志，
        防止 DOM 变更被递归地当成用户输入重新 compose 回 userChangeset（L92-94 的早返回）。
```

### 8.3 一个具体例子

> 场景：baseAText 是 "AB"（长度 2）；用户在位置 1 插入 "X"（变成 "AXB"）→ 这份 USER_CHANGES 刚发出未确认（oldSubmittedChangeset = insert 1 X, oldLen=2, newLen=3）；用户继续在 "X" 后敲 "Y" → oldUserChangeset = insert 2 Y, oldLen=3, newLen=4。
>
> 此时断线恢复，服务端补回一条 CLIENT_RECONNECT：**另一个作者在末尾插入了 "Z"**，即 c = insert 2 Z, oldLen=2, newLen=3。

执行 6 步后：
- 步骤 1：baseAText 成为 "ABZ"
- 步骤 2：oldSubmittedChangeset 被变换 → 插入位置保持 1（因为插入 Z 在后面不影响），submittedChangeset 仍表示在 1 处插 X，但 oldLen=3/newLen=4
- 步骤 3：c2 = 先插 X、再插 Z 的等价外部 c → 在新文档的位置 3 插 Z
- 步骤 4：userChangeset（在 2 处插 Y）被变换 → 仍在 2 处插 Y
- 步骤 5：postChange = 在屏幕上看到"AB" 末尾出现 Z，光标处 X/Y 布局不变

最终屏幕内容 "AXYB Z"，本地待发送的 submittedChangeset + userChangeset 都在正确坐标，下次发送时 baseRev 已经是 headRev，不会不匹配。

### 8.4 重连特判：CLIENT_RECONNECT 下作者是自己

在 [collab_client.ts:258-260](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L258-L260) 有个特判：
```
if (author === pad.getUserId()) {
  acceptCommit();
} else {
  editor.applyChangesToBase(changeset, author, apool);
}
```

如果 CLIENT_RECONNECT 补回的那条 revision **本来就是我断线前提交出去的**（服务端在我断线期间接受了它），就走 `acceptCommit()`：
- `editor.applyPreparedChangesetToBase()`：把 submittedChangeset 合并到 baseAText，并清空 submittedChangeset
- `setStateIdle()`：committing=false
- `handleUserChanges()`：立即准备下一轮

这避免了对"自己的已接受变更"再做一遍上面的六步 OT，省算力且不会让同一条 changeset 被重复应用。

---

## 九、心跳超时在哪发起：两层检测的叠加

断线检测并不是单点，而是**传输层心跳 + 应用层超时**两层叠加，各自有不同的触发边界。

### 9.1 第一层：engine.io 的心跳（传输层）

Socket.IO 建立在 engine.io 之上，底层心跳由 engine.io 自动发起，相关参数在 Etherpad 代码中**没有覆盖**，完全使用默认值：

- 服务端创建 socket.io 的位置：[socketio.ts:76-80](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/hooks/express/socketio.ts#L76-L80)
  ```
  io = new Server(args.server, {
    transports: settings.socketTransportProtocols,   // ['websocket','polling']
    cookie: false,
    maxHttpBufferSize: settings.socketIo.maxHttpBufferSize, // 1e6
  })
  ```
  注意这里**没有**传 `pingInterval` / `pingTimeout`。

- engine.io v6（Socket.IO v4 依赖）的默认值：
  - `pingInterval = 25000 ms`  —— 服务端每 25s 向客户端发 ping
  - `pingTimeout  = 20000 ms`  —— 发出 ping 后 20s 内没收到 pong 就认为断线

- 心跳方向：**由服务端发起 PING**，客户端用 engine.io 内置 PONG 回应。任一方在 pingTimeout 窗口未收到对端包，都触发 `disconnect` 事件，reason 包含 `'ping timeout'` / `'transport close'` / `'transport error'`。

- Etherpad 客户端监听位置：[pad.ts:390-396](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L390-L396)
  ```
  socket.on('disconnect', (reason) => { ... socketReconnecting(); })
  ```
  被触发后 Socket.IO 内置的自动重连（reconnectionDelay=1s, 指数退避, 最多 5 次）就开始了。

### 9.2 第二层：collab_client 的应用层超时

即使传输层通着（ping/pong 正常），服务端也可能因为某些卡死 / 队列阻塞不回业务包。handleUserChanges() 用**递归 setTimeout 轮询**做第二道防线：

| 超时阈值 | 位置 | 效果 |
|----------|------|------|
| 初始连接 > 20s | [collab_client.ts:102-104](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L102-L104) | `DISCONNECTED('initsocketfail')`，强制重连路径 |
| committing > 5s | [collab_client.ts:116-117](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L116-L117) | `onConnectionTrouble('SLOW')` → 仅顶部状态栏提示 "正在同步…" |
| committing > 20s | [collab_client.ts:113-115](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L113-L115) | `DISCONNECTED('slowcommit')`，强制重连路径 |
| 正常周期 < 500ms | [collab_client.ts:125-129](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L125-L129) | 节流，最短 500ms 才提交一次 |
| 发送后/空闲时 | [collab_client.ts:120,151,156](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L120-L156) | 每 3s 再次 poll，保证最长 3s 内就能检测到上一档次的超时 |

注意 3s 的检查粒度意味着：`committing > 5s` 的 SLOW 提示在实际时间 6~8s 才出现，`> 20s` 的断开在 21~23s 出现，和 engine.io 45s 的 pingInterval+pingTimeout 错峰，互为兜底。

### 9.3 为什么要两层

| 场景 | 被哪层检测到 | 延迟 |
|------|-------------|------|
| 拔网线、断 Wi-Fi | 传输层（TCP RST / 浏览器通知 websocket close）| ~0~数秒 |
| NAT 映射超时、中间设备丢包 | 传输层（ping timeout 20+25s）| 20~45s |
| Node.js 进程 GC STW、事件循环阻塞 | 应用层 slowcommit | 5~23s（比传输层快）|
| Socket 通着但服务端 pad 协程挂死 | 应用层 slowcommit | 21~23s |
| 首屏 handshake 时服务端 accept 后静默 | 应用层 initsocketfail | 20s |

两层叠加之后，"看似连上了但其实协作死了"这种最讨厌的灰色状态被极大缩短。

---

## 十、边界与退化分支详解

### 10.1 闸门（isPendingRevision）期间：新协作改动是排队，不会丢

`isPendingRevision` 只**阻塞发送**，不阻塞用户打字的累积。具体两条独立路径：

**路径 A：本地打字如何累积（完全不受闸门影响）**

调用链：
```
用户按键 → ace2_inner 的 onMutation/onInput 回调
  → editorInfo.ace_composeUserChangeset(c)
    → changesettracker.composeUserChangeset(c)  [changesettracker.ts:91-98]
        if (applyingNonUserChanges) return;   // 外部 changeset 正在应用，防止递归
        if (isIdentity(c)) return;
        userChangeset = compose(userChangeset, c, apool);   // ← 永远会执行
        setChangeCallbackTimeout();   // 调度下一次 handleUserChanges(0ms)
```

→ **结论：闸门期间，每一次按键都会正常 compose 到 userChangeset 上排队，数据不丢。**

**路径 B：handleUserChanges 的发送逻辑被闸门挡住**

[collab_client.ts:131-152](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L131-L152)：
```
if (!isPendingRevision) {
  prepareUserChangeset() → sendMessage(USER_CHANGES)    ← 闸门关闭（false）时才发送
  sentMessage = true
} else {
  setTimeout(handleUserChanges, 3000);   ← 闸门打开（true）时每 3s 重试一次
}
```

→ **结论：发送被无限重试排队，直到闸门关闭。闸门一关闭（false），立即走 L458-459 的副作用：**
```
if (wasPending && !value) {
  handleUserChanges();   // 把 userChangeset 整个 prepare 并发出
}
```

所以闸门期间，屏幕上的用户编辑 **既在内存里存在（userChangeset），又会在闸门关闭的第一刻自动发出去**——这就是"温和重连"能无缝恢复的原因。

### 10.2 服务端 handleClientReady 的退化分支

在 [PadMessageHandler.ts:1208-1269](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1208-L1269) 的 reconnect 分支里，包含三层防御性钳制，以及一层并发防御：

#### 10.2.1 client_rev 越界

```
startNum = message.client_rev + 1
endNum   = pad.getHeadRevisionNumber() + 1

// 钳制 1：client_rev < -1 时（比如客户端传了非法极小值），startNum 不能低于 0
if (startNum < 0) startNum = 0;    // [L1231-1233]

// 钳制 2：在拉取 revisionsNeeded 的 await 期间，若有新编辑推进 head，endNum 可能超过新 head+1
if (endNum > headNum + 1) endNum = headNum + 1;   // [L1227-1229]

// 退化：client_rev >= headNum（和服务器同步 / 或客户端乱填超大数）→ startNum === endNum
if (startNum === endNum) {
  socket.emit('message', {
    type: 'COLLABROOM',
    data: { type: 'CLIENT_RECONNECT', noChanges: true, newRev: headNum }
  });
}
```

→ **结论：无论客户端的 client_rev 乱填成什么，服务端都不会崩。**
- `client_rev < -1` → 从 rev=0 开始补整个 pad 历史（最坏情况，慢但正确）
- `client_rev >= headNum` → 发 noChanges=true，直接让客户端解封，从"此刻"的 head 开始继续同步

#### 10.2.2 并发 head 变化防御

注意代码里 `headNum` 在循环外赋值（L1225）、`endNum` 在循环外算一次后又和 L1227 的新 `pad.getHeadRevisionNumber()` 对比钳制——但下发 CLIENT_RECONNECT 消息时每条的 `headRev` 却是**在发送循环里实时取** `pad.getHeadRevisionNumber()`（L1254）：

```
for (const r of revisionsNeeded) {
  wireMsg = { ... headRev: pad.getHeadRevisionNumber(), newRev: r, ... };
  socket.emit('message', wireMsg);
}
```

这意味着：如果在逐条发送的过程中又有新的协作者推进了修订号，每条消息带上的 `headRev` 会变大，客户端要等到新 head 对应的那条 CLIENT_RECONNECT 也收到之后才会解封 isPendingRevision。对客户端来说看起来是"多等几秒补完新增的几条"，但正确性不受影响。

#### 10.2.3 异步等待期间客户端断线

在所有 await 之前（L1170）有一道检查：
```
if (sessionInfo !== sessioninfos[socket.id]) throw new Error('client disconnected');
```

因为整个 handleClientReady 是 async，中间有多次 `await padManager.getPad()` / `await Promise.all(revisionsNeeded.map(...))`。如果客户端在等待期间彻底断线且 socket.id 对应的 sessioninfos 被覆写，这道检查就会抛出，提前终止后续消息下发，避免往死连接里 emit。

#### 10.2.4 客户端 newRev 校验只 warn、不抛

在客户端接收 NEW_CHANGES / ACCEPT_COMMIT / CLIENT_RECONNECT 时都有类似校验：
```
if (newRev !== (rev + 1)) {
  window.console.warn(`bad message revision on NEW_CHANGES: ${newRev} not ${rev + 1}`);
  // setChannelState("DISCONNECTED", "badmessage_newchanges");  ← 被注释掉！
  return;
}
```

代码保留了注释掉的 setChannelState 调用，意味着**设计上有意选择"容忍单条坏消息、继续尝试后续消息"而非强断开**。这样做的理由：
- 并发场景下如果服务端在 CLIENT_RECONNECT 发完之前又有新的 NEW_CHANGES 广播，可能出现"CLIENT_RECONNECT 中 newRev 跳号 + NEW_CHANGES 先一步到达"交织的情况，跳过继续等后续消息最终还能对齐；
- 若真的是坏消息，后续要么 slowcommit 超时、要么用户手动 refresh，同样有兜底。

这也是第七节时序图末尾提到的"出错时尽量保可用、不直接崩"设计哲学的一部分。

---

## 十一、五处代码细节深度校正

### 11.1 endNum 钳制：为何"永远不触发"还写着

在上一节 10.2.1 我曾解读为"await 期间 head 推进的防御"，但对照 [PadMessageHandler.ts:1222-1229](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1222-L1229) 的**同步执行上下文**，这其实是个有趣的历史遗留陷阱：

```
L1222  let startNum = message.client_rev! + 1;
L1223  let endNum = pad.getHeadRevisionNumber() + 1;    // ① 第一次取 head
L1224
L1225  const headNum = pad.getHeadRevisionNumber();     // ② 紧接着第二次取 head
L1226
L1227  if (endNum > headNum + 1) {                      // ③ 钳制
L1228    endNum = headNum + 1;
L1229  }
```

关键：L1223 和 L1225 之间**没有 await、没有任何让出事件循环的调用**，在 Node.js 单线程模型下 pad 对象不会被改。因此对于任何一次 handleClientReady 调用，**endNum === headNum + 1 恒成立**，L1227 的判断恒为 false，钳制死代码。

#### 真实意图

结合 L1250-L1260 发送循环里每条消息单独取 `pad.getHeadRevisionNumber()` 作 `headRev` 的做法，可以还原这段的设计意图：

1. **未来维护防御**：这段代码的结构是"先算边界 → 中间可能加 DB 操作 → 再验证边界"。如果未来有人在 L1224 插入了 `await`（例如加一段权限校验或 hook），钳制就会自动生效，避免 startNum 大于实际 head。

2. **与发送循环 headRev 实时取法互补**：真正保护"逐条发消息期间 head 可能被并发推进"的其实是 L1254：
   ```
   for (const r of revisionsNeeded) {
     wireMsg.data.headRev = pad.getHeadRevisionNumber();   // 每条单独取
     socket.emit('message', wireMsg);
   }
   ```
   每条消息带的 `headRev` 是当下的最新值，客户端解封条件是 `if (newRev === headRev)`。如果发消息期间又有新协作者推进了 3 条 revision，那么原来的最后一条 CLIENT_RECONNECT 的 `headRev` 已经是新值，客户端不会解封；紧接着广播的三条 NEW_CHANGES（走 room 广播）会由 NEW_CHANGES 分支 `rev++` 推进到新 head——但这里有个微妙的时序问题：CLIENT_RECONNECT 消息还在 `isPendingRevision=true` 期间发 NEW_CHANGES，两者的 serverMessageTaskQueue 会严格按到达顺序执行，所以最后到达的那一条 CLIENT_RECONNECT 不会解封（newRev < headRev），等 NEW_CHANGES 推到 head 后 NEW_CHANGES 分支本身也不会主动调用 `setIsPendingRevision(false)`。

   这个"理论上的漏解封"实际上靠什么兜住？因为客户端在重连后第一个 CLIENT_READY 已经 join 回房间了，NEW_CHANGES 广播会同时到达。当客户端 NEW_CHANGES 分支推进 rev 到 headRev 时，虽然 NEW_CHANGES 分支不会 setIsPendingRevision(false)，但下一次 handleUserChanges 的发送分支会检查 `!isPendingRevision` 为 false 而继续 3s 轮询。因此只有两种结果：
   - 如果 CLIENT_RECONNECT 的最后一条 headRev 与当时 NEW_CHANGES 的最后一条 newRev 重合，CLIENT_RECONNECT 会正常解封（发消息循环是同步的，而 NEW_CHANGES 要经过广播链路，会更晚到，所以通常 headRev ≤ 当时 CLIENT_RECONNECT 的 newRev）。
   - 如果 CLIENT_RECONNECT 发送完之后才产生 NEW_CHANGES，那 NEW_CHANGES 会到达后 rev 推进，但此时 headRev 的比较已经做过了 → 客户端将永远停留在 isPendingRevision 直到 slowcommit 超时。这是一个已知的极小概率窗口，bug 表现为"重连后无法发送编辑"，需要用户手动刷新。

结论：L1227-L1229 的钳制是**给未来维护者留的护栏**，不是当前代码路径会触发的分支。

### 11.2 applyChangesToBase 前置 apool 翻译（moveOpsToNewPool）

此前八章 8.2 节列了"六步调用链"，但漏掉了**真正的第 0 步**——属性池翻译。这是重连场景里极易搞错的前提。代码在 [changesettracker.ts:103-106](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/changesettracker.ts#L103-L106)：

```
if (apoolJsonObj) {
  const wireApool = (new AttributePool()).fromJsonable(apoolJsonObj);
  c = moveOpsToNewPool(c, wireApool, apool);
}
```

#### 为什么必须翻译

- 服务端和客户端**各自维护独立的 AttributePool**，属性编号（`*0`, `*1`, `*j`…）分配是局部自增的。服务端 `*0` 可能代表 `['bold', 'true']`，客户端 `*0` 可能代表 `['author', 'a.john']`，完全对不上。
- 因此每条 NEW_CHANGES / CLIENT_RECONNECT 都附带一个"精简 apool"——只包含这条 changeset 真正用到的属性映射，格式是服务端当时的编号。
- 客户端在 apply 之前必须把 changeset 里所有 `*<num>` 属性引用的编号**从 wireApool 坐标系改写到本地 apool 坐标系**，否则后续 follow() / compose() 里比较属性字符串或取属性值都会对到错误的键上，表现为加粗被当成加粗+红色，或者作者颜色丢失等鬼畜 bug。

#### moveOpsToNewPool 实现细节

定义在 [Changeset.ts:934-952](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/Changeset.ts#L934-L952)：

```
1. 找到第一个'$'分隔符（changeset 格式是 ops$charBank，属性引用只出现在 ops 部分）
2. 对 ops 部分跑正则 /\*([0-9a-z]+)/g 抓所有属性编号
3. 对每个编号：
   · wireApool.getAttrib(oldNum) → 取出 [key, value]
   · 若找不到 → return ''（该属性被丢弃，对应 timeslider 删除场景）
   · apool.putAttrib(pair) → 在本地池子里创建/查询到新编号 newNum
   · 用 '*' + numToString(newNum) 替换原来的引用
4. charBank（$ 之后的原文）不动，因为它不含属性编号
```

所以八章 8.2 的"六步调用链"实际应为**七步**：在步骤 1 `applyToAText(c, baseAText)` 之前，必须先完成"0. 属性池坐标归一化"。属性池翻译是 OT 对齐能跑通的前提。

### 11.3 串行队列只罩三类包，USER/CHAT 走内联：对重连时序的影响

此前六章只说"消息进 queue"，但实际 `handleMessageFromServer` 里进 `serverMessageTaskQueue` 的只有**精确的三类**，其余全部同步处理。完整分区如下（对照 [collab_client.ts:209-323](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L209-L323)）：

| 消息类型 | 是否进串行队列 | 处理位置 |
|----------|---------------|---------|
| NEW_CHANGES | ✅ 进 | L209-227：await 输入法合成 → 校验 rev → applyChangesToBase |
| ACCEPT_COMMIT | ✅ 进 | L228-241：校验 rev → acceptCommit() |
| CLIENT_RECONNECT | ✅ 进 | L242-267：按序补回历史，到达 headRev 解封闸门 |
| USER_NEWINFO | ❌ 内联同步 | L268-278：直接写 userSet + 立即更新 ACE 作者颜色 |
| USER_LEAVE | ❌ 内联同步 | L279-286：直接 delete userSet + 褪色 |
| CLIENT_MESSAGE | ❌ 内联同步 | L287-288：直接 call onClientMessage 插件钩子 |
| CHAT_MESSAGE | ❌ 内联同步 | L289-290：直接 addMessage 渲染到聊天框 |
| CHAT_MESSAGES | ❌ 内联同步 | L291-310：批量加载历史聊天 |
| `handleClientMessage_${type}` hook | ❌ 外于队列 | L323：**队列外同步触发**，哪怕该类型进了队列 |

#### 对重连时序的具体影响

这些"逃逸"的包会和队列里的三类包发生顺序交错，带来三个可见的用户体验后果：

**1. "张三来了"比"张三写的字"先出现在屏幕上**

用户列表更新（USER_NEWINFO）是同步的，而张三的编辑内容要等 CLIENT_RECONNECT 队列按序跑完。温和重连补回 100 条 revision 期间，用户列表先于正文更新，看起来像"人已经在那了但字还没浮现"。这是有意设计——元数据不影响协作正确性，提前显示降低用户困惑。

**2. 聊天消息会穿插在补回的编辑中间**

如果在重连期间有人在聊天窗发消息，CHAT_MESSAGE 同步执行，聊天框里会立刻看到。但同一位作者在同一时段写的正文还在 CLIENT_RECONNECT 队列里排队。不影响数据正确性，只是"聊天快、正文慢"。

**3. handleClientMessage_NEW_CHANGES hook 先于 applyChangesToBase 执行**

这是最容易坑插件作者的一条。由于 L323 在所有分支之后同步执行，而且 NEW_CHANGES 队列任务是 `.then()` 异步调度的，事件顺序为：

```
1. 收到 NEW_CHANGES → 同步入队（同步创建 promise，但 fn 还没跑）
2. 同步走 L323 → hooks.callAll('handleClientMessage_NEW_CHANGES', ...)
3. 下个微任务 → serverMessageTaskQueue 里的 fn() 才执行 → applyChangesToBase
```

因此插件 hook 如果读取 DOM，会发现"正文还是旧的，changeset 还没 apply"。正确做法是在 hook 里也 await 或者监听后续事件。这个时序陷阱同样适用于 ACCEPT_COMMIT 和 CLIENT_RECONNECT。

### 11.4 stale tab 被踢 userdup：在 reconnect 入口之前执行

上一节 10.2.3 讲了"异步等待期间断线"，但更常见的 stale tab（同浏览器同作者旧 tab）被踢的路径在更前面，且**对 reconnect 和首次连接都生效**。代码在 [PadMessageHandler.ts:1174-1197](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1174-L1197)：

```
位置：L1174-1197，在 handleClientReady 的 message.reconnect 判断（L1208）之前执行。
```

#### 踢人的精确条件

同时满足**全部四条**才会触发：

| 条件 | 代码位置 | 含义 |
|------|---------|------|
| ① `user == null` | L1181 | 未走认证：无 basic auth / 无 SSO / 无 getAuthorId 插件映射作者ID |
| ② `!sessionInfo.embed` | L1181 + L1190 | 当前 socket 不是 iframe 内嵌的 timeslider 等嵌入场景 |
| ③ room 中存在 otherSocket | L1180 + L1182 | pad 房间里还有别的连接 |
| ④ otherSocket.author === 本 socket.author 且 `!sinfo.embed` | L1190 | 对方和我是同一作者（同一浏览器 cookie 派生 authorID），且对方也不是嵌入 |

L1174-1179 整段长注释解释了设计：cookie 派生的 authorID 是**per-browser** 的，同浏览器开两个 tab 打开同一个 pad 会得到相同 author。历史上这一直是 stale tab 的来源——用户打开新 tab 编辑，但旧 tab 还挂着。两个 tab 都声称"我是 a.john"会导致协作状态错乱（双向提交会把自身 changeset 当成外部变更）。

#### 执行顺序

```
handleClientReady 入口（L1128）
  ↓
L1137-L1165: await padManager.getPad()、拉取作者信息等（多个 await）
  ↓
L1170: sessionInfo 一致性检查（防止等 DB 期间断线）
  ↓
L1174-L1197: ★ 遍历 room 里同 author 的旧 socket，踢掉
                · sessioninfos[otherSocket.id] = {}
                · otherSocket.leave(padId)
                · otherSocket.emit('message', {disconnect: 'userdup'})
  ↓
L1208: 分 message.reconnect ? 分支 : 首次连接分支
```

**关键点：reconnect 和首次连接都会走到这段踢人逻辑。** 温和重连场景下，如果旧 tab 在断线期间，用户在新 tab 里继续编辑（新 tab 进了房间），那么旧 tab 重连发送 CLIENT_READY + reconnect=true 时，就会在 L1174 被新 tab 的存在触发踢人——给旧 tab 发 userdup 断连，旧 tab 端编辑器禁用，用户需要关掉。反过来，如果旧 tab 先重连成功，用户在新 tab 点刷新，新 tab 的首次 CLIENT_READY 会把旧 tab 踢掉。

#### 客户端 userdup 的处理

在 [pad.ts:478-487](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L478-L487)：
```
if (obj.disconnect) {
  padconnectionstatus.disconnected(obj);
  padeditor.ace.setEditable(false);
}
```
`padconnectionstatus.disconnected('userdup')` 会在白名单里匹配到"重复会话 / 你在另一个标签页打开了同一文档"的文案。注意 userdup 路径**不会触发自动重连倒计时**（因为再连还会被踢，会陷入踢-连-踢循环）。

#### 有认证用户不会被踢

L1181 的 `if (user == null && !sessionInfo.embed)` 是关键开关。如果用户通过 basic auth / SSO 登录且配置了 `getAuthorId` hook 把 username 映射到稳定 authorID，那么 `req.session.user` 存在 → `user != null` → 跳过整段踢人逻辑。这样同一账号在两台设备上并发编辑是被允许的（此时 authorID 虽然相同但来自不同 socket，不构成 stale tab）。

### 11.5 ACCEPT_COMMIT 放过 newRev===rev 与 NEW_CHANGES 严格 newRev===rev+1 的不对称意图

此前二章只说了三处"都只 warn 不抛"，但其实校验的严格程度不对称，对照 [collab_client.ts:220,234,252](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L220-L252)：

```
NEW_CHANGES      :  if (newRev !== (rev + 1))       // 严格：必须 +1
ACCEPT_COMMIT    :  if (![rev, rev + 1].includes(newRev))   // 宽松：等于 rev 也可以
CLIENT_RECONNECT :  if (newRev !== (rev + 1))       // 严格：必须 +1
```

#### ACCEPT_COMMIT 为何允许相等

代码里附了两行长注释（L231-233），完整覆盖三种场景：

```
// newRev will equal rev if the changeset has no net effect (identity changeset, removing
// and re-adding the same characters with the same attributes, or retransmission of an
// already applied changeset).
```

| 场景 | 发生时机 | newRev === rev 的理由 |
|------|---------|---------------------|
| ① identity changeset | 用户删除一段文本后立即原样粘回，且格式/属性完全一致 | 服务端合并结果文档没变化，rev 不增长（pad.appendRevision 内部会判空） |
| ② 移除并重新添加相同字符+属性 | 敲"a"再删除"a"的来回两次操作被合并为最终 changeset 净效果为 0 | 同上，服务端最终 changeset 是 identity，rev 不长 |
| ③ 重传已应用的 changeset | 重连时 CLIENT_RECONNECT 里作者是自己 → acceptCommit() 先把 rev 推了，旧的 ACCEPT_COMMIT 随后到达（socket 乱序）| rev 已被推到 newRev，所以相等 |

第三种场景在重连里极其常见：客户端断线前发了一份 USER_CHANGES，服务端接受后本来要回 ACCEPT_COMMIT，但此时断了；重连恢复后，服务端用 CLIENT_RECONNECT 把这条 revision 补回来，客户端走"作者是自己"分支 acceptCommit() 把 rev 推上去；等旧的 ACCEPT_COMMIT 再到的时候 newRev 已经和 rev 相等。如果这时候严格判断，就会"明明成功了却警告 bad revision"。

#### NEW_CHANGES / CLIENT_RECONNECT 为何严格 +1

这两类消息的共同特征是**代表文档状态的新增外部修订**，每条 revision 都必须严格单调且无间隔地应用到 baseAText 上：

- 如果 `newRev <= rev`：这是旧消息重播（或乱序到了），changeset.oldLen 和当前 baseAText 长度不匹配，直接 apply 会崩。跳过，让后续消息继续推进对齐。
- 如果 `newRev > rev + 1`：中间漏了 revision，同样 changeset.oldLen 不是当前 baseAText 的长度，会崩。跳过，靠 CLIENT_RECONNECT 重新补回或后续消息重试兜。

ACCEPT_COMMIT 之所以可以宽容，是因为**它不承载外部文档内容**——它只是告诉客户端"你发的那份我确认了"，真正把 changeset 合并进 baseAText 的是 `acceptCommit()` 里调的 `applyPreparedChangesetToBase()`，用的是客户端自己缓存的 submittedChangeset（肯定 oldLen 对得上），所以 ACCEPT_COMMIT 的 newRev 只做版本号推进，不参与 OT 计算。这就是不对称性的根本原因：

- **NEW_CHANGES / CLIENT_RECONNECT**：changeset 来自外部，oldLen 依赖 rev 严格匹配，必须严格 newRev = rev+1。
- **ACCEPT_COMMIT**：changeset 在客户端本地缓存（submittedChangeset），oldLen 永远对得上，newRev 只用来推进版本计数器，可以宽容 newRev ∈ {rev, rev+1}。

#### 对重连时序的影响

正因为 ACCEPT_COMMIT 的宽松，在"CLIENT_RECONNECT 里已经 acceptCommit 推了 rev → 旧 ACCEPT_COMMIT 到达"的交织场景下不会误报，rev 保持不变即可。这是整个重连系统能稳定运行而不被乱序打断的关键细节之一。
