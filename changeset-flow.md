# Changeset 合并与广播流程分析

## 概述

Etherpad 使用 **操作转换 (Operational Transformation, OT)** 算法实现多人实时协作编辑。核心数据结构是 `Changeset`，它描述了对文档的一次变更（插入、删除、属性变更）。整个协作流程围绕 changeset 的生成、提交、合并、广播展开。

## 核心概念

### Changeset 三层状态（客户端）

客户端维护三层文档状态，确保本地编辑流畅性与服务端一致性：

| 层级 | 变量 | 说明 |
|------|------|------|
| 服务端基准 | `baseAText` | 最新的已确认服务端版本 |
| 已提交待确认 | `submittedChangeset` | 已发送给服务器、等待 ACCEPT_COMMIT 的变更 |
| 用户未提交 | `userChangeset` | 用户最新编辑产生、尚未发送的变更 |

三层之间的关系：
- `baseAText + submittedChangeset + userChangeset = 用户当前看到的文档`

### 核心操作函数

| 函数 | 作用 | 典型场景 |
|------|------|----------|
| `compose(cs1, cs2)` | 串联两个 changeset，cs2 基于 cs1 的结果 | 合并多个连续变更 |
| `follow(cs1, cs2)` | 操作转换：将 cs2 转换为在 cs1 之后可应用的形式 | 并发变更合并/重基 |
| `applyToAText(cs, atext)` | 将 changeset 应用到带属性文本 | 更新文档状态 |
| `identity(N)` | 生成恒等 changeset（无实际变更） | 占位/初始化 |

核心文件：[Changeset.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/Changeset.ts)

---

## 一、本地变更生成与提交

### 1.1 变更追踪器 (ChangesetTracker)

位置：[changesettracker.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/changesettracker.ts)

`makeChangesetTracker()` 是客户端的核心状态管理器，维护三层文档状态。

#### 用户输入时的变更累积

当用户在编辑器中输入文字时，编辑器调用 `composeUserChangeset(c)` 将新产生的变更累加到 `userChangeset`：

```javascript
composeUserChangeset: (c) => {
  if (!tracking) return;
  if (applyingNonUserChanges) return;
  if (isIdentity(c)) return;
  userChangeset = compose(userChangeset, c, apool);  // 叠加到 userChangeset
  setChangeCallbackTimeout();  // 触发变更通知
}
```

关键点：
- 使用 `compose()` 将新变更追加到 `userChangeset`
- `applyingNonUserChanges` 标志防止递归触发（应用他人变更时不触发回调）
- 通过 `setChangeCallbackTimeout()` 异步通知 collab_client

#### 准备提交：prepareUserChangeset()

当 collab_client 决定提交时，调用 `prepareUserChangeset()`：

```javascript
prepareUserChangeset: () => {
  let toSubmit;
  if (submittedChangeset) {
    // 如果之前有已提交但未确认的，合并在一起重新提交
    toSubmit = compose(submittedChangeset, userChangeset, apool);
  } else {
    // 清洗作者信息（防止复制粘贴带来的错误作者）
    // ...遍历所有插入操作，替换 author 属性为当前用户
    toSubmit = userChangeset;
  }

  if (toSubmit) {
    submittedChangeset = toSubmit;  // 移入"已提交"层
    userChangeset = identity(newLen(toSubmit));  // userChangeset 重置为恒等
  }

  // prepareForWire: 转换属性池编号，生成可传输格式
  const forWire = prepareForWire(cs, apool);
  return { changeset: forWire.translated, apool: forWire.pool.toJsonable() };
}
```

关键点：
- 如果有 `submittedChangeset`（说明上次提交还没确认），就用 `compose()` 合并后重新提交
- 提交后，`userChangeset` 重置为 identity，`submittedChangeset` 保存待确认的变更
- `prepareForWire()` 将本地属性池编号转换为独立的 wire 格式，方便服务端合并

### 1.2 协作客户端 (CollabClient)

位置：[collab_client.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/collab_client.ts)

`getCollabClient()` 管理与服务端的通信，控制提交节奏。

#### 提交流程：handleUserChanges()

```javascript
const handleUserChanges = () => {
  // ... 连接状态检查 ...
  
  if (committing) {
    // 正在提交中，稍后再检查
    setTimeout(handleUserChanges, 3000);
    return;
  }

  // 节流：距离上次提交至少 commitDelay(500ms)
  const earliestCommit = lastCommitTime + commitDelay;
  if (now < earliestCommit) {
    setTimeout(handleUserChanges, earliestCommit - now);
    return;
  }

  if (!isPendingRevision) {  // 没有待接收的服务端变更
    const userChangesData = prepareUserChangeset();
    if (userChangesData.changeset) {
      lastCommitTime = now;
      committing = true;
      stateMessage = {
        type: 'USER_CHANGES',
        baseRev: rev,           // 基于哪个修订版
        changeset: userChangesData.changeset,
        apool: userChangesData.apool,
      };
      sendMessage(stateMessage);  // 发送到服务端
    }
  }
};
```

提交策略：
- **节流提交**：默认 500ms 间隔，防止过于频繁的网络请求
- **单请求模式**：同一时间只有一个在途提交（`committing` 标志）
- **baseRev**：携带当前客户端已知的最新修订号，服务端据此判断是否需要重基

#### 提交确认：acceptCommit()

收到服务端的 `ACCEPT_COMMIT` 消息后：

```javascript
const acceptCommit = () => {
  editor.applyPreparedChangesetToBase();  // 将 submittedChangeset 并入 baseAText
  stateMessage = null;
  setStateIdle();  // committing = false
  handleUserChanges();  // 立即检查是否有新的待提交变更
};
```

`applyPreparedChangesetToBase()` 在 changesettracker 中的实现：

```javascript
applyPreparedChangesetToBase: () => {
  baseAText = applyToAText(submittedChangeset, baseAText, apool);
  submittedChangeset = null;
}
```

---

## 二、服务端变更合并

### 2.1 消息入口与串行队列

位置：[PadMessageHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/node/handler/PadMessageHandler.ts)

服务端通过 `Channels` 类确保**同一 pad 的变更串行处理**：

```javascript
const padChannels = new Channels((ch, {socket, message}) => handleUserChanges(socket, message));
```

在 `handleMessage` 中，`USER_CHANGES` 类型消息被推入队列：

```javascript
case 'USER_CHANGES':
  stats.counter('pendingEdits').inc();
  await padChannels.enqueue(thisSession.padId, {socket, message});
  break;
```

**为什么要串行？**
- OT 算法要求对同一文档的变更必须按确定顺序处理
- 并行处理会导致竞态条件，修订号混乱

### 2.2 核心合并逻辑：handleUserChanges()

`handleUserChanges()` 是服务端协作的核心，执行以下步骤：

#### 步骤 1：安全校验

```javascript
// 1. 校验 changeset 语法正确性
checkRep(changeset);

// 2. 校验作者属性：插入操作必须是当前用户，不能冒充他人
for (const op of deserializeOps(unpack(changeset).ops)) {
  const opAuthorId = AttributeMap.fromString(op.attribs, wireApool).get('author');
  if (opAuthorId && opAuthorId !== thisSession.author) {
    // '=' 操作允许恢复其他作者（用于撤销清除作者）
    // '+' 操作必须是当前用户
    if (op.opcode !== '=') {
      throw new Error(`Author ${thisSession.author} tried to submit changes as author ${opAuthorId}`);
    }
  }
  // 插入操作必须携带 author 属性
  if (op.opcode === '+' && !opAuthorId) {
    throw new Error('insert op without an author attribute');
  }
}
```

#### 步骤 2：属性池迁移

将客户端的属性池编号映射到服务端全局属性池：

```javascript
let rebasedChangeset = moveOpsToNewPool(changeset, wireApool, pad.pool);
```

#### 步骤 3：重基 (Rebase) — 核心 OT 操作

这是最关键的一步。如果客户端提交的变更不是基于最新修订版，需要逐个跳过中间的修订，使用 `follow()` 进行操作转换：

```javascript
let r = baseRev;  // 客户端提交时的基准修订号

while (r < pad.getHeadRevisionNumber()) {
  r++;
  const {changeset: c, meta: {author: authorId}} = await pad.getRevision(r);
  
  // 检测是否为重复提交（网络重传）
  if (canonicalCs === c && thisSession.author === authorId) {
    rebasedChangeset = identity(unpack(canonicalCs).oldLen);
  }
  
  // 操作转换：将客户端变更 rebasedChangeset 转换为在 c 之后可应用的形式
  // cs1 (服务端已有的变更) 在前，cs2 (客户端变更) 在后
  rebasedChangeset = follow(c, rebasedChangeset, false, pad.pool);
}
```

**follow() 函数详解**

位置：[Changeset.ts#L1446-L1565](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/Changeset.ts#L1446-L1565)

`follow(cs1, cs2, reverseInsertOrder, pool)` 实现操作转换：

- **输入**：两个基于同一旧版本的 changeset cs1 和 cs2
- **输出**：一个新的 cs2'，使得 `compose(cs1, cs2')` 等价于在 cs1 之后应用 cs2 的效果

处理规则：
1. **双方都插入** (`+` vs `+`)：按确定性规则排序（insertorder 属性、是否换行、reverseInsertOrder 参数）
2. **一方删除** (`-` vs `=`)：删除优先，被删除的部分不再保留
3. **双方都删除** (`-` vs `-`)：取交集，都删的部分抵消
4. **都保留** (`=` vs `=`)：属性合并（`followAttributes`）

#### 步骤 4：追加新修订

```javascript
const newRev = await pad.appendRevision(rebasedChangeset, thisSession.author);
```

`appendRevision()` 在 [Pad.ts#L280-L335](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/node/db/Pad.ts#L280-L335) 中实现：

```javascript
async appendRevision(aChangeset, authorId = '') {
  // 应用 changeset 到当前 atext
  const newAText = applyToAText(aChangeset, this.atext, this.pool);
  
  // 如果没有实际变更（恒等），不增加修订号
  if (newAText.text === this.atext.text && newAText.attribs === this.atext.attribs &&
      this.head !== -1) {
    return this.head;
  }
  
  copyAText(newAText, this.atext);
  const newRev = ++this.head;
  
  // 保存到数据库
  await this.db.set(`pad:${this.id}:revs:${newRev}`, {
    changeset: aChangeset,
    meta: {
      author: authorId,
      timestamp: Date.now(),
      // 关键修订（每 100 个）额外存 atext 和 pool，加速回放
      ...newRev === this.getKeyRevisionNumber(newRev) ? {pool: this.pool, atext: this.atext} : {},
    },
  });
  
  return newRev;
}
```

### 2.5 确认与广播（精确执行顺序）

处理完变更后，服务端按以下**精确顺序**执行三个关键操作（见 [PadMessageHandler.ts#L983-L986](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/node/handler/PadMessageHandler.ts#L983-L986)）：

1. **给提交者发确认**：
```javascript
// 第 1 步：发出 ACCEPT_COMMIT 消息
socket.emit('message', {
  type: 'COLLABROOM',
  data: { type: 'ACCEPT_COMMIT', newRev }
});

// 第 2 步：立即更新提交者的服务端会话 rev
// 注意：thisSession 是 sessioninfos[socket.id] 的引用（同一个对象），
// 所以修改 thisSession.rev 等价于修改 sessioninfos[socket.id].rev
thisSession.rev = newRev;
if (newRev !== r) thisSession.time = await pad.getRevisionDate(newRev);

// 第 3 步：广播给所有客户端
await exports.updatePadClients(pad);
```

**关键细节**：
- **执行顺序是严格的**：先 `socket.emit(ACCEPT_COMMIT)`，再更新 `thisSession.rev`，最后才调用 `updatePadClients()`
- **对象引用机制**：`const thisSession = sessioninfos[socket.id]` 保存的是引用而非副本，所以第 2 步对 `thisSession.rev` 的修改会立即反映在 `sessioninfos[socket.id]` 上
- 当第 3 步 `updatePadClients()` 内部遍历 `sessioninfos[socket.id]` 时，拿到的 `rev` **已经是 newRev 了**

---

## 三、广播与协作者同步

### 3.1 服务端广播：updatePadClients()

位置：[PadMessageHandler.ts#L997-L1055](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/node/handler/PadMessageHandler.ts#L997-L1055)

```javascript
exports.updatePadClients = async (pad) => {
  const roomSockets = _getRoomSockets(pad.id);
  if (roomSockets.length === 0) return;

  const revCache = {};  // 缓存已读的修订，避免重复 DB 查询

  await Promise.all(roomSockets.map(async (socket) => {
    const sessioninfo = sessioninfos[socket.id];
    if (sessioninfo == null) return;

    // 逐个发送客户端缺失的修订
    // ★ 对于提交者：由于 handleUserChanges 中已经执行过 thisSession.rev = newRev，
    //   而 thisSession 和 sessioninfos[socket.id] 是同一个对象引用，
    //   所以这里拿到的 sessioninfo.rev 已经是 newRev 了
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
      sessioninfo.rev = r;
      sessioninfo.time = revision.meta.timestamp;
    }
  }));
};
```

特点：
- 每个会话独立跟踪自己的 `rev`（已收到的修订号）
- 使用 `revCache` 缓存修订数据，避免多次数据库查询
- 按修订号顺序发送，保证客户端接收有序
- **提交者不会收到自己原始变更的 NEW_CHANGES**：因为调用 `updatePadClients()` 时，提交者的 `sessioninfo.rev` 已被设置为 `newRev`，while 条件 `sessioninfo.rev < pad.getHeadRevisionNumber()` 在无 correction 时不成立
- **有 correction 时提交者会收到 correction 的 NEW_CHANGES**：因为 pad.head = newRev + 1（比 newRev 大），while 条件成立

### 3.2 客户端接收协作者变更

在 collab_client 的 `handleMessageFromServer` 中处理 `NEW_CHANGES`：

```javascript
if (msg.type === 'NEW_CHANGES') {
  serverMessageTaskQueue.enqueue(async () => {
    const {newRev, changeset, author = '', apool} = msg;
    if (newRev !== (rev + 1)) {
      window.console.warn(`bad message revision on NEW_CHANGES: ${newRev} not ${rev + 1}`);
      return;
    }
    rev = newRev;
    editor.applyChangesToBase(changeset, author, apool);  // 应用到编辑器
  });
}
```

### 3.3 应用他人变更：applyChangesToBase()

位置：[changesettracker.ts#L99-L132](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/changesettracker.ts#L99-L132)

这是客户端处理协作者变更的核心，需要将他人的变更"穿过"本地的两层变更（submitted 和 user）：

```javascript
applyChangesToBase: (c, optAuthor, apoolJsonObj) => {
  // 1. 属性池迁移
  if (apoolJsonObj) {
    const wireApool = (new AttributePool()).fromJsonable(apoolJsonObj);
    c = moveOpsToNewPool(c, wireApool, apool);
  }

  // 2. 应用到 baseAText
  baseAText = applyToAText(c, baseAText, apool);

  // 3. 重基 submittedChangeset（如果有）
  let c2 = c;
  if (submittedChangeset) {
    const oldSubmittedChangeset = submittedChangeset;
    // submittedChangeset 需要在 c 之后应用，所以用 follow 转换
    submittedChangeset = follow(c, oldSubmittedChangeset, false, apool);
    // c2 是穿过 submittedChangeset 后的版本，用于继续重基 userChangeset
    c2 = follow(oldSubmittedChangeset, c, true, apool);
  }

  // 4. 重基 userChangeset
  const preferInsertingAfterUserChanges = true;
  const oldUserChangeset = userChangeset;
  userChangeset = follow(c2, oldUserChangeset, preferInsertingAfterUserChanges, apool);
  
  // postChange 是最终要应用到 DOM 的变更（考虑了本地未提交变更的位置调整）
  const postChange = follow(oldUserChangeset, c2, !preferInsertingAfterUserChanges, apool);

  // 5. 应用到 DOM
  applyingNonUserChanges = true;
  try {
    callbacks.applyChangesetToDocument(postChange, preferInsertionAfterCaret);
  } finally {
    applyingNonUserChanges = false;
  }
}
```

**为什么需要两次 follow？**

当有三层状态时（base → submitted → user），收到新的服务端变更 c，需要：

1. 把 `submittedChangeset` 重基到 c 之后：`follow(c, oldSubmitted, false)`
2. 计算 c 在 submitted 之后的"影像"c2：`follow(oldSubmitted, c, true)`
3. 把 `userChangeset` 重基到 c2 之后：`follow(c2, oldUser, true)`

可以理解为：他人的变更需要"穿过"本地的每一层变更，每层都做一次操作转换。

**preferInsertingAfterUserChanges 参数的含义**：
- 当双方在同一位置插入内容时，决定谁的内容在前
- `true`：用户的插入在前，他人的插入在后（符合直觉：你输入的内容在光标处，别人的插在你后面）
- 对应 `follow()` 的 `reverseInsertOrder` 参数

---

## 四、提交者 vs 协作者：消息接收与状态更新详解

> 这是整个 changeset 流程中最容易混淆的部分：当提交者的变更被服务端接受后，提交者的本地基线如何更新？是通过 ACCEPT_COMMIT 直接回基线，还是也经过 NEW_CHANGES？协作者又如何收到变更？本节沿着代码中的 `rev`（客户端）和 `sessioninfo.rev`（服务端）逐行追踪。

### 4.1 场景设定

假设：
- 当前 pad.head = N（服务端最新修订号）
- 客户端 A（提交者）本地 rev = N，向服务端提交 changeset
- 客户端 B（协作者）本地 rev = N，未提交任何变更
- 没有 correction changeset（先看最简单的路径，后面再补充）

### 4.2 服务端处理流程（逐行追踪）

服务端 [handleUserChanges()](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/node/handler/PadMessageHandler.ts#L808-L995) 中，关键的确认与广播代码：

```javascript
// 第 966 行：追加修订到 pad
const newRev = await pad.appendRevision(rebasedChangeset, thisSession.author);
// newRev = N+1（有实际变更时），或 N（恒等变更时）
assert([r, r + 1].includes(newRev));

// 第 971-974 行：可能的修正修订（correction changeset，如行标记修正）
const correctionChangeset = _correctMarkersInPad(pad.atext, pad.pool);
if (correctionChangeset) {
  await pad.appendRevision(correctionChangeset, thisSession.author);
  // 如果执行了这一步，pad.head = N+2
}

// 第 978 行：断言提交者的 sessioninfo.rev 还在 r（重基前的版本）
assert.equal(thisSession.rev, r);

// ★ 第 983 行：先给提交者发 ACCEPT_COMMIT
socket.emit('message', {type: 'COLLABROOM', data: {type: 'ACCEPT_COMMIT', newRev}});

// ★ 第 984 行：更新提交者的服务端 session 追踪
thisSession.rev = newRev;

// 第 985 行：如果 newRev 推进了，更新时间戳
if (newRev !== r) thisSession.time = await pad.getRevisionDate(newRev);

// ★ 第 986 行：广播给所有客户端（包括提交者！）
await exports.updatePadClients(pad);
```

**关键发现**：服务端的执行顺序是：
1. 先 `socket.emit(ACCEPT_COMMIT)` 给提交者
2. 再更新 `thisSession.rev = newRev`（由于 thisSession 是 sessioninfos[socket.id] 的引用，sessioninfos 中对应项同步更新）
3. 最后才调用 `updatePadClients()` 广播给所有客户端（包括提交者自己的 socket）

所以提交者会不会再收到 `NEW_CHANGES`，取决于调用 `updatePadClients()` 时，提交者的 `sessioninfo.rev` 与 `pad.head` 的关系——而此时提交者的 `sessioninfo.rev` 已经在第 2 步被更新为 `newRev` 了。

### 4.3 提交者的消息路径（无 correction 的情况）

**前提**：无 correction changeset，`newRev = N+1`，`pad.head = N+1`

时序：

```
服务端                                              客户端 A（提交者）
  │                                                    │
  │ ① socket.emit(ACCEPT_COMMIT, newRev=N+1)         │
  │ ─────────────────────────────────────────────────► │
  │                                                    │  客户端 rev: N → N+1
  │                                                    │  acceptCommit() 执行
  │                                                    │  ┊ applyPreparedChangesetToBase()
  │                                                    │  ┊   baseAText += submittedChangeset
  │                                                    │  ┊   submittedChangeset = null
  │                                                    │  ┊ committing = false
  │                                                    │  ┊ handleUserChanges() 检查新变更
  │                                                    │
  │ ② thisSession.rev = N+1                           │
  │    (thisSession 是 sessioninfos[socket.id] 的引用)│
  │                                                    │
  │ ③ updatePadClients(pad)                            │
  │    遍历所有 socket（包含 A）                        │
  │    检查 sessioninfo.rev (N+1) < pad.head (N+1)?    │
  │    ❌ 不满足，不发送 NEW_CHANGES                    │
  │                                                    │
```

**更正说明**：之前可能误解为 sessioninfo.rev 在 emit 前更新，但实际代码顺序是先 emit ACCEPT_COMMIT（步骤①），再更新 thisSession.rev（步骤②），最后调用 updatePadClients（步骤③）。由于步骤②先于步骤③执行，且 thisSession 和 sessioninfos[socket.id] 是同一个对象引用，因此步骤③遍历到提交者时 sessioninfo.rev 已经是 N+1。

**结论**：在无 correction changeset 的情况下，提交者 **只收到 `ACCEPT_COMMIT`**，不会收到关于自己变更的 `NEW_CHANGES`。

提交者的本地基线更新路径是：
1. `ACCEPT_COMMIT` → `acceptCommit()` → `applyPreparedChangesetToBase()`
2. `applyPreparedChangesetToBase()` 将 `submittedChangeset` 应用到 `baseAText`，然后清空 `submittedChangeset`
3. **不是**通过 `NEW_CHANGES` → `applyChangesToBase()` 的路径更新基线

这两条路径的本质区别：

| 路径 | 触发消息 | 基线更新方式 | 是否需要 follow 重基 |
|------|----------|-------------|---------------------|
| 提交者确认 | ACCEPT_COMMIT | `baseAText += submittedChangeset`（直接应用） | 不需要（因为就是自己的变更） |
| 协作者同步 | NEW_CHANGES | `baseAText += c`（他人变更），然后 follow 重基本地变更 | 需要（把自己的变更重基到他人变更之后） |

### 4.4 协作者的消息路径（无 correction 的情况）

**前提**：无 correction changeset，客户端 B 的 `sessioninfo.rev = N`

```
服务端                                              客户端 B（协作者）
  │                                                    │
  │ ③ updatePadClients(pad)                            │
  │    遍历所有 socket（包含 B）                        │
  │    检查 sessioninfo.rev (N) < pad.head (N+1)?       │
  │    ✅ 满足，发送 NEW_CHANGES                        │
  │ ─────────────────────────────────────────────────► │
  │                                                    │  客户端 rev: N → N+1
  │    sessioninfo.rev = N+1                           │  editor.applyChangesToBase(cs, author, apool)
  │                                                    │  ┊ baseAText += c
  │                                                    │  ┊ submittedChangeset: follow(c, old) 重基
  │                                                    │  ┊ userChangeset: follow(c2, old) 重基
  │                                                    │  ┊ applyChangesetToDocument(postChange)
  │                                                    │
```

**结论**：协作者 **只收到 `NEW_CHANGES`**，通过 `applyChangesToBase()` 处理，需要将服务端变更"穿过"本地的两层未确认变更。

### 4.5 有 correction changeset 的情况

当 `_correctMarkersInPad()` 检测到行标记位置错误时（比如列表标记不在行首），会产生一个修正 changeset：

```
服务端                                              客户端 A（提交者）       客户端 B（协作者）
  │                                                    │                        │
  │ appendRevision(rebased) → pad.head = N+1           │                        │
  │ appendRevision(correction) → pad.head = N+2        │                        │
  │                                                    │                        │
  │ ① socket.emit(ACCEPT_COMMIT, newRev=N+1)         │                        │
  │ ─────────────────────────────────────────────────► │                        │
  │                                                    │ rev: N → N+1           │
  │                                                    │ acceptCommit()         │
  │                                                    │                        │
  │ ② thisSession.rev = N+1  (对象引用)                │                        │
  │                                                    │                        │
  │ ③ updatePadClients(pad)                            │                        │
  │                                                    │                        │
  │  对 A：sessioninfo.rev (N+1) < pad.head (N+2)?     │                        │
  │        ✅ 满足！发送 NEW_CHANGES (r=N+2)           │                        │
  │ ─────────────────────────────────────────────────► │                        │
  │                                                    │ rev: N+1 → N+2         │
  │        sessioninfo.rev = N+2                       │ applyChangesToBase()   │
  │                                                    │                        │
  │  对 B：sessioninfo.rev (N) < pad.head (N+2)?       │                        │
  │        ✅ 满足！发送 NEW_CHANGES (r=N+1)           │                        │
  │ ──────────────────────────────────────────────────────────────────────────► │
  │                                                    │               rev: N → N+1
  │        继续：sessioninfo.rev (N+1) < pad.head (N+2)?                        │
  │        ✅ 满足！发送 NEW_CHANGES (r=N+2)                                    │
  │ ──────────────────────────────────────────────────────────────────────────► │
  │                                                    │              rev: N+1 → N+2
  │        sessioninfo.rev = N+2                                                │
  │                                                    │              applyChangesToBase() ×2
```

**关键点**：
- 有 correction 时，提交者会收到 1 条 `ACCEPT_COMMIT`（自己的变更确认）+ 1 条 `NEW_CHANGES`（correction 修订）
- 协作者会收到 2 条 `NEW_CHANGES`（原始变更 + correction）
- `updatePadClients` 内部的 while 循环确保按修订号顺序逐个补发

### 4.6 恒等变更 (identity changeset) 的情况

当客户端提交的 changeset 经过重基后变成恒等变更（如重传检测命中），`appendRevision()` 不会增加 `pad.head`：

```javascript
// Pad.appendRevision() 中：
const newAText = applyToAText(aChangeset, this.atext, this.pool);
if (newAText.text === this.atext.text && newAText.attribs === this.atext.attribs &&
    this.head !== -1) {
  return this.head;  // 返回当前 head，不增加
}
```

此时 `newRev = r`（等于重基前的版本号），服务端仍然发送 `ACCEPT_COMMIT`：

```
服务端                                              客户端 A（提交者）
  │                                                    │
  │ socket.emit(ACCEPT_COMMIT, newRev=r)              │
  │ ─────────────────────────────────────────────────► │
  │                                                    │ 客户端处理：
  │                                                    │ if (![rev, rev+1].includes(newRev))
  │                                                    │   → 如果 newRev === rev，通过校验
  │                                                    │ rev = newRev (= rev，无变化)
  │                                                    │ acceptCommit() 正常执行
  │                                                    │
  │ thisSession.rev = r (= 原值)                       │
  │                                                    │
  │ updatePadClients(pad)                              │
  │ 对所有客户端：sessioninfo.rev >= pad.head          │
  │ → 不发送任何 NEW_CHANGES                           │
```

**客户端的 ACCEPT_COMMIT 处理逻辑**（[collab_client.ts#L228-L241](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/collab_client.ts#L228-L241)）：

```javascript
} else if (msg.type === 'ACCEPT_COMMIT') {
  serverMessageTaskQueue.enqueue(() => {
    const {newRev} = msg;
    // newRev 可以等于 rev（恒等变更）或 rev+1（正常推进）
    if (![rev, rev + 1].includes(newRev)) {
      window.console.warn(`bad message revision on ACCEPT_COMMIT: ${newRev} not ${rev + 1}`);
      return;
    }
    rev = newRev;
    acceptCommit();
  });
}
```

### 4.7 客户端 rev 与服务端 sessioninfo.rev 的对应关系

客户端和服务端各自独立维护修订号追踪，但保持同步：

| 变量 | 所在位置 | 更新时机 |
|------|----------|----------|
| 客户端 `rev` | [collab_client.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/collab_client.ts) 闭包变量 | 收到 `ACCEPT_COMMIT` 或 `NEW_CHANGES` 时更新 |
| 服务端 `sessioninfo.rev` | [PadMessageHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/node/handler/PadMessageHandler.ts) `sessioninfos[socket.id].rev` | 发送 `ACCEPT_COMMIT` 后立即更新，或 `updatePadClients()` 发送 `NEW_CHANGES` 后更新 |

两者在正常情况下始终相等，因为：
- 提交者：`ACCEPT_COMMIT` 发送后，服务端先更新 `sessioninfo.rev`，客户端后收到消息更新 `rev`
- 协作者：`NEW_CHANGES` 发送后，服务端在 while 循环内更新 `sessioninfo.rev`，客户端异步处理时更新 `rev`

### 4.8 完整消息流转图（修订版）

```
用户 A 输入文字 → 提交 USER_CHANGES(baseRev=N, cs=csA)
  │
  ▼
服务端 handleUserChanges() [串行队列]
  │
  ├─ 校验 → moveOpsToNewPool → 重基(follow循环)
  ├─ appendRevision(csA') → pad.head = N+1
  ├─ (可能的 correction → pad.head = N+2)
  │
  ├─ ① 发送 ACCEPT_COMMIT(newRev) → 提交者 A
  │
  ├─ ② 更新提交者 A 的服务端会话 rev
  │     thisSession.rev = newRev
  │     (thisSession 是 sessioninfos[socket.id] 的引用，
  │      sessioninfos 中对应条目立即同步更新)
  │
  └─ ③ updatePadClients(pad)
       │
       ├─ 对提交者 A:
       │    while sessioninfo.rev < pad.head:
       │      (sessioninfo 与 thisSession 是同一个对象，
       │       sessioninfo.rev 已经是 newRev)
       │      无 correction: sessioninfo.rev(N+1) = pad.head(N+1) → 跳过
       │      有 correction: 发送 NEW_CHANGES(N+2) → A 收到后 applyChangesToBase
       │
       └─ 对协作者 B:
            while sessioninfo.rev(N) < pad.head(N+1):
              发送 NEW_CHANGES(N+1) → B 收到后 applyChangesToBase
              sessioninfo.rev = N+1
              (有 correction 则继续发送 N+2)

═══════════════════════════════════════════════════════

提交者 A 收到 ACCEPT_COMMIT:
  │
  ▼
  客户端 rev: N → newRev
  acceptCommit()
  ├─ editor.applyPreparedChangesetToBase()
  │   └─ baseAText = applyToAText(submittedChangeset, baseAText, apool)
  │   └─ submittedChangeset = null
  ├─ committing = false
  └─ handleUserChanges()  ← 立即检查是否有新待提交变更

  ⚠️ 注意：提交者的基线更新是通过直接应用 submittedChangeset，
  而不是通过 NEW_CHANGES 的 applyChangesToBase() 路径。
  因为 submittedChangeset 就是自己的变更，不需要 follow 重基。

═══════════════════════════════════════════════════════

协作者 B 收到 NEW_CHANGES:
  │
  ▼
  客户端 rev: N → N+1
  editor.applyChangesToBase(csA', author=A, apool)
  ├─ baseAText = applyToAText(csA', baseAText, apool)
  ├─ submittedChangeset: follow(csA', oldSubmitted, false)  ← 重基
  ├─ c2 = follow(oldSubmitted, csA', true)
  ├─ userChangeset: follow(c2, oldUser, true)               ← 重基
  ├─ postChange = follow(oldUser, c2, false)
  └─ applyChangesetToDocument(postChange)  ← 更新 DOM（光标位置自动调整）
```

### 4.9 两条基线更新路径的本质区别

这是理解 Etherpad 协作机制的核心：

**路径一：ACCEPT_COMMIT（提交者专有）**

提交者已经知道自己的变更内容（就是 `submittedChangeset`），所以不需要通过网络重新接收。`acceptCommit()` 直接将本地的 `submittedChangeset` 合并进 `baseAText`，然后清空它。

这相当于：
```
baseAText = baseAText + submittedChangeset
submittedChangeset = null
```
没有信息丢失，不需要操作转换。

**路径二：NEW_CHANGES（所有人，包括提交者收到的 correction）**

接收到的变更来自他人（或服务端自动修正），本地可能有未提交的变更需要与之协调。`applyChangesToBase()` 必须使用 `follow()` 做操作转换，确保本地未提交变更的位置被正确调整。

这相当于：
```
baseAText = baseAText + c
submittedChangeset = follow(c, submittedChangeset)    ← 重基
userChangeset = follow(c', userChangeset)             ← 重基
DOM = apply(postChange)                                ← 反映位置调整
```

信息来自外部，需要与本地状态协调，所以需要操作转换。

### 4.10 消息不乱序的四层保证

客户端永远不会乱序处理 `ACCEPT_COMMIT` 和 `NEW_CHANGES`，这是由四层机制共同保证的：

#### 第一层：服务端同一 pad 的变更串行处理

服务端通过 `Channels` 类确保**同一 pad 的 `USER_CHANGES` 消息串行处理**：

```javascript
// PadMessageHandler.ts L171-L202
class Channels {
  async enqueue(ch, task) {
    const p = (this._promiseChains.get(ch) || Promise.resolve())
        .then(() => this._exec(ch, task));
    // ...
    this._promiseChains.set(ch, pc);
    return await p;
  }
}

// 每个 pad 一个串行队列
const padChannels = new Channels(
  (ch, {socket, message}) => handleUserChanges(socket, message)
);
```

当用户 A 和用户 B 几乎同时提交变更时：
1. A 的 `USER_CHANGES` 先进入队列，开始执行 `handleUserChanges(A)`
2. B 的 `USER_CHANGES` 后进入队列，**必须等 A 的处理完全结束（包括 ACCEPT_COMMIT 和 updatePadClients 全部完成）**才能开始
3. 串行执行保证：A 的 `ACCEPT_COMMIT` 和 `NEW_CHANGES` 全部发送完成后，才会开始处理 B 的变更

这避免了 "A 的 ACCEPT_COMMIT 在 B 的 NEW_CHANGES 之后到达" 这种极端乱序场景。

#### 第二层：同一 handleUserChanges 内的发送顺序保证

在同一个 `handleUserChanges` 执行中，代码顺序是严格的：

```javascript
// PadMessageHandler.ts L976-L986
// 断言：在发送前，sessioninfo.rev 还停留在 r（重基前的版本）
assert.equal(thisSession.rev, r);

// ① 先发 ACCEPT_COMMIT
socket.emit('message', {type: 'COLLABROOM', data: {type: 'ACCEPT_COMMIT', newRev}});

// ② 立即更新提交者的 session rev
// ★ 更正：thisSession 是 sessioninfos[socket.id] 的对象引用，
//   此处修改后，sessioninfos 中的数据立即更新
thisSession.rev = newRev;
if (newRev !== r) thisSession.time = await pad.getRevisionDate(newRev);

// ③ 后调用 updatePadClients 发送 NEW_CHANGES
// ★ 关键：此时提交者的 sessioninfo.rev 已经是 newRev，
//   所以 updatePadClients 内 while 循环条件对提交者不成立（无 correction 时），
//   不会给提交者发送关于自己原始变更的 NEW_CHANGES
await exports.updatePadClients(pad);
```

发送顺序是**程序级别的先后顺序**：先 `socket.emit(ACCEPT_COMMIT)` → 更新 `thisSession.rev` → 再 `await updatePadClients()`。代码中的注释明确指出了这一点：
> "The client assumes that ACCEPT_COMMIT and NEW_CHANGES messages arrive in order. Make sure we have already sent any previous ACCEPT_COMMIT and NEW_CHANGES messages."

**关键更正**：之前若误以为 "先更新 rev 再发 ACCEPT_COMMIT" 是不准确的。实际顺序是先发出 ACCEPT_COMMIT，再更新 rev，最后广播。由于 JavaScript 单线程执行，这三步之间不会被打断，且 thisSession 与 sessioninfos[socket.id] 指向同一个对象，因此当 updatePadClients 内部读取 sessioninfos[socket.id].rev 时已经拿到新值了。

#### 第三层：Socket.IO 同一连接的 FIFO 保证

Socket.IO（基于 WebSocket/长轮询）在**同一个 socket 连接**上保证消息按发送顺序到达。即：
- 服务端先发 M1，再发 M2
- 客户端一定先收到 M1，再收到 M2

对于提交者来说，`ACCEPT_COMMIT` 和 `NEW_CHANGES`（如果有）通过**同一个 socket 连接**发送，所以它们的到达顺序与发送顺序完全一致。

#### 第四层：客户端 serverMessageTaskQueue 串行处理

即使由于某些极端原因（如 TCP 分包重组）消息到达顺序有波动（实际上 Socket.IO 已保证），客户端还有 `serverMessageTaskQueue` 确保**按到达顺序逐个处理**：

```javascript
// collab_client.ts L187-L200
const serverMessageTaskQueue = new class {
  constructor() {
    this._promiseChain = Promise.resolve();
  }

  async enqueue(fn) {
    const taskPromise = this._promiseChain.then(fn);
    this._promiseChain = taskPromise.catch(() => {});
    return await taskPromise;
  }
}();
```

`ACCEPT_COMMIT` 和 `NEW_CHANGES` 都会进入同一个队列：

```javascript
// collab_client.ts L209-L241
if (msg.type === 'NEW_CHANGES') {
  serverMessageTaskQueue.enqueue(async () => { /* 处理 NEW_CHANGES */ });
} else if (msg.type === 'ACCEPT_COMMIT') {
  serverMessageTaskQueue.enqueue(() => { /* 处理 ACCEPT_COMMIT */ });
}
```

队列保证：
- 先到的消息先开始处理
- 前一个消息处理完成（包括其中的异步操作）后，才会开始下一个
- 即使处理 NEW_CHANGES 时有 `await editor.getInInternationalComposition()`，也不会让后面的 ACCEPT_COMMIT 插队

#### 乱序防御：客户端的版本校验

即使四层保证全部失效（极端罕见场景），客户端还有最后一道防线——版本号校验：

```javascript
// NEW_CHANGES 处理
if (newRev !== (rev + 1)) {
  window.console.warn(`bad message revision on NEW_CHANGES: ${newRev} not ${rev + 1}`);
  return;  // 跳过不合法的消息
}

// ACCEPT_COMMIT 处理
if (![rev, rev + 1].includes(newRev)) {
  window.console.warn(`bad message revision on ACCEPT_COMMIT: ${newRev} not ${rev + 1}`);
  return;  // 跳过不合法的消息
}
```

如果收到的 `newRev` 不符合预期，消息会被丢弃，不会破坏本地状态。

---

### 4.11 客户端 rev 与服务端 sessioninfo.rev 的一致性分析

**结论：两者在正常情况下是一致的，但并非强一致，而是**最终一致**。在某些时间窗口和异常场景下会出现短暂不一致。**

#### 正常路径的一致性保证

**提交者路径：**

```
服务端                              客户端
  │                                   │
  │ socket.emit(ACCEPT_COMMIT)        │  此时：服务端 sessioninfo.rev 还是旧值 r
  │─────── 网络传输 ────────          │
  │                                   │
  │ thisSession.rev = newRev    │  客户端 rev 还是旧值 r
  │ (对象引用)                      │
  │                                   │
  │ await updatePadClients()          │
  │ (读取 sessioninfo.rev             │
  │  已是 newRev，所以不              │
  │  给提交者发原始变更)              │
  │─────── 网络传输 ────────          │
  │                                   │  收到 ACCEPT_COMMIT：
  │                                   │  rev = newRev
  │                                   │  ← 此时两者一致
```

时间差：服务端先发出 ACCEPT_COMMIT 消息，再更新 `sessioninfo.rev`，最后广播。客户端要等 ACCEPT_COMMIT 到达后才更新 `rev`。两者之间存在**网络 RTT（往返时间）+ 服务端几步同步操作耗时**的不一致窗口，通常几十到几百毫秒。

**协作者路径：**

```
服务端                              客户端 B
  │                                   │
  │ while sessioninfo.rev < head:     │
  │   socket.emit(NEW_CHANGES)        │
  │   sessioninfo.rev = r      │  客户端 rev 还是旧值
  │─────── 网络传输延迟 ────────│
  │                                   │  收到消息：rev = r
  │                                   │  ← 此时两者一致
```

同样存在 RTT 级别的短暂不一致。

#### 可能出现不一致的场景

| 场景 | 说明 | 是否最终一致 |
|------|------|-------------|
| **网络传输途中** | 服务端已更新 sessioninfo.rev，消息在网络上，客户端尚未收到 | ✅ 是 |
| **客户端队列排队** | 消息已到达客户端，但 serverMessageTaskQueue 中还有前面的任务未处理完 | ✅ 是 |
| **重连期间** | 客户端断开重连，在 CLIENT_RECONNECT 完成前两者不同步 | ✅ 是（重连后补发） |
| **消息校验失败被丢弃** | newRev 不符合预期，打 warn 后 return，不更新客户端 rev | ❌ 永久不一致（但被注释的 disconnect 本应处理） |
| **客户端主动重发** | 客户端超时后重传 USER_CHANGES，服务端检测为重传，返回 newRev 等于当前 rev | ✅ 是 |

#### 代码中故意不一致的处理：坏消息只 warn 不 disconnect

注意到在 `NEW_CHANGES` 和 `ACCEPT_COMMIT` 的校验失败分支中：

```javascript
if (newRev !== (rev + 1)) {
  window.console.warn(`bad message revision on NEW_CHANGES: ${newRev} not ${rev + 1}`);
  // setChannelState("DISCONNECTED", "badmessage_newchanges");
  return;
}
```

**原本的 disconnect 被注释掉了**，只打 warn 然后 return。这意味着：
- 这个坏消息会被跳过，**不会更新客户端 rev**
- 但服务端的 `sessioninfo.rev` 已经更新了
- 此时两者永久不一致，后续的 NEW_CHANGES 都会因为 newRev !== rev+1 而被跳过
- 客户端进入"消息黑洞"状态，但不会断开

这是一个已知的缺陷——当前实现选择"尽量不崩溃"而不是"强制重连恢复一致"。

#### 重连恢复机制

当客户端重连时，会通过 `CLIENT_RECONNECT` 消息重新同步：

```javascript
// 客户端 getMissedChanges() 返回当前状态
const getMissedChanges = () => {
  const obj = {};
  obj.baseRev = rev;
  if (committing && stateMessage) {
    obj.committedChangeset = stateMessage.changeset;
    editor.applyPreparedChangesetToBase();  // 先把 submitted 并入 base
  }
  const userChangesData = prepareUserChangeset();
  if (userChangesData.changeset) {
    obj.furtherChangeset = userChangesData.changeset;
  }
  return obj;
};
```

服务端收到 `CLIENT_RECONNECT` 后，会从 `client_rev` 开始补发所有缺失的修订，确保两者重新对齐。

---

### 4.12 为什么 pad 房间包含提交者自己的连接

`_getRoomSockets(padId)` 返回的是 pad 房间内**所有 socket**，包括提交者自己的 socket。这不是 bug，而是有意的设计。

#### 原因一：correction changeset 需要广播给提交者

服务端的 `_correctMarkersInPad()` 会自动修正行标记位置（如列表标记 `*` 不在行首的情况），产生额外的修订：

```javascript
// PadMessageHandler.ts L971-L974
const correctionChangeset = _correctMarkersInPad(pad.atext, pad.pool);
if (correctionChangeset) {
  await pad.appendRevision(correctionChangeset, thisSession.author);
  // pad.head 从 N+1 变成 N+2
}
```

此时 `ACCEPT_COMMIT` 只确认了用户自己的变更（newRev = N+1），但 correction 修订（N+2）也需要通知提交者。提交者的 socket 在房间内，`updatePadClients()` 的 while 循环会检测到：
```
sessioninfo.rev(N+1) < pad.head(N+2) → 发送 NEW_CHANGES(r=N+2)
```
提交者通过 `applyChangesToBase()` 接收并应用这个自动修正。

#### 原因二：通过 rev 跟踪自然避免重复，无需特殊排除

**关键机制**：服务端在调用 `updatePadClients()` **之前**，已经先更新了 `thisSession.rev = newRev`，而 `thisSession` 与 `sessioninfos[socket.id]` 是**同一个对象引用**。因此当 `updatePadClients()` 内部遍历提交者的 socket 时：

```javascript
const sessioninfo = sessioninfos[socket.id];  // 拿到的就是 thisSession（同一个对象）
while (sessioninfo.rev < pad.getHeadRevisionNumber()) {
  // 无 correction 时：sessioninfo.rev(newRev) = pad.head(newRev) → 条件不成立，跳过
  // 有 correction 时：sessioninfo.rev(newRev) < pad.head(newRev+1) → 条件成立，发送 correction
}
```

如果 `updatePadClients()` 要排除提交者，需要额外的判断逻辑：
```javascript
// 伪代码：需要额外判断和传参
if (socket.id !== submitterSocketId) {
  socket.emit('message', msg);
}
```

而且还需要传递 `submitterSocketId` 到 `updatePadClients()` 中，增加了函数耦合。当前设计：
- 无 correction 时，提交者自动不会收到自己的变更（rev 自然对齐）
- 有 correction 时会自然收到修正修订
- 其他协作者按正常流程接收
- 代码不需要特殊分支，简洁可靠

#### 原因三：后续其他用户的变更也需要广播给提交者

提交者提交完成后，仍然是 pad 的协作者之一，后续其他用户的变更也需要通过房间广播给他。

#### 佐证：客户端加入房间的时机

客户端在 `CLIENT_READY` 处理中加入房间：

```javascript
// PadMessageHandler.ts L1389-L1390
// Join the pad and start receiving updates
socket.join(sessionInfo.padId);
```

然后才发送 `CLIENT_VARS` 和初始化 `sessionInfo.rev`。注意注释说明：

```javascript
// PadMessageHandler.ts L1409-L1412
// Flush any revisions that may have been appended while we were awaiting the
// clientVars hook (before socket.join).  Those revisions were broadcast to
// existing room members but this socket hadn't joined yet so it missed them.
await exports.updatePadClients(pad);
```

在 `socket.join()` 之前，其他用户的广播这个新客户端是收不到的，所以需要主动调用 `updatePadClients()` 补发。这也从侧面说明：**一旦加入房间，客户端会收到包括自己提交在内的所有广播消息，但通过 rev 跟踪机制避免重复处理。**

---

## 五、完整状态流转图

```
                          服务器
                            |
              ┌─────────────┴─────────────┐
              |       (pad.head = N)      |
              |                           |
              └─────────────┬─────────────┘
                            |
              ┌─────────────▼─────────────┐
              |     NEW_CHANGES 广播      |
              |   (给所有在线客户端)      |
              └─────────────┬─────────────┘
                            |
  ┌─────────────────────────┼─────────────────────────┐
  |                         |                         |
  ▼                         ▼                         ▼
客户端 A                 客户端 B                 客户端 C
(baseRev = N)           (baseRev = M)           (baseRev = K)

用户输入 → userChangeset 累积
  |
  ▼
handleUserChanges 触发提交 (commitDelay 节流)
  |
  ▼
USER_CHANGES(baseRev=N, changeset=cs)
  |
  ▼
服务端 padChannels 串行处理
  |
  ├─→ 安全校验 (作者、语法)
  ├─→ moveOpsToNewPool (属性池迁移)
  ├─→ while r < head: follow(c_svr, cs_client) 重基
  ├─→ appendRevision (保存为新修订)
  ├─→ ACCEPT_COMMIT (给提交者)
  └─→ updatePadClients (给所有客户端广播 NEW_CHANGES)
  |
  ▼
客户端收到 NEW_CHANGES:
  ├─→ applyToAText(c, baseAText)      更新基准
  ├─→ follow(c, submittedChangeset)   重基已提交变更
  ├─→ follow(c2, userChangeset)       重基未提交变更
  └─→ applyChangesetToDocument        更新 DOM
```

---

## 五、关键技术细节

### 5.1 compose vs follow 的区别

| 维度 | compose | follow |
|------|---------|--------|
| 输入关系 | cs2 基于 cs1 的结果 | cs1 和 cs2 基于同一旧版本 |
| 输出 | 一个合并后的 changeset | 转换后的 cs2'（在 cs1 之后应用） |
| 用途 | 连续变更合并 | 并发变更重基 |
| 类比 | 顺序执行 | 分支合并 / rebase |

### 5.2 属性池 (Attribute Pool)

Changeset 中的属性用数字编号引用（如 `*0` 代表第一个属性），每个客户端和服务端都有自己的属性池。

- **发送时**：`prepareForWire()` 将本地编号转换为独立格式
- **接收时**：`moveOpsToNewPool()` 将 wire 编号转换为本地池编号
- **目的**：避免每次传输完整属性名，节省带宽

### 5.3 串行处理的必要性

服务端用 `Channels` 对同一 pad 的消息串行排队，原因：
1. OT 算法要求变更按确定顺序应用
2. `appendRevision` 需要原子性，不能并发修改 `pad.head`
3. 保证广播给各客户端的修订顺序一致

### 5.4 重传检测

服务端在重基过程中检测重复提交：

```javascript
if (canonicalCs === c && thisSession.author === authorId) {
  // 这是重复提交（如网络超时导致客户端重发）
  rebasedChangeset = identity(unpack(canonicalCs).oldLen);
}
```

如果检测到已经应用过相同的 changeset（同一作者、相同内容），就转换为恒等 changeset，不会产生新修订。

---

## 六、异常场景与容错

### 6.1 客户端提交超时

collab_client 检测提交超时：
- 5 秒：触发 `onConnectionTrouble('SLOW')`
- 20 秒：断开连接 `setChannelState('DISCONNECTED', 'slowcommit')`

### 6.2 重连恢复

客户端重连后，服务端通过 `CLIENT_RECONNECT` 消息补发缺失的修订：
- 客户端发送自己的 `baseRev` 和未确认的变更
- 服务端补发缺失的修订，并重新处理未确认的变更
- 对应代码见 collab_client.ts 中 `CLIENT_RECONNECT` 分支

### 6.3 坏的 Changeset

服务端如果处理失败，发送 `disconnect: 'badChangeset'` 断开客户端，防止损坏文档。

---

## 七、涉及的核心文件清单

| 文件 | 作用 |
|------|------|
| [src/static/js/Changeset.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/Changeset.ts) | Changeset 数据结构与核心算法（compose, follow, applyToAText 等） |
| [src/static/js/changesettracker.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/changesettracker.ts) | 客户端三层状态管理（base/submitted/user） |
| [src/static/js/collab_client.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/collab_client.ts) | 客户端协作逻辑（提交、确认、接收变更） |
| [src/node/handler/PadMessageHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/node/handler/PadMessageHandler.ts) | 服务端消息处理（handleUserChanges, updatePadClients） |
| [src/node/db/Pad.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/node/db/Pad.ts) | Pad 数据模型（appendRevision, 修订存储） |
| [src/static/js/AttributePool.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/1-etherpad-lite/src/static/js/AttributePool.ts) | 属性池管理 |
