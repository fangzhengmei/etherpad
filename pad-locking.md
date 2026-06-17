# Etherpad 协作文档脏写防护设计

Etherpad 的并发编辑并不依赖数据库事务或悲观行锁,而是由三层机制叠加构成"脏写防护":

1. **锁粒度** —— 以 pad 为单位的进程内串行队列;
2. **操作变换串行化(OT)** —— 把客户端基于旧版本的 changeset 重新基线(rebase)到最新 head 版本;
3. **冲突回滚** —— 校验先于提交,任何不合法的 changeset 都被整体拒绝并触发客户端重载。

下面逐块对照代码说明。核心逻辑集中在 [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts)、[Pad.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/db/Pad.ts)、[Changeset.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/Changeset.ts) 与客户端 [collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/collab_client.ts)。

---

## 一、锁粒度:per-pad 的 Promise 链串行队列

### 1.1 设计选择:粒度是"整个 pad",不是"用户"或"行段"

Etherpad 没有按字符区间或行加锁,而是把**对同一个 pad 的所有写操作串行化**。这把"并发编辑"问题降维成"对 pad 的单写者"问题——只要一次只有一个 changeset 在被处理,后续到达的 changeset 看到的就一定是已被前者更新过的 head 版本,从而把"脏写"(基于过期版本的写)变成可被 OT 修正的"过期基线",而不是真正冲突的覆盖写。

实现是一个极简的 `Channels` 类,见 [Channels 类定义](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L171-L202):

```ts
class Channels {
  private readonly _exec: (ch:any, task:any) => any;
  private _promiseChains: Map<any, Promise<any>>;
  constructor(exec = (ch: string, task:any) => task(ch)) {
    this._exec = exec;
    this._promiseChains = new Map();
  }
  async enqueue(ch:any, task:any): Promise<any> {
    const p = (this._promiseChains.get(ch) || Promise.resolve()).then(() => this._exec(ch, task));
    const pc = p
        .catch(() => {}) // Prevent rejections from halting the queue.
        .then(() => {
          if (this._promiseChains.get(ch) === pc) this._promiseChains.delete(ch);
        });
    this._promiseChains.set(ch, pc);
    return await p;
  }
}
```

关键点:

- **Map 的 key 是 channel**。对于 pad 写入,这个 channel 就是 `padId`(见 [padChannels 实例化](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L207)与 [USER_CHANGES 入队](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L576-L579))。所以**不同 pad 之间互不阻塞、完全并行**;同一 pad 内严格串行。
- **串行靠 Promise 链实现**:`this._promiseChains.get(ch).then(() => exec(...))` 把新任务挂在同 channel 既有链的末尾,前一个未 settle,后一个就不会执行。这正是"自旋等待锁"的 async 版本,但它**只阻塞逻辑、不阻塞事件循环**。
- **`.catch(() => {})` 防止毒队列**:任意一个 changeset 处理失败被 reject,都会被吞掉,从而不会卡死后续任务——这一点对"冲突回滚"至关重要(见第三节):被拒绝的 changeset 决不能让整条 pad 队列停摆。
- **空闲清理**:链尾 `.then` 里检查"是否仍是自己",是则 `delete`,避免 Map 无限增长。

### 1.2 为什么只串行 USER_CHANGES

注意 `handleMessage` 的分发:只有 `COLLABROOM` 下的 `USER_CHANGES` 走 [padChannels.enqueue](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L576-L579)。`USERINFO_UPDATE`、`CHAT_MESSAGE` 等不进队列——它们不触碰 pad 文本与版本号,没有脏写风险,串行它们只会徒增延迟。这体现了"锁只覆盖真正会竞争的资源"的粒度原则。

### 1.3 进程内锁的边界(重要前提)

这个锁是**单 Node 进程内**的。Etherpad 默认单进程运行;水平扩展时靠前置的 sticky session 把同一 `padId` 的连接固定到同一进程。因此该队列**不提供跨进程/跨服务器的互斥**。这是 Etherpad 在架构上用"sticky session + 进程内串行"换取"无分布式锁、无两阶段提交"的明确取舍。

### 1.4 客户端侧的对称串行

服务端串行还不够,客户端也必须保证同一时刻只有一个 changeset 在途,否则服务端会收到基于同一 baseRev 的多份重叠提交。客户端用 `committing` 标志位实现,见 [handleUserChanges 状态机](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/collab_client.ts#L112-L123) 与 [提交动作](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/collab_client.ts#L137-L148):

```ts
if (committing) { /* 上一次提交还没拿到 ACCEPT_COMMIT,轮询等待 */ return; }
...
committing = true;
stateMessage = { type: 'USER_CHANGES', baseRev: rev, changeset, apool };
sendMessage(stateMessage);
```

收到 `ACCEPT_COMMIT` 才在 [acceptCommit](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/collab_client.ts#L160-L169) 里 `setStateIdle()` 把 `committing` 复位,并立即触发下一轮 `handleUserChanges()` 把"提交期间累积"的本地编辑继续发出去。这样客户端→服务端形成"一次一个、确认后再发下一个"的管道,与服务端的 per-pad 串行队列端到端对齐。

此外,客户端用一个 [serverMessageTaskQueue](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/collab_client.ts#L187-L200) 把服务端回推的 `NEW_CHANGES`/`ACCEPT_COMMIT` 串行化(同样的 Promise 链手法),避免 DOM 应用顺序错乱。

---

## 二、操作变换串行化:把"过期基线"rebase 到 head

锁保证了"同一 pad 内 changeset 依次处理",但**依次处理 ≠ 版本相同**:客户端发出的 changeset 是基于它本地记录的 `baseRev`,等它排到队首被处理时,服务端 head 可能已经被前面的同事推进了好几版。这时不能直接套用,必须用 OT 把它"重基线"到最新版本。这一段是整个脏写防护里最绕的部分。

入口在 [handleUserChanges](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L819-L1006)。串行化的核心是下面这个 while 循环,见 [OT rebase 循环](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L939-L951):

```ts
let r = baseRev;
// 客户端 changeset 可能不是基于最新版本,因为其他客户端同时在提交。
// 把 changeset 重写为可应用于最新版本的形式。
while (r < pad.getHeadRevisionNumber()) {
  r++;
  const {changeset: c, meta: {author: authorId}} = await pad.getRevision(r);
  if (canonicalCs === c && thisSession.author === authorId) {
    // 判定为已提交 changeset 的重传,降级为 identity(空操作)。
    rebasedChangeset = identity(unpack(canonicalCs).oldLen);
  }
  // 此时 c(来自 pad)与 rebasedChangeset(来自客户端)都相对于 r-1。
  // follow 把 rebasedChangeset 重基线到"可紧接在 c 之后应用"。
  rebasedChangeset = follow(c, rebasedChangeset, false, pad.pool);
}
```

### 2.1 follow 做了什么

`follow(cs1, cs2, reverseInsertOrder, pool)` 是 Etherpad 的 OT 变换算子,定义在 [follow 函数](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/Changeset.ts#L1446)。语义是:**给定两个都基于同一 oldLen 的 changeset `cs1`、`cs2`,返回一个可紧接在 `cs1` 之后应用的、与 `cs2` 意图等价的新 changeset**。也就是说 `follow(c, mine)` = "把我的编辑挪到同事那版编辑的后面去"。

循环里每轮 `r++` 取出第 r 版已落库的 changeset `c`,然后用 `follow(c, rebasedChangeset)` 把客户端 changeset 在 `c` 之上重基线一次。循环结束时,`rebasedChangeset` 已经是基于当前 head 的合法 changeset,`oldLen` 等于 head 文档长度,可以直接套用。

`reverseInsertOrder = false` 这第三个参数控制"双方都在同一处插入"时的对称打破:谁先谁后。Etherpad 用 `false` 表示按服务端已落库版本(`cs1`)优先的确定性顺序,并辅以 `insertorder=first` 属性、换行优先等规则,见 [follow 内部 tie-breaking](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/Changeset.ts#L1463-L1490)。确定性保证所有客户端最终收敛到同一文本。

### 2.2 重传幂等:identity 兜底

网络抖动会让同一个 changeset 被发两遍。循环里的 `if (canonicalCs === c && thisSession.author === authorId)` 分支把"内容相同且作者相同"的命中识别为重传,用 [identity](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/Changeset.ts#L809)(`pack(N, N, '', '')`,oldLen==newLen、无 op 的空变更集)替换,确保重传不会把文字重复插入两遍。注意比较的是 `canonicalCs`——即 `moveOpsToNewPool` 之后的形式,见 [pool 映射快照](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L925-L931),否则属性号被重编号后(如 `*0`→`*1`)就比对不上了。

### 2.3 属性池重映射:moveOpsToNewPool

客户端发来的 changeset 自带一个"线材 apool"(属性号是客户端本地的),服务端要把它翻译到 pad 全局池,见 [moveOpsToNewPool](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L925)。这一步在 rebase 之前完成,保证 `follow`/`compose` 时两边共用同一个 `pad.pool`。

### 2.4 串行化链条的校验闸门

rebase 完成后、落库前,还要过两道结构校验,任何一道不过都直接进冲突回滚:

- **长度不变量**:[oldLen 检查](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L955-L959)——rebase 后的 `oldLen` 必须等于当前文档长度,否则说明 rebase 出错或 changeset 损坏。
- **尾换行不变量**:[trailing newline 校验](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L969-L975)——把 rebase 后的 changeset 投影到文本上,若结果不以 `\n` 结尾则拒绝。注释里解释了为何从"服务端默默补一个修正版"改成"直接拒绝":修正版会晚于第一份畸形广播到达浏览器,触发 `line assembler not finished` 断言而把会话打挂。

### 2.5 落库幂等

[appendRevision](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/db/Pad.ts#L280-L335) 自带一道去重:若新文本与旧文本完全相同且 head≠-1,直接返回当前 head 而不新增版本号,见 [幂等短路](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/db/Pad.ts#L288-L292)。这样 identity changeset、"删了又加回同样字符"等净效果为零的提交不会污染版本链。

---

## 三、冲突回滚:校验先于提交 + 拒绝重载

"冲突回滚"在 Etherpad 里有两层含义,要分清。

### 3.1 第一层(主):脏写防护——整体拒绝,而不是局部撤销

Etherpad **不做**"发现冲突就回滚对方那部分"的细粒度撤销。它的策略更简单也更稳健:**所有校验都放在 `appendRevision` 之前,任何不合法的 changeset 根本不会触碰 pad 状态;失败时把这个客户端踢下线,让它重载到服务端规范状态**。

看 [handleUserChanges 的 try/catch 结构](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L842-L1005):try 块里依次是 `checkRep`(语法规范)→ 作者属性校验(防伪造/冒充作者,见 [L859-L920](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L859-L920))→ `moveOpsToNewPool` → OT rebase 循环 → 长度校验 → 尾换行校验,而**唯一的落库动作 `pad.appendRevision` 排在最后**,见 [L977](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L977)。

这意味着:只要前面任意一步抛错,控制流就跳到 catch,`appendRevision` 永远不会执行,pad 仍停留在拒绝前的 head 版本——**回滚的"成本"为零,因为根本没写进去**。这才是"脏写防护"的本质:**先验证再写入,写不进去就当无事发生**。

catch 块见 [冲突处理](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L998-L1002):

```ts
} catch (err:any) {
  socket.emit('message', {disconnect: 'badChangeset'});
  stats.meter('failedChangesets').mark();
  messageLogger.warn(`Failed to apply USER_CHANGES from author ${...}: ${err.stack || err}`);
}
```

服务端做三件事:给该 socket 单发一条 `{disconnect: 'badChangeset'}`、打 `failedChangesets` 计数、记日志。注意它**只断开这一个客户端**,对同一 pad 的其他客户端毫无影响(他们仍在 per-pad 队列里正常串行)。

客户端收到带 `disconnect` 字段的消息后,见 [pad.ts 的 disconnect 分支](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/pad.ts#L478-L483):

```ts
} else if (obj.disconnect) {
  padconnectionstatus.disconnected(obj.disconnect);
  socket.disconnect();
  padeditor.disable();   // 禁止用户继续编辑,锁死本地
}
```

`padconnectionstatus.disconnected('badChangeset')` 会弹出本地化的"你的修改未被接受"模态,见 [disconnected 处理](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/pad_connectionstatus.ts#L54-L86);`padeditor.disable()` 把编辑器置为只读,阻止用户在被踢期间继续累积脏写。同时,若存在"未接受的提交",[showUnacceptedCommitWarning](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/pad.ts#L1022) 会明确提示用户**这次编辑没存上**。

用户点击"重新连接"触发 [window.location.reload()](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/pad_connectionstatus.ts#L35-L37),页面重新加载、重新握手 `CLIENT_VARS`,从服务端拿回 head 的规范 AText。客户端那一份被拒绝的、基于旧 baseRev 的 changeset 随之丢弃——这就是"回滚到服务端真相"。

> 设计要点:Etherpad 选择"拒绝并重载"而非"服务端帮忙重做",是因为 OT 在 rebase 失败(长度对不上、结构损坏)时往往意味着客户端本地状态已与服务端不可调和,继续自动重试只会反复失败。强制重载是最干净的状态收敛手段。

### 3.2 第二层(辅):基于 inverse 的版本回退——append-only,绝不改写历史

另一类"回滚"是用户主动回到历史版本,典型场景是时间轴(timeslider)与 `restoreRevision` API。它们用 [inverse](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/Changeset.ts#L1279)——给定一个 changeset 和它应用后的文档行,算出能把它"撤销"的逆 changeset。

- **时间轴前/后退**:为支持播放/倒放,服务端预计算每个区段的正向与逆向 changeset 对,见 [getPadChangesetInfo 中的 inverse 调用](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1586-L1600):`forwards` 是 r→r+n 的合成变更集,`backwards = inverse(forwards, lines...)` 是它的逆。客户端拿到这对数组就能在版本间双向滑动。
- **restoreRevision**:[API.restoreRevision](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/db/API.ts#L599) 把历史某版的 atext 作为一条**新的正向 changeset** 追加到 head,见 [L633-L634](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/db/API.ts#L633)。它**不删除、不改写**任何历史版本,而是用 `appendRevision` 追加一个"恢复到旧貌"的新版本。这与脏写防护的"整体拒绝"是两套独立机制:历史永远是 append-only 的,所谓"回退"其实是"新建一个长得像旧版的新版"。

---

## 四、三层如何协同

把三块串起来看一次完整的并发写入:

1. **客户端** A、B 同时编辑同一 pad。各自 `committing` 标志保证本地一次只发一个 changeset(第一节客户端侧)。
2. 两份 `USER_CHANGES` 到达服务端,都进 [padChannels](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L207),按 `padId` 排进同一条 Promise 链——**A 先处理,B 在 await 中排队**(第一节服务端侧)。这把并发竞争消除在进入 OT 之前。
3. A 的 changeset(假设 baseRev 已是 head)直接通过校验,[appendRevision](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/db/Pad.ts#L977) 落库,head 推进。
4. 轮到 B。B 的 `baseRev` 现在落后于 head,进入 [OT rebase 循环](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L939-L951),用 `follow(A的changeset, B的changeset)` 把 B 的意图挪到 A 之后,变成基于新 head 的合法 changeset(第二节)。
5. rebase 后过长度/尾换行校验。通过则落库;若 rebase 出错或 changeset 畸形(第三节第一层),则**不落库**,直接 `badChangeset` 踢 B 下线重载,服务端 pad 状态毫发无损。

一句话总结:**锁把并发变成串行,OT 把串行后的过期基线纠正到当前版本,校验先于提交则保证纠正不了的脏写被整体拒之门外而不留痕迹**。三者缺一:没有锁,OT 要处理真正的并发覆盖;没有 OT,串行也救不了过期基线;没有校验先于提交,畸形 changeset 就会污染 pad 的版本链与文本。

---

## 附:关键代码索引

| 机制 | 位置 |
| --- | --- |
| per-pad 串行队列 `Channels` | [PadMessageHandler.ts#L171-L202](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L171-L202) |
| USER_CHANGES 入队(锁的入口) | [PadMessageHandler.ts#L576-L579](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L576-L579) |
| handleUserChanges 主体 | [PadMessageHandler.ts#L819-L1006](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L819-L1006) |
| OT rebase 循环 | [PadMessageHandler.ts#L939-L951](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L939-L951) |
| follow(OT 变换算子) | [Changeset.ts#L1446](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/Changeset.ts#L1446) |
| identity(空 changeset) | [Changeset.ts#L809](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/Changeset.ts#L809) |
| 尾换行不变量校验 | [PadMessageHandler.ts#L969-L975](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L969-L975) |
| appendRevision(幂等落库) | [Pad.ts#L280-L335](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/db/Pad.ts#L280-L335) |
| badChangeset 拒绝 | [PadMessageHandler.ts#L998-L1002](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L998-L1002) |
| 客户端 disconnect 分支 | [pad.ts#L478-L483](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/pad.ts#L478-L483) |
| 客户端 committing 状态机 | [collab_client.ts#L112-L148](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/static/js/collab_client.ts#L112-L148) |
| inverse(逆 changeset,时间轴) | [PadMessageHandler.ts#L1586-L1600](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1586-L1600) |
| restoreRevision(历史回退) | [API.ts#L599](file:///d:/fz/0601-2/solo-dogfeeding/code/24-etherpad-lite/src/node/db/API.ts#L599) |
