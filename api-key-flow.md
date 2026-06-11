# Etherpad Lite API 流程分析（按机制和行为整理）

## 一、两套对外接口的入口机制与差异

Etherpad 同时注册了两套互相独立的对外 HTTP API，它们在 Express 生命周期的不同阶段挂载，路由形态和请求处理细节都不同，但最终都汇入同一个业务函数。

### 1.1 旧版 OpenAPI 入口

挂载位置：[expressPreSession 钩子](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/hooks/express/openapi.ts#L677-L852)

**挂载阶段**：在 express-session 和 webaccess.checkAccess 中间件之前。意味着旧版 API 不依赖 session，也不会经过页面访问控制检查。

**路由生成策略**：对每一个 API 版本 × 每一种路径风格，单独实例化一个 `OpenAPIBackend` 对象并挂载：

```
版本数（15个：v1 ~ v1.3.1）
× 路径风格（2种：flat + rest）
= 30 个独立的 API 路由根
```

路由根计算方式：`/${style}/${version}`，即：
- `/api/1`, `/api/1.1`, ..., `/api/1.3.1`（flat 风格）
- `/rest/1`, `/rest/1.1`, ..., `/rest/1.3.1`（rest 风格）

**请求方法**：flat 风格下每个函数名同时接受 GET 和 POST（如 `/api/1.3.1/createPad` 无论 GET 还是 POST 都会命中 `createPadUsingGET` 或 `createPadUsingPOST`，但两者指向同一个 handler 实现）。REST 风格同理，每个资源操作同样双注册 GET/POST。**不区分 PUT/PATCH/DELETE**，这些方法在旧版入口会被 openapi-backend 路由到 `notFound`，返回 `{code:3, message:"no such function"}`。

### 1.2 新版 REST API 入口

挂载位置：[expressCreateServer 钩子](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/RestAPI.ts#L335-L1551)

**挂载阶段**：在 express-session 和 webaccess.checkAccess 之后（理论上会经过会话中间件，但实际 API 校验在 handler 内部，不使用 session）。

**路由生成策略**：单一的 `/api/2` 前缀，内部由一个手写的 `Map<HTTP_METHOD, { path → {apiVersion, functionName} }>` 做分发，覆盖所有历史版本中需要暴露的函数：

- 方法维度：GET / POST / PUT / DELETE / PATCH 各自独立一组路径
- 路径维度：RESTful 资源风格，如 `/pads/text`、`/pads/html`、`/savedRevisions` 等
- 版本绑定：每个 `(method, path)` 元组在 mapping 里写死了对应的 `apiVersion`（例如 GET `/pads/text` 绑 v1，PATCH `/pads/text` 绑 v1.3.0）

**请求方法**：严格区分 5 种方法。`PATCH /api/2/pads/text` 走 `appendText`（v1.3.0），而 `GET /api/2/pads/text` 走 `getText`（v1），路径相同但方法不同即命中不同函数。在 mapping 中找不到 `(method, path)` 组合时，直接返回 `{code:1, message:"not found"}`。

### 1.3 两套入口的本质差异对照

| 维度 | 旧版 OpenAPI | 新版 REST API |
|---|---|---|
| 挂载钩子 | `expressPreSession`（session 前） | `expressCreateServer`（session 后） |
| 路由前缀 | `/api/{v}` + `/rest/{v}` 共 30 个根 | 单一根 `/api/2` |
| 版本选择方式 | URL 路径里写版本号（`/api/1.3.0/...`） | mapping 表按 path+method 固定绑定版本 |
| 请求方法支持 | GET + POST 双注册，PUT/PATCH/DELETE 返回 no such function | 5 种方法全部独立映射 |
| 路径风格 | flat（函数名即路径）+ REST（资源/动作） | 仅 REST 风格 |
| 路由匹配实现 | openapi-backend 库按 operationId 路由 | 手写 Map 直接查表 |
| 匹配失败返回 | code 3 / "no such function"（走 404→code3） | code 1 / "not found"（直接 json） |
| CORS 头 | REST 风格分支会加 `Access-Control-Allow-Origin: *` | 不加 CORS 头 |

---

## 二、不同请求方法的参数解析边界

两套入口在参数合并上的原则一致（**body > query > path params**），但在「哪些方法会读 body」以及「body 解析条件」这两个边界上行为不同。请求中的额外 headers 不会被合并进 fields，只有 Authorization header 被单独提取。

### 2.1 参数合并总规则（两套入口相同）

```
fields = Object.assign({},
  pathParams,    // URL 路径参数，优先级最低
  queryString,   // ?k=v 查询串，中间
  body           // 请求体，优先级最高
)
```

Authorization header 单独处理：
```
if headers.authorization 存在:
  fields.authorization = fields.authorization || headers.authorization
```
只做 fallback（falsy 时才覆盖），不会覆盖用户通过 query/body 已经传入的 authorization 字段。

### 2.2 旧版 OpenAPI 的 body 解析边界

代码位置：[openapi.ts#L736-L752](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/hooks/express/openapi.ts#L736-L752)

**触发条件：只在 method === 'post'（不区分大小写）时读 body。**

GET / PUT / PATCH / DELETE：`formData = {}`，即使客户端带了 body 也完全忽略。

Body 解析分支判断：

```
if (req.body && typeof req.body === 'object')
    → 用 req.body（express.json() 已解析好的对象）
else
    → new IncomingForm().parse(req)，取 fields 部分
```

这里的判断**不看 Content-Type**，只看 `req.body` 是否存在且是对象。意味着：
- 只要上游有任意中间件往 `req.body` 写了对象（即使是 `text/plain`），就走第一条分支
- POST 一个不带 Content-Type 的裸请求体，会 fall 到 formidable
- form-data 形式的数组字段会被拍平（取数组第一个元素）

### 2.3 新版 REST API 的 body 解析边界

代码位置：[RestAPI.ts#L1458-L1472](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/RestAPI.ts#L1458-L1472)

**触发条件：method === 'post' 或 method === 'delete'。**

GET / PUT / PATCH：formData = {}，body 被忽略。

⚠️ **重点行为差异**：DELETE 在新版会解析 body（旧版不会）。用 DELETE 带 JSON body 提交 `padID` 的写法只在新版生效，在旧版是读不到的。

Body 解析分支判断：

```
if (content-type 缺失 || content-type 以 application/json 开头)
    → formData = req.body
else
    → formidable 解析
```

这里**显式检查 Content-Type**，而不是检查 `req.body` 是否存在。同时注意 `/api/2` 路由之前在同一个 expressCreateServer 钩子内部注册了 `app.use(express.json())`，所以 JSON body 已经被解析好了。

### 2.4 参数解析边界汇总表

| 方法 | 旧版 OpenAPI 解析 body？ | 新版 REST API 解析 body？ | 备注 |
|---|---|---|---|
| GET | ❌ 否 | ❌ 否 | 一致，参数只能来自 path + query |
| POST | ✅ 是 | ✅ 是 | 一致，但 JSON 判定条件不同（旧版看 req.body，新版看 Content-Type） |
| PUT | ❌ 否 | ❌ 否 | 两套入口都不解析 PUT 的 body；目前 mapping 里 PUT 没有注册任何路径 |
| DELETE | ❌ 否 | ✅ 是 | **不一致**：旧版 DELETE `/api/1.3.1/deletePad` 的 padID 必须走 query；新版 DELETE `/api/2/pads` 的 padID 可以走 JSON body |
| PATCH | ❌ 否（且旧版无 PATCH 路由） | ❌ 否（但 method 本身在 mapping 里有路径） | 新版 PATCH 虽然有路由，但不解析 body，参数只能来自 path + query |

### 2.5 版本层的参数二次提取

经过入口层合并后，fields 对象里可能混有各种多余字段（包括 apikey、api_key、authorization 等非业务字段）。真正传给业务函数的参数在 [APIHandler.ts#L232-L237](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIHandler.ts#L232-L237) 做二次过滤：

```
paramNames = version[apiVersion][functionName]   // e.g. ['padID', 'text', 'authorId']
functionParams = paramNames.map(name => fields[name])
api[functionName].apply(this, functionParams)
```

只按声明顺序取声明的参数，多传的字段被静默丢弃；如果字段在 fields 里不存在，则传 `undefined` 给业务函数，由业务函数自己做类型校验（例如 setText 里 `if (typeof text !== 'string')` 抛 apierror）。

---

## 三、API Key 校验的边界和分支行为

两套入口最终都调用 `apiHandler.handle(apiVersion, functionName, fields, req)`，鉴权就在这个函数内部完成。校验的代码位置：[APIHandler.ts#L180-L219](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIHandler.ts#L180-L219)

### 3.1 校验在整个调用链中的位置

```
1. 版本号存在性检查 → 不存在抛 404（→ 包装成 code:3）
2. 函数名存在性检查 → 不存在抛 404（→ 包装成 code:3）
3. ── 鉴权检查开始 ──（这里抛的全是 401，→ 包装成 code:4）
4. padID / padName sanitize
5. 按版本声明提取参数
6. 调用 db/API.ts 的业务函数
```

**边界结论**：鉴权在「版本+函数都确认存在」之后、「任何 pad 操作和业务逻辑」之前执行。意味着即使请求了一个不存在的函数，也不会消耗 API key 校验——但只要函数名有效，不管 padID 是否存在，鉴权都会执行。

### 3.2 鉴权走哪条分支由什么决定

由启动时 `apikey` 全局变量是否被初始化决定。初始化逻辑在 [APIKeyHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIKeyHandler.ts#L21-L35)：

```
settings.authenticationMethod === 'apikey'
    → 读 APIKEY.txt，不存在则生成随机 32 字节 → apikey = <非空字符串>
settings.authenticationMethod === 'sso'（默认）
    → 上面那段 if 不执行 → apikey = null（模块导出的初始值）
```

于是进入 `handle()` 时：

| apikey 变量状态 | 走的分支 |
|---|---|
| `!= null && trim().length > 0` | API Key 模式（下面 3.3） |
| `== null || trim().length == 0` | SSO 模式（下面 3.4） |

这意味着**如果把 authenticationMethod 改成 `apikey` 但手工把 APIKEY.txt 清空成空文件或纯空白**，虽然 apikey 变量不是 null，但 trim 后长度为 0，**会走 SSO 分支**——这是个隐晦的配置陷阱。

### 3.3 API Key 模式下的行为

```
步骤 1：拼接入参
  fields.apikey = fields.apikey || fields.api_key || fields.authorization
  （从三个来源取第一个非 falsy 的值）

步骤 2：常量时间比较
  Buffer.from(提供值) vs Buffer.from(apikey.trim())
  先比长度，再用 crypto.timingSafeEqual 比内容

步骤 3：结果
  匹配 → 放行
  不匹配 → 401 Unauthorized，消息 = "no or wrong API Key"
```

所以密钥的三种等价传法：
- `?apikey=xxx`
- `?api_key=xxx`（别名，兼容旧客户端）
- `Authorization: Bearer xxx` 或者直接 `Authorization: xxx`（整个 header 原样比较，不会剥 `Bearer ` 前缀）

### 3.4 SSO 模式下的行为（重点：缺信息时返回什么）

SSO 模式分两段判断，任何一段失败都返回不同的 401 消息：

```
段 A：有没有 Authorization header
  if (!req.headers.authorization)
      → 401 / "no or wrong API Key"

段 B：JWT 验证 try/catch 块内
  1. 剥 Bearer 前缀 → jwtToCheck
  2. jwtVerify(jwtToCheck, publicKey, {algorithms:['RS256']})
     任何失败 → catch → 401 / "no or wrong OAuth token"
  3. 验证通过后分两种子模式：
     a) sub 在 clientIds 列表 → client credentials 模式，直接放行
     b) 否则 → authorization code 模式，需要 verified.admin === true
         不满足 → 显式 throw 401("admin claim missing or not true")
                 → 被 catch 捕获 → 401 / "no or wrong OAuth token"
```

#### SSO 模式下各场景精确返回值：

| 场景 | HTTP 状态 | response.code | response.message |
|---|---|---|---|
| **完全没有 Authorization header** | 401 | 4 | **"no or wrong API Key"** |
| Authorization header 存在但不是 Bearer JWT（格式错、解析失败） | 401 | 4 | "no or wrong OAuth token" |
| JWT 签名不对 / 过期 / 算法不对 | 401 | 4 | "no or wrong OAuth token" |
| JWT 有效但 sub 不在 clientIds 列表，且 admin claim 不存在 / 为 false | 401 | 4 | "no or wrong OAuth token"（显式 throw 的消息被 catch 吞掉了） |
| JWT 有效且 admin=true / client credentials | - | - | 放行 |

**两个容易混淆的点**：
1. SSO 模式下「缺 Authorization header」返回的消息仍然写着 **"no or wrong API Key"**，不是 "no or wrong OAuth token"——这是代码里段 A 和段 B 写了两条不同消息的直接后果。
2. admin claim 缺失时内部抛出的消息是 `"admin claim missing or not true"`，但被 catch 块统一替换成 `"no or wrong OAuth token"`，外部永远看不到具体原因——这是故意做的信息隐藏。

### 3.5 401 错误的最终响应格式

无论上面哪条路径抛出 401，最终都会被两套入口各自的错误处理 switch 命中 `case 401`：
```
response = { code: 4, message: err.message, data: null }
```
所以对外的最终形态都是：
```
HTTP 401
{ "code": 4, "message": <上述区分的消息>, "data": null }
```

---

## 四、Pad 操作映射机制

### 4.1 三层映射模型

整个调用链是一个三次查表的过程，每一层缩小范围：

```
第 1 层：入口路由表
  旧版：OpenAPIBackend.definition.paths[url][method] → operationId
  新版：mapping[method][path]            → {apiVersion, functionName}
  产出：functionName（以及 apiVersion）

第 2 层：版本参数表
  version[apiVersion][functionName] → [paramName1, paramName2, ...]
  产出：参数名列表（按调用顺序）

第 3 层：业务实现表
  api[functionName] → 业务函数引用
  产出：真正执行的函数
```

两层入口共用第 2 层和第 3 层，所以同一个 functionName 在新旧入口下最终跑的是同一段业务代码。

### 4.2 Pad 相关操作的映射对照（从 HTTP 到业务函数的完整链）

| 操作意图 | 新版 REST API（method, path） | 旧版 flat（URL） | 绑定版本 | 业务函数 | 参数列表（按调用顺序） |
|---|---|---|---|---|---|
| 创建 pad | POST `/api/2/pads` | POST `/api/1.3.0/createPad` | 1.3.0 | createPad | padID, text, authorId |
| 创建组 pad | POST `/api/2/pads/group` | POST `/api/1.3.0/createGroupPad` | 1.3.0 | createGroupPad | groupID, padName, text, authorId |
| 取 pad 文本 | GET `/api/2/pads/text` | GET `/api/1/getText` | 1 | getText | padID, rev |
| 设置 pad 文本 | POST `/api/2/pads/text` | POST `/api/1.3.0/setText` | 1.3.0 | setText | padID, text, authorId |
| 追加 pad 文本 | PATCH `/api/2/pads/text` | POST `/api/1.3.0/appendText` | 1.3.0 | appendText | padID, text, authorId |
| 取 pad HTML | GET `/api/2/pads/html` | GET `/api/1/getHTML` | 1 | getHTML | padID, rev |
| 设置 pad HTML | POST `/api/2/pads/html` | POST `/api/1.3.0/setHTML` | 1.3.0 | setHTML | padID, html, authorId |
| 删除 pad | DELETE `/api/2/pads` | POST `/api/1/deletePad` | 1 | deletePad | padID, deletionToken |
| 取修订数 | GET `/api/2/pads/revisions` | GET `/api/1/getRevisionsCount` | 1 | getRevisionsCount | padID |
| 取最后编辑时间 | GET `/api/2/pads/lastEdited` | GET `/api/1/getLastEdited` | 1 | getLastEdited | padID |
| 只读 ID | GET `/api/2/pads/readonly` | GET `/api/1/getReadOnlyID` | 1 | getReadOnlyID | padID |
| 公开状态（查） | GET `/api/2/pads/publicStatus` | GET `/api/1/getPublicStatus` | 1 | getPublicStatus | padID |
| 公开状态（设） | POST `/api/2/pads/publicStatus` | POST `/api/1/setPublicStatus` | 1 | setPublicStatus | padID, publicStatus |
| 作者列表 | GET `/api/2/pads/authors` | GET `/api/1/listAuthorsOfPad` | 1 | listAuthorsOfPad | padID |
| 在线用户数 | GET `/api/2/pads/usersCount` | GET `/api/1/padUsersCount` | 1 | padUsersCount | padID |
| 在线用户列表 | GET `/api/2/pads/users` | GET `/api/1.1/padUsers` | 1.1 | padUsers | padID |
| 群发客户端消息 | POST `/api/2/pads/clientsMessage` | POST `/api/1.1/sendClientsMessage` | 1.1 | sendClientsMessage | padID, msg |
| 列出所有 pad | GET `/api/2/pads` | GET `/api/1.2.1/listAllPads` | 1.2.1 | listAllPads | （无参数） |
| 差异 HTML | POST `/api/2/pads/diff` | POST `/api/1.2.7/createDiffHTML` | 1.2.7 | createDiffHTML | padID, startRev, endRev |
| 聊天历史 | GET `/api/2/pads/chatHistory` | GET `/api/1.2.7/getChatHistory` | 1.2.7 | getChatHistory | padID, start, end |
| 聊天头 | GET `/api/2/pads/chatHead` | GET `/api/1.2.7/getChatHead` | 1.2.7 | getChatHead | padID |
| 属性池 | GET `/api/2/pads/attributePool` | GET `/api/1.2.8/getAttributePool` | 1.2.8 | getAttributePool | padID |
| 修订 changeset | GET `/api/2/pads/revisionChangeset` | GET `/api/1.2.8/getRevisionChangeset` | 1.2.8 | getRevisionChangeset | padID, rev |
| 复制 pad | POST `/api/2/pads/copypad` | POST `/api/1.2.9/copyPad` | 1.2.9 | copyPad | sourceID, destinationID, force |
| 移动 pad | POST `/api/2/pads/movePad` | POST `/api/1.2.9/movePad` | 1.2.9 | movePad | sourceID, destinationID, force |
| 只读 ID 反查 | POST `/api/2/pads/padId` | POST `/api/1.2.10/getPadID` | 1.2.10 | getPadID | roID |
| 列出保存的修订 | GET `/api/2/savedRevisions` | GET `/api/1.2.11/listSavedRevisions` | 1.2.11 | listSavedRevisions | padID |
| 保存修订 | POST `/api/2/savedRevisions` | POST `/api/1.2.11/saveRevision` | 1.2.11 | saveRevision | padID, rev |
| 保存的修订数 | GET `/api/2/savedRevisions/revisionsCount` | GET `/api/1.2.11/getSavedRevisionsCount` | 1.2.11 | getSavedRevisionsCount | padID |
| 恢复修订 | PATCH `/api/2/savedRevisions` | PATCH 旧版不支持 / 用 POST `/api/1.3.0/restoreRevision` | 1.3.0 | restoreRevision | padID, rev, authorId |
| 追加聊天消息 | PATCH `/api/2/chats/messages` | PATCH 旧版不支持 / 用 POST `/api/1.2.12/appendChatMessage` | 1.2.12 | appendChatMessage | padID, text, authorID, time |
| 复制（无历史） | POST `/api/2/pads/copyWithoutHistory` | POST `/api/1.3.0/copyPadWithoutHistory` | 1.3.0 | copyPadWithoutHistory | sourceID, destinationID, force, authorId |

从表中能看到的机制：
- v1.3.0 版本中 7 个写操作函数（setText / setHTML / appendText / createPad / createGroupPad / copyPadWithoutHistory / restoreRevision）都追加了 authorId 作为最后一个参数，用于追踪操作来源——在旧版入口用 v1 调用这些函数时，因为 v1 的参数列表里没有 authorId，所以 authorId 不会被传入，业务函数里 authorId 参数拿到的是默认空串。
- 同一个 path 在不同 method 下绑定的版本和函数可以完全不同（典型的 `/pads/text`：GET→v1 getText, POST→v1.3.0 setText, PATCH→v1.3.0 appendText）。

---

## 五、代码索引

| 关注点 | 文件 | 行范围 |
|---|---|---|
| 旧版 OpenAPI 入口 + 参数解析 + 错误包装 | [hooks/express/openapi.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/hooks/express/openapi.ts) | L677-L852 |
| 新版 REST API 入口 + mapping 表 + 参数解析 + 错误包装 | [handler/RestAPI.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/RestAPI.ts) | L25-L1551 |
| 版本定义 + handle()（鉴权 + 参数提取 + 调用业务） | [handler/APIHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIHandler.ts) | L33-L238 |
| apikey 全局变量初始化 | [handler/APIKeyHandler.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/handler/APIKeyHandler.ts) | L1-L35 |
| authenticationMethod 默认值（'sso'） | [utils/Settings.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/utils/Settings.ts) | L499-L504 |
| Pad 类业务函数实现（getText / setText / getHTML / 等） | [db/API.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/db/API.ts) | 全文 |
| Express 钩子执行顺序（expressPreSession → session → webaccess → expressCreateServer） | [hooks/express.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/6-etherpad-lite/src/node/hooks/express.ts) | L244-L252 |
