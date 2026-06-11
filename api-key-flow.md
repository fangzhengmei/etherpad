# Etherpad Lite API 与 API Key 校验流程分析

## 一、整体架构

Etherpad 存在 **两套 API 系统并存**，分别通过不同的 Express 钩子注册：

| API 系统 | 注册钩子 | 入口文件 | 路由前缀 | 说明
---|---|---|---|---
旧版 OpenAPI | `expressPreSession` | [openapi.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/hooks/express/openapi.ts) | `/api/{version}/*` `/rest/{version}/*` | 基于 `openapi-backend` 库，支持 flat 和 restful 两种路径风格
新版 REST API | `expressCreateServer` | [RestAPI.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/RestAPI.ts) | `/api/2/*` | 手写路由映射表，仅 RESTful 风格

两套系统最终都调用同一个处理核心 [APIHandler.handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIHandler.ts#L166-L238)。

### 钩子执行顺序

根据 [express.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/hooks/express.ts#L244-L252) 中的顺序：

1. `expressPreSession` —— 旧版 OpenAPI 在此阶段注册（session 之前）
2. `express-session` 中间件
3. `webaccess.checkAccess` 访问控制
4. `expressCreateServer` —— 新版 REST API 在此阶段注册（session 之后）

> **关键点**：API key 校验发生在 `APIHandler.handle()` 内部，不依赖 express-session，因此两套 API 都能正常工作。

---

## 二、版本分派机制

### 2.1 版本定义

版本定义集中在 [APIHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIHandler.ts) 的 `version` 对象中：

```
v1 → v1.1 → v1.2 → v1.2.1 → v1.2.7 → v1.2.8 → v1.2.9 → v1.2.10 → v1.2.11 → v1.2.12 → v1.2.13 → v1.2.14 → v1.2.15 → v1.3.0 → v1.3.1 (latest)
```

每个版本通过**扩展上一版本**（使用展开运算符 `...`），实现函数集是累积的。

`latestApiVersion = `'1.3.1'`。

### 2.2 各版本新增内容

| 版本 | 新增函数 | 说明
---|---|---
v1 | createGroup, createPad, getText, setText, getHTML, setHTML 等 | 基础 API
v1.1 | getAuthorName, padUsers, sendClientsMessage, listAllGroups | 补充作者名、在线用户、群发消息
v1.2 | checkToken | Token 校验
v1.2.1 | listAllPads | 列出所有 pad
v1.2.7 | createDiffHTML, getChatHistory, getChatHead | 差异 HTML、聊天历史
v1.2.8 | getAttributePool, getRevisionChangeset | 属性池、修订 changeset
v1.2.9 | copyPad, movePad | 复制/移动 pad
v1.2.10 | getPadID | 只读 ID 反查
v1.2.11 | listSavedRevisions, saveRevision, getSavedRevisionsCount | 保存的修订
v1.2.12 | appendChatMessage | 追加聊天消息
v1.2.13 | appendText | 追加文本
v1.2.14 | getStats | 统计信息
v1.2.15 | copyPadWithoutHistory | 无历史复制
v1.3.0 | 多个函数增加 authorId 参数 | 引入作者追踪
v1.3.1 | compactPad, anonymizeAuthor | 压缩 pad、匿名化作者

### 2.3 版本分派流程（新版 REST API）

在 [RestAPI.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/RestAPI.ts#L1451-L1486) 中：

1. 请求到达 `/api/2/*`
2. 从 `mapping` Map 中按 `method + path` 查找对应配置
3. 取出 `{apiVersion, functionName}`
4. 调用 `apiHandler.handle(apiVersion, functionName, fields, req, res)`

**版本选择示例**：

| HTTP Method | Path | apiVersion | functionName |
---|---|---|---
POST | `/pads | 1.3.0 | createPad |
GET | `/pads/text` | 1 | getText |
PATCH | `/pads/text` | 1.3.0 | appendText |
DELETE | `/pads` | 1 | deletePad |

### 2.4 版本分派（旧版 OpenAPI）

在 [openapi.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/hooks/express/openapi.ts) 中：

1. 对每个 API 版本生成一个 `OpenAPIBackend` 实例
2. 支持 flat 风格 (`/api/1.3.1/createPad`) 和 rest 风格 (`/rest/1.3.1/pad/create`)
3. 通过 `operationId` 找到 `funcName`
4. 调用 `apiHandler.handle(version, funcName, fields, req, res)`

---

## 三、参数解析流程

### 3.1 参数来源优先级

两套 API 都采用相同的参数合并策略，优先级从低到高：

```
path params → query string → request body
```

即 **body > query > path params**

### 3.2 新版 REST API 参数解析

位置：[RestAPI.ts#L1454-L1482](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/RestAPI.ts#L1454-L1482)

```
1. 从 req 中提取 headers, params, query
2. POST/DELETE 方法解析 body：
   - Content-Type 为 application/json 时使用 req.body
   - 否则使用 formidable 解析 form data
3. 合并：fields = Object.assign({}, params, query, formData)
4. Authorization header 特殊处理：
   fields.authorization = fields.authorization || headers.authorization
```

### 3.3 旧版 OpenAPI 参数解析

位置：[openapi.ts#L732-L760](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/hooks/express/openapi.ts#L732-L760)

逻辑与新版完全一致：

```
1. 从 c.request 提取 headers, params, query
2. POST 方法解析 body（同样区分 JSON 和 form-data
3. 合并：fields = Object.assign({}, params, query, formData)
4. Authorization header 特殊处理
```

### 3.4 参数校验

在 [APIHandler.ts#L232-L237](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIHandler.ts#L232-L237)：

```javascript
const functionParams = version[apiVersion][functionName]
    .map((field) => fields[field]);

return api[functionName].apply(this, functionParams);
```

根据版本定义的参数列表，从 `fields` 中按顺序提取参数，然后调用实际的 API 函数。

**注意**：参数列表在 `version` 对象中定义，每个函数有固定的参数顺序。如果调用时传入的参数多于定义的参数不会被传递给 API 函数。

---

## 四、Pad 操作映射

### 4.1 三层映射架构

```
┌─────────────────────────────────────────────────────┐
│  路由层 (RestAPI.ts / openapi.ts)          │
│  HTTP method + URL path → functionName       │
└───────────────────┬─────────────────────────┘
                    │
┌───────────────────▼─────────────────────────┐
│  版本层 (APIHandler.ts version 对象)      │
│  functionName → paramNames[]                  │
└───────────────────┬─────────────────────────┘
                    │
┌───────────────────▼─────────────────────────┐
│  实现层 (db/API.ts)                        │
│  functionName → 实际业务逻辑                │
└─────────────────────────────────────────────┘
```

### 4.2 Pad 操作映射表

| 操作 | HTTP Method | REST Path | API 函数 | 起始版本 |
---|---|---|---|---
创建 pad | POST | `/pads` | createPad | v1 (v1.3.0 增加 authorId)
获取文本 | GET | `/pads/text` | getText | v1
设置文本 | POST | `/pads/text` | setText | v1 (v1.3.0 增加 authorId)
追加文本 | PATCH | `/pads/text` | appendText | v1.2.13 (v1.3.0 增加 authorId)
获取 HTML | GET | `/pads/html` | getHTML | v1
设置 HTML | POST | `/pads/html` | setHTML | v1 (v1.3.0 增加 authorId)
删除 pad | DELETE | `/pads` | deletePad | v1
获取修订数 | GET | `/pads/revisions` | getRevisionsCount | v1
最后编辑时间 | GET | `/pads/lastEdited` | getLastEdited | v1
只读 ID | GET | `/pads/readonly` | getReadOnlyID | v1
公开状态 | GET/POST | `/pads/publicStatus` | getPublicStatus / setPublicStatus | v1
作者列表 | GET | `/pads/authors` | listAuthorsOfPad | v1
用户数 | GET | `/pads/usersCount` | padUsersCount | v1
用户列表 | GET | `/pads/users` | padUsers | v1.1
发送客户端消息 | POST | `/pads/clientsMessage` | sendClientsMessage | v1.1
列出所有 pad | GET | `/pads` | listAllPads | v1.2.1
创建差异 HTML | POST | `/pads/diff` | createDiffHTML | v1.2.7
聊天历史 | GET | `/pads/chatHistory` | getChatHistory | v1.2.7
聊天头部 | GET | `/pads/chatHead` | getChatHead | v1.2.7
属性池 | GET | `/pads/attributePool` | getAttributePool | v1.2.8
修订 changeset | GET | `/pads/revisionChangeset` | getRevisionChangeset | v1.2.8
复制 pad | POST | `/pads/copypad` | copyPad | v1.2.9
移动 pad | POST | `/pads/movePad` | movePad | v1.2.9
获取 padID | POST | `/pads/padId` | getPadID | v1.2.10
创建组 pad | POST | `/pads/group` | createGroupPad | v1 (v1.3.0 增加 authorId)
无历史复制 | POST | `/pads/copyWithoutHistory` | copyPadWithoutHistory | v1.2.15 (v1.3.0 增加 authorId)
保存修订 | POST | `/savedRevisions` | saveRevision | v1.2.11
列出保存的修订 | GET | `/savedRevisions` | listSavedRevisions | v1.2.11
保存的修订数 | GET | `/savedRevisions/revisionsCount` | getSavedRevisionsCount | v1.2.11
恢复修订 | PATCH | `/savedRevisions` | restoreRevision | v1.2.11 (v1.3.0 增加 authorId)
追加聊天消息 | PATCH | `/chats/messages` | appendChatMessage | v1.2.12

### 4.3 实际实现位置

Pad 相关函数在 [db/API.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/db/API.ts) 中实现，大部分函数通过 `getPadSafe()` 获取 pad 实例后调用 pad 的方法。

---

## 五、API Key 校验边界

### 5.1 认证方式选择

由 [Settings.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/utils/Settings.ts#L504) 中 `authenticationMethod` 配置决定：

- `'sso'`（默认）— OAuth2 / OIDC
- `'apikey'` — 传统 API Key

### 5.2 API Key 的初始化

位置：[APIKeyHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIKeyHandler.ts)

```
1. 模块加载时执行
2. 仅当 authenticationMethod === 'apikey' 时才初始化
3. 从 APIKEY.txt 文件读取
4. 文件不存在则生成随机 32 位字符串并写入文件
5. apikey 变量导出供 APIHandler 使用
```

### 5.3 校验发生位置

**唯一的 API Key 校验发生在 [APIHandler.handle()](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIHandler.ts#L180-L219) 函数中。

校验顺序：

```
1. 检查 apiVersion 是否存在 → 不存在抛 404
2. 检查 functionName 是否存在 → 不存在抛 404
3. 【API Key 校验】 → 不通过抛 401
4. padID / padName  sanitize
5. 提取函数参数
6. 调用实际 API 函数
```

> **边界结论**：API Key 校验在**版本和函数存在性检查之后**，**参数 sanitize 和业务逻辑之前**。

### 5.4 API Key 模式（apikey）

当 `apikey !== null && apikey.trim().length > 0` 时走此分支：

```
1. 从三个来源合并获取 apikey：
   fields.apikey || fields.api_key || fields.authorization
2. 使用 crypto.timingSafeEqual 进行常量时间比较
3. 不匹配则抛出 401 Unauthorized: "no or wrong API Key"
```

**API Key 的传递方式**：

| 位置 | 参数名 | 说明
---|---|---
Query String | `apikey` | 主要方式
Query String | `api_key` | 别名，兼容旧客户端
HTTP Header | `Authorization` | Authorization header

### 5.5 SSO 模式（sso）

当 apikey 为空时走此分支：

```
1. 检查 req.headers.authorization 是否存在
2. 解析 Bearer token
3. 使用 RS256 算法验证 JWT 签名
4. 分两种情况：
   a. client credentials 模式：sub 在 clientIds 列表中即可
   b. authorization code 模式：必须有 admin: true claim
5. 验证失败统一返回 "no or wrong OAuth token"
```

### 5.6 安全特性

1. **常量时间比较**：使用 `crypto.timingSafeEqual` 防止时序攻击
2. **统一错误信息**：SSO 模式下无论什么原因失败都返回相同错误字符串，避免泄露信息
3. **参数污染防护**：Authorization header 只在字段值为 falsy 时才 fallback，不会覆盖已有的值

### 5.7 校验边界总结

```
请求到达
   │
   ▼
Express 路由匹配
   │
   ▼
参数解析合并（params + query + body）
   │
   ▼
apiHandler.handle() 被调用
   │
   ├─► 版本存在性检查（404）
   │
   ├─► 函数存在性检查（404）
   │
   ├─► ────────────────────────────┐
   │                                   │
   │   ┌────── authenticationMethod? ──────┐│
   │   │                            ││
   │   ▼                            ▼│
   │  apikey 模式                sso 模式
   │   │                            ││
   │   ├─ apikey / api_key /       ├─ Authorization header
   │   │   authorization 合并取值     ││
   │   │                            ││
   │   ▼                            ▼│
   │  timingSafeEqual 比较         JWT 验证
   │   │                            ││
   │   └────── 都失败抛 401 ────────┘│
   │                                   │
   ├─► padID / padName sanitize     │
   │                                   │
   ├─► 参数提取                      │
   │                                   │
   └─► 调用 db/API 函数 ◄────────────┘
```

---

## 六、关键代码索引

| 模块 | 文件 | 关键函数/变量 |
---|---|---
路由层（新） | [RestAPI.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/RestAPI.ts) | `mapping` Map, `expressCreateServer()`
路由层（旧） | [openapi.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/hooks/express/openapi.ts) | `generateDefinitionForVersion()`, `expressPreSession()`
核心处理 | [APIHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIHandler.ts) | `version`, `handle()`
API Key 管理 | [APIKeyHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIKeyHandler.ts) | `apikey`, `APIFields`
业务实现 | [db/API.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/db/API.ts) | 各 API 函数实现
配置 | [utils/Settings.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/utils/Settings.ts) | `authenticationMethod`
钩子注册 | [ep.json](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/ep.json) | `restApi`, `apicalls` 等 part
Express 框架 | [hooks/express.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/hooks/express.ts) | `expressPreSession`, `expressCreateServer` 钩子调用点
