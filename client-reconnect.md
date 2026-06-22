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

---

## 十二、五处补充细节校正

### 12.1 强制重连表单与 missedChanges：到底走哪条服务端路径

上一版文档里有一处推断错误——我假设 `pad.forceReconnect()` → `form#reconnectform.submit()` → `POST /ep/pad/reconnect` 是主路径。对照代码发现真实情况更反直觉：

#### 两条完全分叉的路径

| 触发方式 | 代码位置 | 实际行为 |
|----------|---------|---------|
| **路径 A（占 100% 实际触发）** | [pad_connectionstatus.ts:35-37](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_connectionstatus.ts#L35-L37) | `$('button#forcereconnect').on('click', () => window.location.reload())` ——**直接刷新页面** |
| **路径 B（死代码路径）** | [pad.ts:1065-1072](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L1065-L1072) `pad.forceReconnect()` | 填三个 hidden input（padId / diagnosticInfo / missedChanges）→ `form.trigger('submit')` POST `/ep/pad/reconnect` |

在整个 src 目录里 grep `pad.forceReconnect` / `.forceReconnect(` **没有任何调用方**。它是暴露在 pad 命名空间里的外部 API（可能给插件用），但默认 UI 的"手动重新连接"按钮和倒计时触发的 `forceReconnection($modal)` [pad_automatic_reconnect.ts:100-102](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_automatic_reconnect.ts#L100-L102) 走的是 `#forcereconnect` 的 click → 直接 `location.reload()`。

#### 服务端 `/ep/pad/reconnect` 路由：根本不存在

在所有 `expressCreateServer` hook 里搜索不到对 `/ep/pad/reconnect` 的 `app.post` 注册。`tests/backend/specs/urlBasePath.ts` 只断言 HTML 模板里的 `action="/ep/pad/reconnect"` 字符串存在，并不断言这个路由实际返回 200。

#### missedChanges 的真实去向：丢失（但影响面小）

因为路径 A 是纯刷新，`getMissedChanges()` 里辛苦打包的 `{userInfo, baseRev, committedChangeset, furtherChangeset, ...}`（[collab_client.ts:428-443](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L428-L443)）**根本不会送到服务端**。表单 input 被填充了但从不被提交。

这意味着：当走强制重连（`slowcommit` / `initsocketfail` / 用户手动点按钮）时，**本地未发编辑会丢失**。用户界面用 gritter "There are unsaved changes." 提示过，但实际数据没被抢救回来——路径 B 的 `missedChanges` 抢救机制从未真正跑通过。这是一个设计上的"半截"：前端表单和 getMissedChanges 都写了，但后端路由没接上（或历史上被移除了），只剩 UI 提示。

#### 历史推断

从 HTML 模板同时存在 `form#reconnectform` 和按钮 click 直接 reload 来看，更可能的演进是：早期版本确实走表单 POST → 后端重放 missedChanges → 重定向回 pad；后来为了简化，改成直接 reload，但旧表单和 `pad.forceReconnect()` API 被作为兼容层保留下来，无人调用。

### 12.2 prepareUserChangeset：为什么要清作者归属再重新赋值

之前没提这一步——它在 OT 对齐里极其关键。代码在 [changesettracker.ts:142-163](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/changesettracker.ts#L142-L163)，只在 **`submittedChangeset` 为空**的分支执行（有 submittedChangeset 时直接 `compose(submitted, userChangeset)`，因为 submitted 已经是清洗过的作者归属）：

```
L142  const authorId = window.pad.myUserInfo.userId;
L145  // Sanitize authorship: Replace all author attributes with this user's author ID in case the
L146  // text was copied from another author.
L147  const cs = unpack(userChangeset);
L148  const assem = new MergingOpAssembler();
L150  for (const op of deserializeOps(cs.ops)) {
L151    if (op.opcode === '+') {                 // 只处理插入操作
L152      const attribs = AttributeMap.fromString(op.attribs, apool);
L153      const oldAuthorId = attribs.get('author');
L154      if (oldAuthorId != null && oldAuthorId !== authorId) {
L155        attribs.set('author', authorId);    // 强制替换成当前用户
L156        op.attribs = attribs.toString();
L157      }
L158    }
L159    assem.append(op);
L160  }
L162  userChangeset = pack(cs.oldLen, cs.newLen, assem.toString(), cs.charBank);
```

#### 为什么必须做这一步

典型场景：用户在浏览器里从别人写的段落里**复制粘贴一段带属性的文本**到自己正在编辑的位置。粘贴产生的 DOM mutation 会被 ACE 层捕获为 changeset，里面的 `+` op 带的 `author` 属性仍然是原作者的（因为 DOM 里每个 span 的 data-author 保留了来源）。

如果不清洗直接发送：
1. 服务端会认为这段文本是原作者插入的
2. 作者归属表、贡献统计、光标颜色全部错乱
3. 更糟糕的是，这条 changeset 带着别人的 author 属性被应用到 baseAText 后，下次我再编辑时，作者归属会从"我"跳回"别人"，导致撤销栈混乱

因此 prepareUserChangeset 在发送前**对所有插入操作强制将 author 属性改写成 `myUserInfo.userId`**，确保"从这台浏览器发出的改动 100% 归属于当前用户"，而不管 DOM mutation 里带的来源作者是谁。

注意：Keep（`=`）和 Delete（`-`）op 不处理——它们不产生新文本，不涉及作者归属。同时用 `MergingOpAssembler` 重打包，把相邻同属性的 op 合并回去，降低 changeset 体积。

### 12.3 follow 的 reverseInsertOrder：并发插入的光标偏好映射

在八章里我只提到"reverseInsertOrder 解决对称性破缺"，但它的真实语义和"用户光标位置"强绑定。先看 `follow` 的定义签名 [Changeset.ts:1446](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/Changeset.ts#L1446)：

```
follow(cs1, cs2, reverseInsertOrder, pool)
  · cs1 和 cs2 都基于同一 oldLen（并发）
  · 返回：把 cs2 变换到 cs1 执行后的新文档坐标
  · reverseInsertOrder === false → cs1 的插入排在 cs2 之前（cs1 被视为"更先发生"）
  · reverseInsertOrder === true  → cs2 的插入排在 cs1 之前（cs2 被视为"更先发生"）
```

在 [changesettracker.ts:99-132](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/changesettracker.ts#L99-L132) 的 applyChangesToBase 里，两处 follow 的参数是**严格对称但相反**的：

| 位置 | 调用 | reverseInsertOrder | 语义 |
|------|------|--------------------|------|
| L113 | `follow(c, oldSubmitted, false, apool)` | false | 外部变更 c 优先插入（我的旧提交在外部之后） |
| L114 | `follow(oldSubmitted, c, true, apool)` | true | 同上（反向表达），得到 c2 等价外部 c |
| L119 | `follow(c2, oldUserChangeset, true, apool)` | **true** | 我的本地打字 userChangeset **优先插入** |
| L121 | `follow(oldUserChangeset, c2, false, apool)` | **false** | 同上反向，得到 postChange |

L117 有注释 `preferInsertingAfterUserChanges = true`——意思就是：对 userChangeset（用户正在光标处输入的字）来说，**如果外部变更和我在同一位置插入，要把我的字放在前面**。这样用户的光标不会跳到插入内容后面，不会"打字时光标被别人的字推着走"。

反过来，submittedChangeset（已经发出去的提交）被视为"已经离开这台浏览器"，外部变更 c 的插入要排在前面——因为那份 changeset 已经不在本地控制了，交给服务端的全局序裁决，本地只需做被动 OT 变换。

#### `insertorder=first` 属性的额外层级

`follow` 内部 [Changeset.ts:1459-1489](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/Changeset.ts#L1459-L1489) 还有一层优先级：若 cs1 或 cs2 的插入 op 带 `['insertorder', 'first']` 属性（Etherpad 里按 Ctrl+Enter 换行等特殊操作会带这个），则不管 reverseInsertOrder，带这个属性的那份优先插入。这保证了"特殊换行"等操作在并发时不会被挤错位置。

### 12.4 重连复用首屏 clientVars：对齐窗口在哪

温合重连（socket.io reconnect）不会让浏览器重新渲染 HTML，所以 `window.clientVars`（首屏注入的那个大对象）原封不动保留在内存里。这带来一个有趣的"对齐窗口"问题：重连时服务端**不会重新下发 CLIENT_VARS**，而客户端所有初始化都基于首屏的那份，两者可能相差几百条 revision。

#### 代码层面的证据

**客户端侧**——[pad.ts:353-379](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad.ts#L353-L379)：
```
handshake():
  let receivedClientVars = false;   // L354
  ...
  socket.io.on('connect', () => {
    // 首次连接 receivedClientVars=false, 走 isReconnect=false
    // reconnect 事件 receivedClientVars=true, 走 isReconnect=true
    sendClientReady(receivedClientVars);   // L378
  });
  ...
  socket.on('message'):
    if (!receivedClientVars && obj.type === 'CLIENT_VARS') {
      receivedClientVars = true;           // L462 —— 仅首次置 true
      ... init collabClient using obj.data.collab_client_vars ...
    }
```

**服务端侧**——[PadMessageHandler.ts:1208-1210](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1208-L1210)：
```
if (message.reconnect) {
  // If this is a reconnect, we don't have to send the client the ClientVars again
  socket.join(sessionInfo.padId);
  ... 只发 CLIENT_RECONNECT 补历史修订 ...
} else {
  // first connect — 拉 atext、apool、作者、插件等组装完整 CLIENT_VARS 下发
}
```

#### 对齐窗口的含义与边界

首屏 `clientVars.collab_client_vars` 里包含：`initialAttributedText`（首屏 pad 文本快照）、`rev`（快照对应的修订号）、`apool`（首屏属性池）、历史作者信息等。重连时：

1. `initialAttributedText` 和 `rev` —— 不重要。collabClient 内部的 `baseAText` 和 `rev` 已经在用户在线编辑期间持续被 `applyChangesToBase` / `acceptCommit` 推进到当下值，首屏快照早就不用了。
2. `apool` —— **关键**。客户端本地 apool 是首屏池 + 每次 NEW_CHANGES / prepareUserChangeset 增量添加。重连后，CLIENT_RECONNECT 每条消息都会附带消息内 changeset 用到的精简 apool，并通过 `moveOpsToNewPool()`（11.2 节）把引用编号翻译到本地池，所以不会冲突。
3. 作者颜色、用户名、插件配置等静态元数据 —— 重连时如果服务端这些值变了（比如管理员改了用户颜色），客户端不会更新。这是已知的对齐窗口限制，但这些元数据改变频度极低，下次全页刷新就会自然同步。

所以"对齐窗口"就是：首屏 clientVars 是重连时的**只读基准元数据**，所有动态状态（文本、rev、属性映射）都已经由客户端自己的协作循环推进了，服务端只需补 CLIENT_RECONNECT 修订历史，无需重新下发整份 CLIENT_VARS——这是温合重连能在几百毫秒内恢复的核心优化。

### 12.5 自动重连的一次性 latch：`socket.once('connect')`

在强制重连倒计时过期时，代码 [pad_automatic_reconnect.ts:87-98](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_automatic_reconnect.ts#L87-L98) 有个容易被忽略的分支：

```
waitUntilClientCanConnectToServerAndThen(callback, pad):
  whenConnectionIsRestablishedWithServer(callback, pad);
  pad.socket.connect();   // 主动触发 socket.io 发起连接

whenConnectionIsRestablishedWithServer(callback, pad):
  // only add listener for the first try, don't need to add another listener
  // on every unsuccessful try
  if (reconnectionTries.counter === 1) {
    pad.socket.once('connect', callback);    // 一次性 latch
  }
```

#### 为什么是 `counter === 1`

`reconnectionTries.counter` 是指数退避计时器的全局变量：
- 第一次倒计时结束 → `counter = 1` → 注册 `socket.once('connect', callback)`，并立即 `pad.socket.connect()`
- 第二次（翻倍等待后）→ `counter = 2` → 不重复注册，因为第一次注册的 `once` 监听器**还在**（除非第一次 connect 已经成功触发过）

这是个一次性 latch（门锁）：只在第一次倒计时结束时注册一个 `connect` 回调，后续每轮倒计时只是 `pad.socket.connect()` 让底层再试一次，而 callback 只等**第一次成功 connect** 触发。

#### 这个 callback 是什么

往上追溯 [pad_automatic_reconnect.ts:59-65](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/pad_automatic_reconnect.ts#L59-L65)，onExpire 的 callback 是 `() => forceReconnection($modal)`。因此整条链路：

```
倒计时到 0
  → 若 counter===1：注册 socket.once('connect', () => forceReconnection($modal))
  → 调 pad.socket.connect()
  → 若 connect 成功：callback 触发 → $('#forcereconnect').click() → location.reload()
  → 若 connect 失败：socket.io 内部指数退避继续试，下一轮倒计时结束又调一次 pad.socket.connect()
```

#### 设计意图

**避免重复 reload**：如果不做 latch，每一轮倒计时都注册一个新的 `once('connect')` → 某次 connect 成功时 N 个回调同时触发 → N 次 reload 狂刷页面。counter===1 的条件保证全页面生命周期内最多只注册一个一次性监听器，成功时只触发一次 forceReconnection。

但这里也有个隐藏的"死锁"边界：如果 counter 被加到 2 以上（意味着至少一轮 connect 失败），然后用户手动把网线插上，socket.io 自动 connect 成功，但 latch 监听器在 counter=1 时就注册过了，仍然会正常触发 forceReconnection —— 因为 `once` 只会触发一次，和 counter 的当前值无关。所以最终效果是对的：只要任何一次 connect 成功，就会立即 reload 页面，用户不需要等到下一次倒计时。

---

## 十三、服务端重连 vs 首连：四处不对称细节

### 13.1 socket.join 时序与 updatePadClients 兜底的不对称

`handleClientReady` 函数（[PadMessageHandler.ts:1128-1509](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1128-L1509)）的两条分支在 join 时机和兜底策略上有显著差异：

#### 首次连接（else 分支，L1270-1445）

```
顺序：
  1. 准备 atext / apool / headRev / clientVars 大对象
  2. 跑 clientVars hook（多次 await，可能耗时长）
  3. socket.join(sessionInfo.padId)          ← 在这里才入房  [L1422]
  4. socket.emit('message', CLIENT_VARS)      ← 发首屏数据
  5. sessionInfo.rev = headRev                 ← 记录当前修订号
  6. sessionInfo.time = await pad.getRevisionDate(headRev)
  7. await exports.updatePadClients(pad)      ← 兜底：补上入房前漏的修订  [L1444]
```

L1441-1444 的注释讲得很清楚：
> "Flush any revisions that may have been appended while we were awaiting the clientVars hook (before socket.join). Those revisions were broadcast to existing room members but this socket hadn't joined yet so it missed them."

因为 clientVars hook 期间有多个 await，在这期间如果有协作者推进了修订，那些修订会通过 `updatePadClients()` 广播给已经在房间里的人，但新 socket 还没 join，所以收不到。因此首连分支在 join + 设好 sessionInfo.rev 之后，**显式调用一次 updatePadClients 做兜底**，把漏掉的 revision 以 NEW_CHANGES 形式补回来。

#### reconnect 分支（L1208-1269）

```
顺序：
  1. socket.join(sessionInfo.padId)          ← 第一时间入房  [L1211]
  2. sessionInfo.rev = message.client_rev     ← 用客户端自称的修订号
  3. 自行拉取 revisionsNeeded（client_rev+1 .. head）
  4. 逐条 socket.emit CLIENT_RECONNECT
  5. （没有 updatePadClients 兜底调用）
```

reconnect 分支**没有显式调用 updatePadClients 兜底**，原因是它采用了不同的策略：
- join 放在最前面，尽早开始接收后续的 NEW_CHANGES 广播；
- 历史修订由 CLIENT_RECONNECT 自行负责，按顺序逐条发送；
- 在 await 拉取 revision 数据期间，如果有新编辑发生，`updatePadClients()` 会自动把 NEW_CHANGES 推给这个 socket（因为它已经 join 了房间，且 sessionInfo.rev 已设置）。

#### 不对称带来的乱序窗口

reconnect 分支的"先 join 再补历史"策略会引入一个乱序窗口：

**场景**：客户端断线时 rev=100，服务端当前 head=150。重连时在 await 拉取 rev 数据期间又有新编辑推到 152。

到达客户端的消息顺序可能是：
```
CLIENT_RECONNECT newRev=101  ← 历史补回，按序
CLIENT_RECONNECT newRev=102  ← 历史补回，按序
...
NEW_CHANGES    newRev=151   ← 新编辑，从 updatePadClients 广播过来
NEW_CHANGES    newRev=152   ← 新编辑
...
CLIENT_RECONNECT newRev=120  ← 历史补回的后面部分还在继续发
CLIENT_RECONNECT newRev=121  ← （因为 CLIENT_RECONNECT 是同步逐条 emit，在事件循环里排在 updatePadClients 之后）
```

不对——实际上 CLIENT_RECONNECT 的 for 循环（L1250-1261）是同步 emit 的，它和 updatePadClients 里的 Promise.all + map 是并发关系。如果 CLIENT_RECONNECT 先发到 140，然后 updatePadClients 的 NEW_CHANGES 到了 151、152，然后 CLIENT_RECONNECT 继续发 141…150，那么客户端会收到：140 → 151 → 152 → 141 → 142 → … → 150。

客户端的 `serverMessageTaskQueue` 保证**按到达顺序**串行处理。NEW_CHANGES 分支要求严格 `newRev === rev + 1`，所以当 newRev=151 到达而 rev 才到 140 时，会 `console.warn` 然后跳过（见十一章 11.5）。后面的 CLIENT_RECONNECT newRev=141 会正常推进 rev。

这是一个已知的乱序窗口：中间部分 NEW_CHANGES 会被短暂跳过并打 warning，但最终 CLIENT_RECONNECT 会把它们补回来。由于 NEW_CHANGES 和 CLIENT_RECONNECT 都会调用 `applyChangesToBase` 把 changeset 应用到 baseAText 上，且内容完全相同，所以数据是一致的——只是用户会看到"先跳到后面的编辑，又退回来"的视觉跳变，以及控制台的几条 warning。

### 13.2 updatePadClients：NEW_CHANGES 的统一出口

`updatePadClients`（[PadMessageHandler.ts:1008-1066](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1008-L1066)）是 Etherpad 协作层**唯一的 NEW_CHANGES 广播出口**。所有导致 pad 内容变化的操作最终都调用它。

#### 核心逻辑

```
1. 取房间内所有 sockets：_getRoomSockets(pad.id)
2. 如果没人，直接 return（L1011）
3. 建一个 revCache 缓存本次循环用到的 revision 对象（避免重复查 DB）
4. 对每个 socket 并行处理（Promise.all）：
   · 取 sessioninfo.rev
   · while sessioninfo.rev < head:
       o 取下一条 revision
       o prepareForWire 翻译属性池
       o socket.emit(NEW_CHANGES, {newRev, changeset, apool, author, time, timeDelta})
       o sessioninfo.rev = r  // 推进该 socket 的记录
       o sessioninfo.time = currentTime
```

关键特性：
- **按 socket 独立推进**：每个 socket 有自己的 `sessioninfo.rev`，互不影响。慢的 socket 不会拖慢快的 socket。
- **timeDelta 计算**：`timeDelta = currentTime - sessioninfo.time`，客户端用它来校准协作时间线。
- **revCache 共享**：所有 socket 共享一个 revCache Map，第一条读到的 revision 会被后面的 socket 复用，大幅减少 DB 查询。
- **容错**：单 socket emit 失败（比如连接已断）只打 error log 并 return，不影响其他 socket。

#### 所有调用方

| 调用方 | 位置 | 触发时机 |
|--------|------|---------|
| `handleUserChanges`（用户编辑提交） | [L997](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L997) | 每次成功 apply 一条用户 changeset |
| `handlePadDelete` 间接（通过 pad.remove） | —— | pad 删除后 padChannels 自动清理 |
| `ImportHandler.reloadPad` / `setText` | [ImportHandler.ts:190,310](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/ImportHandler.ts#L190) | 导入文件替换 pad 内容后 |
| `API.copyPad` / `movePad` / `deletePad` / `restoreRevision` | [API.ts:242,265,332,670](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/db/API.ts#L242) | HTTP API 修改 pad 内容后 |
| 首连兜底 | [L1444](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1444) | 首次连接入房前漏的修订 |

注意：**reconnect 分支的 CLIENT_RECONNECT 不经过 updatePadClients**——它直接从 pad 拉取 revision 并逐条 emit，走的是一条独立的补历史路径。这也是 13.1 节提到的不对称性来源。

### 13.3 USER_NEWINFO：重连也无条件广播

在 `handleClientReady` 函数的末尾（[L1447-1508](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1447-L1508)），有两组 USER_NEWINFO 广播和一个 userJoin hook 调用，它们**在 if-else 块之外**——也就是说 **reconnect 和首次连接都会执行**。

#### 两组广播

**第一组：向其他人广播"我来了"**（L1447-1458）
```
socket.broadcast.to(padId).emit('message', {
  type: 'USER_NEWINFO',
  userInfo: { colorId, name, userId: sessionInfo.author }
});
```

**第二组：向我广播"其他人都在"**（L1460-1499）
```
Promise.all(_getRoomSockets(pad.id).map(roomSocket => {
  if (roomSocket.id === socket.id) return;
  ... 查 authorInfo ...
  socket.emit('message', { type: 'USER_NEWINFO', userInfo: {...} });
}));
```

还有第三处：`userJoin` hook（L1501-1508），插件可以监听它做额外处理。

#### 重连场景下的影响

因为 reconnect 也会触发这些广播，重连时：
- **其他客户端**：会收到一份该用户的 USER_NEWINFO。由于客户端用户集合是按 userId 去重的，收到重复 USER_NEWINFO 是幂等操作——只是更新一下名字/颜色，不会多一个用户头像。副作用是用户列表里该用户可能会"闪一下"（先消失再出现），但这取决于前端实现（[collab_client.ts:268-278](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/static/js/collab_client.ts#L268-L278) 是直接 `set()` 覆盖，没有消失动画）。
- **重连的客户端自己**：会收到一份完整的在线用户列表。这很重要——因为断线期间可能有人加入或离开，如果不重发，客户端的用户列表就过期了。CLIENT_VARS 不重发，但 USER_NEWINFO 的"完整快照"相当于补上了用户列表这一项。

#### 与 USER_LEAVE 的配合

断线时 Socket.IO 的 `disconnect` 事件会触发 `handleDisconnect`（[L245-287](file:///d:/fz/0601-2/solo-dogfeeding/code/75-etherpad-lite/src/node/handler/PadMessageHandler.ts#L245-L287)），其中有个 `isLastSocketForAuthor` 判断：只有该作者的**最后一个 socket** 离开时才广播 USER_LEAVE。这和 11.4 节讲的"stale tab userdup 踢人"是同一个设计思路——允许多设备同账号并发。

因此实际时序是：
```
用户断网
  → 服务端 detect disconnect
  → 如果是该作者最后一个 socket：广播 USER_LEAVE
  → （几秒后）用户网络恢复，socket.io reconnect
  → 客户端发 CLIENT_READY + reconnect=true
  → 服务端 handleClientReady
  → 广播 USER_NEWINFO（作者重新出现）
  → 向客户端发完整在线用户列表
  → 发 CLIENT_RECONNECT 补历史修订
```

如果用户只是"短暂闪断"（浏览器没感知到，socket.io 快速重连），且 disconnect 事件还没来得及触发（或不是最后一个 socket），那么 USER_LEAVE → USER_NEWINFO 的循环就不会发生，用户列表完全不动。

### 13.4 CLIENT_VARS 不重发：哪些有运行时通道，哪些只能刷新

温合重连时服务端明确不重发 CLIENT_VARS（L1209 注释："we don't have to send the client the ClientVars again"）。clientVars 是一个很大的对象，首屏注入时包含了 pad 的元信息、配置、插件变量等。重连时哪些能动态同步、哪些必须等页面刷新，整理如下：

#### 有运行时同步通道的（重连会自动对齐）

| 数据 | 同步通道 | 对应消息类型 |
|------|---------|-------------|
| 文档文本内容 | 协作层 | NEW_CHANGES / CLIENT_RECONNECT |
| 属性池（apool）编号映射 | 协作层 | 每条 NEW_CHANGES / CLIENT_RECONNECT 附带精简 apool + moveOpsToNewPool 翻译 |
| 在线用户列表 | 协作层 | USER_NEWINFO（完整快照）/ USER_LEAVE |
| 用户名字/颜色 | 协作层 | USER_NEWINFO |
| 聊天消息 | 聊天模块 | CHAT_MESSAGE / CHAT_MESSAGES（客户端可主动 GET_CHAT_MESSAGES 拉历史） |
| 插件自定义消息 | 插件 hook | CLIENT_MESSAGE |
| 协作者光标/选区 | 插件 | 通常走 CLIENT_MESSAGE，由 ep_cursor_list 等插件实现 |

#### 只能等页面刷新的（重连不会同步）

| 数据 | 在 clientVars 中的位置 | 为什么不同步 |
|------|----------------------|-------------|
| **插件列表及插件 clientVars 注入** | `pad.plugins` + 各插件通过 clientVars hook 注入的字段 | 重连不跑 `clientVars` hook，插件新增/移除的运行时变量不会更新 |
| pad 基本配置（`padOptions`） | `padOptions` | 服务端配置改了不会推给已连接客户端 |
| 删除权限 | `deletePadEnabled` | 只在首屏计算一次（基于 isCreator、token、allowPadDeletionByAllUsers） |
| 账号权限 | `accountPrivileges` | 同上，首屏一次性计算 |
| 只读模式 | `readOnly` / `readOnlyPadId` | 只读状态切换必须刷新页面 |
| 保存的修订列表 | `savedRevisions` | 没有运行时推送通道，只能刷新或手动触发 |
| UI 配置（滚动行为等） | `scrollWhenFocusLineIsOutOfViewport` | 纯前端配置，改 settings.json 后需刷新 |
| Cookie 前缀、环境模式 | `cookiePrefix` / `mode` | 环境信息，生命周期内不变 |
| cookieConsent 同意状态 | `cookieConsent` | 首屏一次性注入 |
| 自动重连配置 | `automaticReconnection` | 首屏一次性注入 |
| 客户端 IP | `clientIp` | 首屏注入，重连后 IP 不变（同连接） |

#### 一个容易忽略的坑：插件的 clientVars hook

重连时 `clientVars` hook 不跑，意味着插件如果在 hook 里注入了动态数据（比如实时统计、在线人数等），重连后这些数据不会更新。插件作者如果需要重连时刷新数据，需要自己监听 socket 的 reconnect 事件主动拉取，或者通过 CLIENT_MESSAGE 自定义通道推送。

这也是为什么 13.3 节 USER_NEWINFO 的"重连也广播完整用户列表"这么重要——用户列表是 CLIENT_VARS 里少数几个在重连时有专门同步通道的动态元数据。
