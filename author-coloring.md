# Etherpad 多作者协作颜色分配与归属机制

## 概述

Etherpad 的多作者协作系统通过一套完整的颜色分配、作者元数据管理和会话生命周期机制，实现了实时协作中作者身份的可视化标识。每个作者被分配一个独特的颜色，用于在编辑器中标记其输入的文本，并在用户列表中展示。

---

## 1. 颜色池 (Color Palette)

### 1.1 颜色池定义

颜色池在 [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L27-L92) 中定义，包含 64 种预定义的柔和背景色：

```typescript
exports.getColorPalette = () => [
  '#ffc7c7', // 浅红
  '#fff1c7', // 浅黄
  '#e3ffc7', // 浅绿
  '#c7ffd5', // 浅青绿
  // ... 共 64 种颜色
];
```

### 1.2 颜色池特性

- **64 种预设颜色**：足够支撑大部分协作场景，颜色经过挑选确保可读性
- **柔和背景色**：均为浅色背景，避免与文本颜色冲突
- **全局共享**：所有 Pad 实例共享同一个颜色池
- **可通过插件扩展**：通过 `clientVars` 钩子可修改发送到客户端的颜色面板

---

## 2. 作者元数据管理 (Author Metadata)

### 2.1 作者创建与颜色分配

作者创建逻辑位于 [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L201-L212) 的 `createAuthor` 函数：

```typescript
exports.createAuthor = async (name: string) => {
  const author = `a.${randomString(16)}`;  // 生成 a. 开头的 16 位随机 ID
  const now = Date.now();
  const authorObj = {
    colorId: Math.floor(Math.random() * (exports.getColorPalette().length)),  // 随机分配颜色索引
    name,
    timestamp: now,
    lastSeen: now,
  };
  await db.set(`globalAuthor:${author}`, authorObj);
  return {authorID: author};
};
```

**关键设计点**：
- `colorId` 存储的是**颜色池索引**（0-63），而非直接存储颜色值
- 颜色分配是**完全随机**的，不考虑当前已在线用户的颜色使用情况
- 作者 ID 格式：`a.` + 16 位随机字符串（如 `a.xk3jd92mcn48fhqp`）

### 2.2 作者数据结构

存储在数据库中的作者对象结构（键：`globalAuthor:${authorID}`）：

```typescript
{
  colorId: number | string,  // 颜色索引 (0-63) 或直接存储 CSS 颜色值
  name: string | null,       // 作者显示名称
  timestamp: number,         // 创建时间戳
  lastSeen: number,          // 最后活动时间戳
  padIDs?: {                 // 参与的 Pad 列表
    [padId: string]: 1
  },
  erased?: boolean,          // 是否已被匿名化 (GDPR)
  erasedAt?: string,         // 匿名化时间
}
```

### 2.3 作者颜色与名称的读写 API

[AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts) 提供以下核心方法：

| 方法 | 用途 |
|------|------|
| `getAuthor(authorId)` | 获取完整作者对象 |
| `getAuthorColorId(authorId)` | 获取作者颜色 ID |
| `setAuthorColorId(authorId, colorId)` | 设置作者颜色（同时更新 `lastSeen`） |
| `getAuthorName(authorId)` | 获取作者名称 |
| `setAuthorName(authorId, name)` | 设置作者名称（同时更新 `lastSeen`） |

---

## 3. 作者身份映射 (Author Identity Mapping)

### 3.1 Token 到作者的映射

用户通过 token 与作者身份绑定，映射关系存储在数据库中：

```typescript
// 从 token 解析 authorId
const getAuthor4Token = async (token: string) => {
  const author = await mapAuthorWithDBKey('token2author', token);
  return author ? author.authorID : author;
};
```

[AuthorManager.ts:148-153](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L148-L153)

### 3.2 映射创建逻辑

[mapAuthorWithDBKey](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L117-L141) 函数处理映射的创建与查找：

```typescript
const mapAuthorWithDBKey = async (mapperkey: string, mapper: string) => {
  const author = await db.get(`${mapperkey}:${mapper}`);

  if (author == null) {
    // 映射不存在，创建新作者
    const author = await exports.createAuthor(null);
    await db.set(`${mapperkey}:${mapper}`, author.authorID);
    return author;
  }

  // 映射存在，更新 lastSeen
  const now = Date.now();
  await db.setSub(`globalAuthor:${author}`, ['timestamp'], now);
  await db.setSub(`globalAuthor:${author}`, ['lastSeen'], now);

  return {authorID: author};
};
```

**数据库键模式**：
- `token2author:${token}` → `authorID` （浏览器 token 映射）
- `mapper2author:${mapper}` → `authorID` （外部系统映射）

---

## 4. 会话生命周期 (Session Lifecycle)

### 4.1 会话创建

会话管理位于 [SessionManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L105-L159)，通过 API 创建的会话将作者与群组绑定：

```typescript
exports.createSession = async (groupID: string, authorID: string, validUntil: number) => {
  const sessionID = `s.${randomString(16)}`;

  await db.set(`session:${sessionID}`, {groupID, authorID, validUntil});

  // 建立反向索引
  await Promise.all([
    db.setSub(`group2sessions:${groupID}`, ['sessionIDs', sessionID], 1),
    db.setSub(`author2sessions:${authorID}`, ['sessionIDs', sessionID], 1),
  ]);

  return {sessionID};
};
```

### 4.2 会话查找与验证

[findAuthorID](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L39-L85) 用于从 cookie 中解析有效会话对应的作者：

```typescript
exports.findAuthorID = async (groupID: string, sessionCookie: string) => {
  const sessionIDs = sessionCookie.replace(/^"|"$/g, '').split(',');
  const now = Math.floor(Date.now() / 1000);

  // 查找未过期且匹配群组的会话
  const isMatch = (si) => si != null && si.groupID === groupID && now < si.validUntil;
  const sessionInfo = await firstSatisfies(sessionInfoPromises, isMatch);

  return sessionInfo?.authorID;
};
```

### 4.3 连接会话信息 (sessioninfos)

在 [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L102-L103) 中，每个 socket 连接维护会话信息：

```typescript
const sessioninfos = {
  [socketId]: {
    auth: { padID, sessionID, token },  // 认证信息
    author: string,                      // 作者 ID
    padId: string,                       // Pad ID
    readOnlyPadId: string,               // 只读 Pad ID
    readonly: boolean,                   // 是否只读
    rev: number,                         // 已应用的最新修订号
    time: number,                        // 最新修订时间戳
    embed: boolean,                      // 是否为嵌入模式
  }
};
```

---

## 5. 客户端初始化流程 (Client Initialization)

### 5.1 CLIENT_READY 消息处理

用户连接后发送 `CLIENT_READY` 消息，服务器在 [handleClientReady](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1118-L1458) 中进行处理：

```typescript
const handleClientReady = async (socket, message) => {
  // 1. 解析 token/sessionID，获取 authorID
  const {accessStatus, authorID} =
      await securityManager.checkAccess(auth.padID, auth.sessionID, auth.token, user);

  // 2. 应用客户端可能自带的颜色/名称
  let {colorId: authorColorId, name: authorName} = message.userInfo || {};
  await Promise.all([
    authorName && authorManager.setAuthorName(sessionInfo.author, authorName),
    authorColorId && authorManager.setAuthorColorId(sessionInfo.author, authorColorId),
  ]);

  // 3. 重新从数据库读取（确保一致性）
  ({colorId: authorColorId, name: authorName} =
      await authorManager.getAuthor(sessionInfo.author));

  // 4. 收集 Pad 上所有历史作者信息
  const authors = pad.getAllAuthors();
  const historicalAuthorData = {};
  await Promise.all(authors.map(async (authorId) => {
    const author = await authorManager.getAuthor(authorId);
    historicalAuthorData[authorId] = {name: author.name, colorId: author.colorId};
  }));

  // 5. 组装 clientVars
  const clientVars = {
    collab_client_vars: {
      initialAttributedText: atext,
      historicalAuthorData,        // 所有历史作者的颜色/名称
      apool,
      rev: headRev,
    },
    colorPalette: authorManager.getColorPalette(),  // 发送完整颜色池
    userColor: authorColorId,                        // 当前用户的颜色
    userId: sessionInfo.author,                      // 当前用户的 authorID
    userName: authorName,                            // 当前用户名称
    // ... 其他字段
  };

  // 6. 发送 clientVars 给客户端
  socket.emit('message', {type: 'CLIENT_VARS', data: clientVars});

  // 7. 广播给其他在线用户
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
};
```

### 5.2 客户端协作客户端初始化

客户端收到 `CLIENT_VARS` 后，在 [collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts#L39-L529) 中初始化协作客户端：

```typescript
const getCollabClient = (ace2editor, serverVars, initialUserInfo, options, _pad) => {
  const userId = initialUserInfo.userId;
  const userSet = {};  // userId -> userInfo
  userSet[userId] = initialUserInfo;

  // 初始化时通知 ACE 编辑器历史作者信息
  const tellAceAboutHistoricalAuthors = (hadata) => {
    for (const [author, data] of Object.entries(hadata)) {
      if (!userSet[author]) {
        tellAceAuthorInfo(author, data.colorId, true);  // true = 淡出显示
      }
    }
  };

  // 通知 ACE 编辑器单个作者的颜色信息
  const tellAceAuthorInfo = (userId, colorId, inactive) => {
    if (typeof colorId === 'number') {
      colorId = clientVars.colorPalette[colorId];  // 索引转颜色值
    }

    if (inactive) {
      editor.setAuthorInfo(userId, {bgcolor: colorId, fade: 0.5});
    } else {
      editor.setAuthorInfo(userId, {bgcolor: colorId});
    }
  };

  // 初始化时调用
  tellAceAboutHistoricalAuthors(serverVars.historicalAuthorData);
  tellAceActiveAuthorInfo(initialUserInfo);
};
```

---

## 6. 实时颜色同步 (Real-time Color Sync)

### 6.1 用户信息更新流程

当用户修改颜色或名称时，触发以下流程：

**1. 客户端发起更新** ([pad_userlist.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/pad_userlist.ts#L642-L649))：

```typescript
const closeColorPicker = (accept) => {
  if (accept) {
    myUserInfo.colorId = newColor;
    pad.notifyChangeColor(newColor);  // 发送 USERINFO_UPDATE
    paduserlist.renderMyUserInfo();
  }
};
```

**2. collab_client 转发** ([collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts#L326-L336))：

```typescript
const updateUserInfo = (userInfo) => {
  userInfo.userId = userId;
  userSet[userId] = userInfo;
  tellAceActiveAuthorInfo(userInfo);  // 本地立即更新
  sendMessage({
    type: 'USERINFO_UPDATE',
    userInfo,
  });
};
```

**3. 服务端处理** ([PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L771-L803))：

```typescript
const handleUserInfoUpdate = async (socket, {data: {userInfo: {name, colorId}}}) => {
  const author = session.author;

  // 验证颜色格式
  if (!/(^#[0-9A-F]{6}$)|(^#[0-9A-F]{3}$)/i.test(colorId)) {
    throw new Error(`malformed color: ${colorId}`);
  }

  // 更新数据库
  await Promise.all([
    authorManager.setAuthorColorId(author, colorId),
    authorManager.setAuthorName(author, name),
  ]);

  // 广播给其他客户端
  socket.broadcast.to(padId).emit('message', {
    type: 'COLLABROOM',
    data: {
      type: 'USER_NEWINFO',
      userInfo: {userId: author, name, colorId},
    },
  });
};
```

**4. 其他客户端接收** ([collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts#L268-L278))：

```typescript
} else if (msg.type === 'USER_NEWINFO') {
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

### 6.2 用户离开处理

当用户断开连接时，在 [handleDisconnect](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L246-L287) 中处理：

```typescript
exports.handleDisconnect = async (socket) => {
  const session = sessioninfos[socket.id];

  // 仅当该作者的最后一个 socket 离开时才广播 USER_LEAVE
  const isLastSocketForAuthor = !_getRoomSockets(session.padId).some(
      (s) => sessioninfos[s.id]?.author === session.author);

  if (isLastSocketForAuthor) {
    socket.broadcast.to(session.padId).emit('message', {
      type: 'COLLABROOM',
      data: {
        type: 'USER_LEAVE',
        userInfo: {
          colorId: await authorManager.getAuthorColorId(session.author),
          userId: session.author,
        },
      },
    });
  }
};
```

客户端收到 `USER_LEAVE` 后，将该作者标记为淡出状态：

```typescript
} else if (msg.type === 'USER_LEAVE') {
  const userInfo = msg.userInfo;
  const id = userInfo.userId;
  if (userSet[id]) {
    delete userSet[userInfo.userId];
    fadeAceAuthorInfo(userInfo);  // 设置 fade: 0.5
    callbacks.onUserLeave(userInfo);
  }
}
```

---

## 7. 文本作者属性标记 (Author Attribution on Text)

### 7.1 插入时的作者标记

在 [stampAuthorOnInserts.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/stampAuthorOnInserts.ts) 中，确保每个插入操作都带有作者属性：

```typescript
export const stampAuthorOnInserts = (changeset, apoolJsonable, authorId) => {
  const pool = (new AttributePool()).fromJsonable(apoolJsonable);
  const unpacked = unpack(changeset);
  const assem = new SmartOpAssembler();
  let modified = false;

  for (const op of deserializeOps(unpacked.ops)) {
    if (op.opcode === '+') {  // 仅处理插入操作
      const attribs = AttributeMap.fromString(op.attribs, pool);
      if (!attribs.get('author')) {
        attribs.set('author', authorId);  // 补充缺失的 author 属性
        op.attribs = attribs.toString();
        modified = true;
      }
    }
    assem.append(op);
  }

  if (!modified) return {changeset, apool: apoolJsonable};
  // 返回重写后的 changeset
};
```

### 7.2 协作客户端调用时机

在 [collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts#L59-L67) 的 `prepareUserChangeset` 中调用：

```typescript
const prepareUserChangeset = () => {
  const data = editor.prepareUserChangeset();
  if (data.changeset) {
    // 确保所有插入操作都带有作者标记
    const stamped = stampAuthorOnInserts(data.changeset, data.apool, userId);
    data.changeset = stamped.changeset;
    data.apool = stamped.apool;
  }
  return data;
};
```

### 7.3 服务端验证

在 [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L859-L920) 中，服务器会验证每个变更的作者属性：

```typescript
for (const op of deserializeOps(unpack(changeset).ops)) {
  const opAuthorId = AttributeMap.fromString(op.attribs, wireApool).get('author');

  // 防止冒充其他作者
  if (opAuthorId && opAuthorId !== thisSession.author) {
    if (op.opcode === '=') {
      // 允许在已有文本上恢复作者属性（用于撤销清除作者色）
      // 但必须是该 pad 已存在的作者
      const knownAuthor = pad.pool.putAttrib(['author', opAuthorId], true) !== -1;
      if (!knownAuthor) throw new Error('unknown author');
    } else {
      // 插入操作必须是当前用户
      throw new Error(`Author ${thisSession.author} tried to submit changes as author ${opAuthorId}`);
    }
  }

  // 插入操作必须有 author 属性
  if (op.opcode === '+' && !opAuthorId) {
    throw new Error('insert without an author attribute');
  }
}
```

---

## 8. 颜色工具函数 (Color Utilities)

[colorutils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/colorutils.ts) 提供完整的颜色处理工具集：

### 8.1 颜色格式转换

```typescript
colorutils.css2triple(cssColor)     // "#fff" → [1.0, 1.0, 1.0]
colorutils.css2sixhex(cssColor)     // "#fff" → "ffffff"
colorutils.triple2css(triple)       // [1.0, 1.0, 1.0] → "#ffffff"
```

### 8.2 WCAG 2.1 可访问性支持

为确保文本与背景色有足够对比度，实现了 WCAG 2.1 标准的对比度计算：

```typescript
// 计算相对亮度
colorutils.relativeLuminance = (c) => {
  const toLinear = (v) => v <= 0.03928 ? v / 12.92 : Math.pow((v + 0.055) / 1.055, 2.4);
  return 0.2126 * toLinear(c[0]) + 0.7152 * toLinear(c[1]) + 0.0722 * toLinear(c[2]);
};

// 计算对比度比 (1-21，4.5 = AA，7.0 = AAA)
colorutils.contrastRatio = (c1, c2) => {
  const l1 = colorutils.relativeLuminance(c1);
  const l2 = colorutils.relativeLuminance(c2);
  return (Math.max(l1, l2) + 0.05) / (Math.min(l1, l2) + 0.05);
};
```

### 8.3 智能文本颜色选择

根据背景色自动选择深色或浅色文本以确保可读性：

```typescript
colorutils.textColorFromBackgroundColor = (bgcolor, skinName) => {
  const refs = skinTextColors(skinName);
  const triple = colorutils.css2triple(bgcolor);
  const ratioDark = colorutils.contrastRatio(triple, colorutils.css2triple(refs.darkRef));
  const ratioLight = colorutils.contrastRatio(triple, colorutils.css2triple(refs.lightRef));
  return ratioDark >= ratioLight ? refs.darkOut : refs.lightOut;
};
```

### 8.4 背景色可读性保障

如果背景色与文本对比度不足，自动调整背景色：

```typescript
colorutils.ensureReadableBackground = (cssColor, skinName, minContrast = 4.5) => {
  if (!colorutils.isCssHex(cssColor)) return cssColor;

  // 检查对比度是否达标
  const ratioDark = colorutils.contrastRatio(triple, dark);
  const ratioLight = colorutils.contrastRatio(triple, light);
  if (Math.max(ratioDark, ratioLight) >= minContrast) return cssColor;

  // 逐步向白色或黑色混合，直到对比度达标
  const blendTarget = ratioDark >= ratioLight ? [1, 1, 1] : [0, 0, 0];
  for (let i = 1; i <= 20; i++) {
    const blended = colorutils.blend(triple, blendTarget, i * 0.05);
    if (colorutils.contrastRatio(blended, textRef) >= minContrast) {
      return colorutils.triple2css(blended);
    }
  }
  return colorutils.triple2css(blendTarget);
};
```

---

## 9. 用户列表 UI 展示 (User List UI)

[pad_userlist.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/pad_userlist.ts) 负责用户列表的渲染：

### 9.1 颜色索引转颜色值

```typescript
// 设置当前用户信息时转换颜色
setMyUserInfo: (info) => {
  if (typeof info.colorId === 'number') {
    info.colorId = clientVars.colorPalette[info.colorId];
  }
  myUserInfo = $.extend({}, info);
  self.renderMyUserInfo();
};
```

### 9.2 用户行渲染

```typescript
const createUserRowTds = (height, data) => {
  return $()
      .add($('<td>')
          .addClass('usertdswatch')
          .append($('<div>')
              .addClass('swatch')
              .css('background', padutils.escapeHtml(data.color))  // 颜色块
              .html('&nbsp;')))
      .add($('<td>')
          .addClass('usertdname')
          .append(name))  // 用户名
      .add($('<td>')
          .addClass('activity')
          .text(data.activity));  // 活动状态
};
```

### 9.3 颜色选择器

```typescript
const showColorPicker = () => {
  const palette = pad.getColorPalette();
  const colorsList = $('#colorpickerswatches');

  // 渲染颜色选择面板
  for (let i = 0; i < palette.length; i++) {
    const li = $('<li>', {style: `background: ${palette[i]};`});
    li.appendTo(colorsList);

    li.on('click', (event) => {
      const newColorId = getColorPickerSwatchIndex($(event.target));
      pad.notifyChangeColor(newColorId);  // 点击后立即更新
    });
  }
};
```

---

## 10. 完整数据流图

### 10.1 新用户加入流程

```
浏览器连接
    ↓
CLIENT_READY (token, padId)
    ↓
[Server] securityManager.checkAccess()
    ↓
[Server] AuthorManager.getAuthor4Token()
    ├─ 存在 → 更新 lastSeen
    └─ 不存在 → createAuthor() → 随机分配 colorId
    ↓
[Server] 收集 pad.getAllAuthors() → 批量查询所有作者的 colorId/name
    ↓
[Server] 组装 clientVars:
    - colorPalette: [64种颜色]
    - userColor: 当前用户 colorId
    - userId: 当前用户 authorID
    - collab_client_vars.historicalAuthorData: {authorId: {name, colorId}}
    ↓
CLIENT_VARS → 浏览器
    ↓
[Client] collab_client 初始化:
    - tellAceAboutHistoricalAuthors() → 所有历史作者设置为 fade:0.5
    - tellAceActiveAuthorInfo() → 当前用户正常显示
    ↓
[Server] 广播 USER_NEWINFO 给其他客户端
    ↓
[Other Clients] userJoin 回调 → 更新用户列表、设置 ACE 作者颜色
```

### 10.2 颜色修改流程

```
用户在 UI 选择新颜色
    ↓
[Client] pad.notifyChangeColor(newColor)
    ↓
[Client] collab_client.updateUserInfo():
    - 本地更新 userSet[userId]
    - tellAceActiveAuthorInfo() → 编辑器立即显示新颜色
    - 发送 USERINFO_UPDATE 到服务器
    ↓
[Server] handleUserInfoUpdate():
    - 验证颜色格式
    - AuthorManager.setAuthorColorId() → 持久化
    - AuthorManager.setAuthorName() → 持久化
    - 广播 USER_NEWINFO 给其他客户端
    ↓
[Other Clients] onUpdateUserInfo 回调
    - 更新 userSet[id]
    - tellAceActiveAuthorInfo() → 更新编辑器显示
    - paduserlist.userJoinOrUpdate() → 更新用户列表
```

---

## 11. 关键设计权衡

### 11.1 颜色分配策略

| 策略 | 优点 | 缺点 |
|------|------|------|
| **完全随机分配** | 实现简单，无状态 | 可能出现颜色冲突（多个用户分到相近颜色） |
| 基于用户 ID 哈希 | 同一用户始终获得相同颜色 | 仍可能冲突 |
| 动态分配可用颜色 | 避免冲突 | 实现复杂，需要维护使用状态 |

**当前选择**：完全随机分配。对于 64 色的池，在常用协作场景（<10 人）中冲突概率较低，且用户可手动修改颜色。

### 11.2 颜色 ID 的两种形式

- **存储形式**：`colorId` 可以是数字索引（0-63）或直接存储 CSS 颜色字符串
- **原因**：允许用户选择自定义颜色（不在预设池中），通过颜色选择器的自由取色功能
- **处理**：在使用时检查 `typeof colorId === 'number'`，若是则从 colorPalette 查找

### 11.3 历史作者的淡出处理

- 历史作者（不在当前会话中）在编辑器中显示为 `fade: 0.5` 半透明
- 但在用户列表中，离开的用户会在 8 秒后完全移除
- 设计考量：编辑器中需要保留历史上下文，用户列表只需显示当前活跃用户

---

## 12. 代码溯源索引

| 功能模块 | 核心文件 | 关键函数/位置 |
|---------|---------|-------------|
| 颜色池定义 | [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts) | `getColorPalette()` [L27-L92](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L27-L92) |
| 作者创建 | [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts) | `createAuthor()` [L201-L212](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L201-L212) |
| 作者映射 | [AuthorManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts) | `mapAuthorWithDBKey()` [L117-L141](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/AuthorManager.ts#L117-L141) |
| 会话管理 | [SessionManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts) | `createSession()` [L105-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/db/SessionManager.ts#L105-L159) |
| 客户端初始化 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleClientReady()` [L1118-L1458](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1118-L1458) |
| 协作客户端 | [collab_client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts) | `getCollabClient()` [L39-L529](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/collab_client.ts#L39-L529) |
| 作者标记 | [stampAuthorOnInserts.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/stampAuthorOnInserts.ts) | `stampAuthorOnInserts()` [L31-L57](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/stampAuthorOnInserts.ts#L31-L57) |
| 颜色工具 | [colorutils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/colorutils.ts) | `ensureReadableBackground()` [L171-L192](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/colorutils.ts#L171-L192) |
| 用户列表 | [pad_userlist.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/pad_userlist.ts) | `userJoinOrUpdate()` [L490-L548](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/static/js/pad_userlist.ts#L490-L548) |
| 用户信息更新 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleUserInfoUpdate()` [L771-L803](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L771-L803) |
| 断开连接 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleDisconnect()` [L246-L287](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L246-L287) |
| 变更验证 | [PadMessageHandler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts) | `handleUserChanges()` 验证 [L859-L920](file:///d:/fz/0601-2/solo-dogfeeding/code/23-etherpad-lite/src/node/handler/PadMessageHandler.ts#L859-L920) |
