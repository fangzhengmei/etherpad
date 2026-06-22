# Etherpad Lite i18n 与本地化加载流程梳理

本文从代码实现角度梳理语言资源的**选择**、**加载**和**页面使用**三个阶段的完整流程。

## 目录

- [1. 总体架构概览](#1-总体架构概览)
- [2. 阶段一：语言资源构建（服务端启动期）](#2-阶段一语言资源构建服务端启动期)
  - [2.1 语言文件扫描与合并](#21-语言文件扫描与合并)
  - [2.2 自定义翻译覆盖（customLocaleStrings）](#22-自定义翻译覆盖customlocalestrings)
  - [2.3 可用语言列表与语言索引](#23-可用语言列表与语言索引)
- [3. 阶段二：语言选择（优先级链）](#3-阶段二语言选择优先级链)
  - [3.1 服务端首屏渲染：Accept-Language 协商](#31-服务端首屏渲染accept-language-协商)
  - [3.2 客户端初始化：Cookie → 浏览器语言 → 英文兜底](#32-客户端初始化cookie--浏览器语言--英文兜底)
  - [3.3 URL 查询参数（?lang=）](#33-url-查询参数lang)
  - [3.4 Pad 级强制设置（padOptions.lang）](#34-pad-级强制设置padoptionslang)
  - [3.5 用户手动切换（语言下拉菜单）](#35-用户手动切换语言下拉菜单)
- [4. 阶段三：语言资源加载（客户端 html10n）](#4-阶段三语言资源加载客户端-html10n)
  - [4.1 语言索引入口（locales.json）](#41-语言索引入口localesjson)
  - [4.2 Loader 加载流程](#42-loader-加载流程)
  - [4.3 语言代码规范化与回退链](#43-语言代码规范化与回退链)
  - [4.4 Import 规则（链式加载）](#44-import-规则链式加载)
- [5. 阶段四：翻译文本的页面应用](#5-阶段四翻译文本的页面应用)
  - [5.1 DOM 标记方式（data-l10n-id）](#51-dom-标记方式data-l10n-id)
  - [5.2 翻译键与属性后缀约定](#52-翻译键与属性后缀约定)
  - [5.3 宏系统与复数规则（plural）](#53-宏系统与复数规则plural)
  - [5.4 参数插值（{{variable}}）](#54-参数插值variable)
  - [5.5 无障碍支持（aria-label 自动填充）](#55-无障碍支持aria-label-自动填充)
- [6. Admin SPA 的独立 i18n 体系（i18next）](#6-admin-spa-的独立-i18n-体系i18next)
  - [6.1 初始化与语言检测](#61-初始化与语言检测)
  - [6.2 懒加载后端（LazyImportPlugin）](#62-懒加载后端lazyimportplugin)
  - [6.3 React 组件中的使用](#63-react-组件中的使用)
- [7. 关键文件索引](#7-关键文件索引)

---

## 1. 总体架构概览

Etherpad Lite 存在**两套并行的 i18n 体系**：

| 体系 | 适用范围 | 核心技术 |
|------|---------|---------|
| **Pad 核心 UI** | 编辑器页面、首页、时间轴 | 自研 html10n 库 + data-l10n-id 属性 |
| **Admin 管理面板** | `/admin` 下的 React SPA | i18next + react-i18next + LanguageDetector |

两者共享同一套语言源文件（`src/locales/*.json`），但加载、选择、应用的链路完全独立。

---

## 2. 阶段一：语言资源构建（服务端启动期）

发生在 Express 启动前的 `expressPreSession` 钩子中，一次性构建所有语言资源。

### 2.1 语言文件扫描与合并

入口函数：`getAllLocales()` [i18n.ts#L16-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/hooks/i18n.ts#L16-L104)

**扫描顺序（后者覆盖前者）：**

1. **核心语言包**：`src/locales/` 目录下所有 `.json` 文件（文件名必须是有效的 BCP 47 语言代码，由 `languages4translatewiki.isValid()` 验证）
2. **插件语言包**：遍历所有已安装插件的 `locales/` 目录（排除 `ep_etherpad-lite` 自身，避免重复加载核心）
3. **自定义覆盖**：`settings.json` 中的 `customLocaleStrings`

```
核心 locales/  →  各插件 ep_*/locales/  →  settings.customLocaleStrings
     ↓                  ↓                          ↓
  浅合并覆盖       可覆盖核心翻译            可覆盖一切
```

合并使用 `_.extend()` 执行浅拷贝，因此同 key 后加载者覆盖先加载者。

### 2.2 自定义翻译覆盖（customLocaleStrings）

[i18n.ts#L71-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/hooks/i18n.ts#L71-L101)

格式要求：
```json
{
  "customLocaleStrings": {
    "en": {
      "pad.social.description": "My custom description"
    },
    "zh-hans": {
      "pad.social.description": "我的自定义描述"
    }
  }
}
```

校验逻辑：
- 顶层必须是对象（语言代码 → 翻译键值对映射）
- 每层值必须是字符串
- 若输入了未知语言代码，会尝试模糊匹配并抛出"可能你想要 xxx"的错误提示

### 2.3 可用语言列表与语言索引

**availableLangs** [i18n.ts#L108-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/hooks/i18n.ts#L108-L122)：
- 调用 `languages4translatewiki.getLanguageInfo(langcode)` 获取每种语言的 `nativeName`（母语名称）和 `direction`（ltr/rtl）
- 按 `nativeName` 字母顺序排序，用于前端语言下拉菜单的展示顺序

**localeIndex** [i18n.ts#L125-L131](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/hooks/i18n.ts#L125-L131)：
- 英文（en）完整内联所有翻译字符串（体积较小且是最终兜底）
- 其他语言仅保留路径引用 `"zh-hans": "locales/zh-hans.json"`，按需懒加载

---

## 3. 阶段二：语言选择（优先级链）

语言选择是一个**多层级、按优先级覆盖**的链条，不同场景走不同分支：

```
┌─────────────────────────────────────────────────────────────┐
│  优先级从高到低：                                              │
│                                                             │
│  1. URL ?lang=xxx 参数           （仅 Pad 页面）              │
│  2. Pad 级 padOptions.lang       （管理员强制设置）           │
│  3. Cookie: language             （用户上次手动选择）         │
│  4. navigator.language           （浏览器/OS 语言）           │
│  5. Accept-Language 请求头       （服务端渲染 HTML lang）     │
│  6. 'en' 兜底                  （最终防线）                  │
└─────────────────────────────────────────────────────────────┘
```

### 3.1 服务端首屏渲染：Accept-Language 协商

**使用场景**：生成 HTML 的 `<html lang="..." dir="...">` 属性、social meta 标签（og:description、og:locale 等），以及无 Cookie 的首次访问。

**模板侧代码**（以 pad.html 为例）[pad.html#L6-L8](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/templates/pad.html#L6-L8)：
```ejs
var renderLang = (req && typeof req.acceptsLanguages === 'function'
  && req.acceptsLanguages(Object.keys(langs))) || 'en';
var renderDir = (langs[renderLang] && langs[renderLang].direction === 'rtl') ? 'rtl' : 'ltr';
```

**Social Meta 专用协商函数** [socialMeta.ts#L113-L119](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/utils/socialMeta.ts#L113-L119)：
```typescript
const negotiateRenderLang = (req, availableLangs) => {
  if (req && typeof req.acceptsLanguages === 'function') {
    const negotiated = req.acceptsLanguages(Object.keys(availableLangs));
    if (negotiated) return negotiated;
  }
  return 'en';
};
```

description 翻译的三级回退 [socialMeta.ts#L33-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/utils/socialMeta.ts#L33-L52)：
1. 精确匹配（如 `de-AT`）
2. 主语言子标签回退（如 `de-AT` → `de`）
3. 英文兜底（`en`）

### 3.2 客户端初始化：Cookie → 浏览器语言 → 英文兜底

核心代码：[l10n.ts#L1-L18](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/l10n.ts#L1-L18)

```typescript
// 1. 尝试从 Cookie 读取（支持 cookiePrefix 命名空间）
const cp = (clientVars?.cookiePrefix || '').replace(/regex-escaped/g, ...);
let language = document.cookie.match(new RegExp(`${cp}language=((\\w{2,3})(-\\w+)?)`))
    || document.cookie.match(/language=((\\w{2,3})(-\\w+)?)/);

// 2. html10n 完成资源索引后调用 localize
html10n.mt.bind('indexed', () => {
  html10n.localize([regexpLang, navigator.language, 'en']);  // 优先级数组
});

// 3. 翻译完成后写回 DOM 属性
html10n.mt.bind('localized', () => {
  document.documentElement.lang = html10n.getLanguage();
  document.documentElement.dir = html10n.getDirection();
});
```

**关键点**：`localize()` 接受一个**优先级数组**，会按顺序加载并合并——后面的语言作为前面语言的缺失翻译的补充。最终生效的语言是数组中最后一个成功加载的语言。

### 3.3 URL 查询参数（?lang=）

定义在 Pad 页面的 `getParameters` 中 [pad.ts#L184-L193](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/pad.ts#L184-L193)：

```typescript
{
  name: 'lang',
  checkVal: null,
  callback: (val) => {
    html10n.localize([val, 'en']);        // 立即切换语言
    Cookies.set(`${prefix}language`, val); // 写入 Cookie 持久化
  },
}
```

在 `getParams()` 中，URL 参数的优先级**高于**服务端下发的 `padOptions`，避免双重触发 `localize` 造成竞态。

### 3.4 Pad 级强制设置（padOptions.lang）

当管理员为某个 Pad 全局设置了语言后，会通过 Socket.io 的 `CLIENT_VARS` 消息下发到 `clientVars.padOptions.lang`。

读取路径 [pad.ts#L560-L565](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/pad.ts#L560-L565)：
```typescript
const effectiveOptions = $.extend(true, {}, pad.padOptions);
const overrides = getMyViewOverrides(); // 合并 Cookie 中的用户偏好
for (const key of ['showChat', 'alwaysShowChat', 'chatAndUsers', 'lang']) {
  if (overrides[key] != null) effectiveOptions[key] = overrides[key];
}
```

应用入口 [pad.ts#L934](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/pad.ts#L934)：
```typescript
if (effectiveOptions.lang) pad.applyLanguage(effectiveOptions.lang);
```

### 3.5 用户手动切换（语言下拉菜单）

调用链：
1. 用户在 "设置" 面板中选择语言 → 触发 `handleOptionsChange`
2. `applyOptionsChange()` 检测到 `effectiveOptions.lang` 变化
3. 调用 `pad.applyLanguage(lang)` [pad.ts#L673-L677](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/pad.ts#L673-L677)：
```typescript
applyLanguage: (lang) => {
  html10n.localize([lang, 'en']);          // 重新加载并应用翻译
  $('#languagemenu').val(lang);            // 同步下拉菜单 UI
  if ($('select').niceSelect) $('select').niceSelect('update'); // 刷新样式
},
```

同时通过 `setMyViewLanguage()` [pad.ts#L644](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/pad.ts#L644) 将选择写入 Cookie 持久化。

---

## 4. 阶段三：语言资源加载（客户端 html10n）

### 4.1 语言索引入口（locales.json）

所有需要翻译的 HTML 模板中都包含这一行：

```html
<!-- pad.html -->
<link rel="localizations" type="application/l10n+json" href="../locales.json" />

<!-- index.html -->
<link rel="localizations" type="application/l10n+json" href="locales.json">

<!-- timeslider.html -->
<link rel="localizations" type="application/l10n+json" href="../../locales.json" />
```

`locales.json` 的内容由服务端的 `generateLocaleIndex()` 生成（[i18n.ts#L125-L131](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/hooks/i18n.ts#L125-L131)）：
```json
{
  "en": {
    "pad.social.description": "Real-time collaborative document editing",
    "...": "..."
  },
  "zh-hans": "locales/zh-hans.json",
  "de": "locales/de.json",
  "...": "..."
}
```

服务端路由注册：
- `GET /locales.json` [i18n.ts#L155-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/hooks/i18n.ts#L155-L159) — 返回语言索引
- `GET /locales/:locale` [i18n.ts#L143-L153](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/hooks/i18n.ts#L143-L153) — 返回具体语言的完整翻译 JSON，带 `Cache-Control` 长缓存

### 4.2 Loader 加载流程

核心代码在 [html10n.ts#L845-L1015](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/vendors/html10n.ts#L845-L1015)

```
DOMContentLoaded
     │
     ▼
html10n.index() ─── 扫描所有 <link type="application/l10n+json">
     │                     │
     │                     ▼
     │               new Loader(resources)
     │                     │
     ▼                     │
html10n.localize(langs)    │
     │                     │
     ▼                     ▼
html10n.build() ───► asyncForEach(langs, Loader.load)
                           │
                           ▼
                    Loader.fetch(href)  ───  XHR GET locales.json
                           │
                           ▼
                    Loader.parse(lang, data)
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
     值是字符串？    值是对象？       找不到该语言？
          │                │                │
          ▼                ▼                ▼
  当作路径递归 fetch   langs.set()   按 BCP 47 层级逐层剥离
  （Import 规则）     存入缓存     - → zh-Hans → zh → 查找变体
```

`build()` 方法会把优先级数组中所有语言的翻译按顺序叠加合并：**低优先级语言先应用，高优先级后应用**，所以缺失的 key 会由更低优先级的语言补齐（典型：英文兜底）。

### 4.3 语言代码规范化与回退链

Loader.parse 中有两层语言代码映射表 [html10n.ts#L912-L940](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/vendors/html10n.ts#L912-L940)：

**浏览器语言 → BCP 47 标准**（`getBcp47LangCode`）：
```
zh-cn  → zh-hans-cn
zh-hk  → zh-hant-hk
zh-mo  → zh-hant-mo
zh-my  → zh-hans-my
zh-sg  → zh-hans-sg
zh-tw  → zh-hant-tw
```

**BCP 47 → JSON 文件名**（`getJsonLangCode`，translatewiki 约定全部小写）：
```
sr-ec   → sr-cyrl
sr-el   → sr-latn
zh-hk   → zh-hant-hk
```

**多级回退链**（当精确语言找不到时）[html10n.ts#L957-L985](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/vendors/html10n.ts#L957-L985)：

以 `zh-Hans-CN` 为例：
1. 尝试 `zh-hans-cn`（原始值）
2. 循环剥离最后一个 `-`：`zh-hans` → `zh`
3. 若仍找不到，遍历所有可用语言，查找是否有 `zh-` 开头的变体（如 `zh-hans`、`zh-hant`），取第一个匹配

### 4.4 Import 规则（链式加载）

当 locales.json 中某个语言的值是**字符串**而非对象时，会被当作路径继续 fetch：

```json
{
  "zh-hans": "locales/zh-hans.json"
}
```

- 绝对路径（`http://` 或 `/` 开头）：直接使用
- 相对路径：相对于当前资源文件的 URL 解析（`href + "/../" + data[lang]`）

这就是为什么英文必须内联——避免额外的一次 HTTP 请求，同时保证英文兜底总能加载成功。

---

## 5. 阶段四：翻译文本的页面应用

### 5.1 DOM 标记方式（data-l10n-id）

最基础用法：
```html
<!-- 替换元素的 textContent -->
<button data-l10n-id="pad.toolbar.bold.title">Bold</button>

<!-- 带参数 -->
<span data-l10n-id="pad.userlist.entername"
      data-l10n-args='{"name":"Alice"}'>Welcome, Alice</span>
```

`translateElement()` 会递归遍历所有带 `data-l10n-id` 的子孙元素（包括自身），调用 `translateNode()` 逐个处理。

### 5.2 翻译键与属性后缀约定

当翻译键的最后一段是以下白名单属性之一时，会写入对应的 DOM 属性而非 textContent [html10n.ts#L644-L659](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/vendors/html10n.ts#L644-L659)：

```
白名单属性：title | innerHTML | alt | textContent | value | placeholder
```

示例：
```html
<!-- placeholder 属性 -->
<input data-l10n-id="pad.impexp.importfrom.placeholder" type="file" />
<!-- 等价于 input.placeholder = t('pad.impexp.importfrom.placeholder') -->

<!-- title 属性（鼠标悬停提示） -->
<a data-l10n-id="pad.editbar.timeslider.title" href="#">History</a>
<!-- 等价于 a.title = t('pad.editbar.timeslider.title') -->

<!-- alt 属性 -->
<img data-l10n-id="pad.img.logo.alt" src="logo.png" />
```

不指定后缀时，默认使用 `textContent`（现代浏览器）或 `innerText`（旧 IE）。

### 5.3 宏系统与复数规则（plural）

`substMacros()` 解析形如 `{[ macroName(paramName) key: value, ... ]}` 的语法 [html10n.ts#L729-L764](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/vendors/html10n.ts#L729-L764)。

目前仅内置了 `plural` 一个宏：

翻译 JSON 中写法：
```json
{
  "pad.chat.nUsers": "{[ plural(n) zero: No users, one: {{n}} user, other: {{n}} users ]}"
}
```

调用方式：
```javascript
html10n.get('pad.chat.nUsers', {n: 5});  // "5 users"
html10n.get('pad.chat.nUsers', {n: 1});  // "1 user"
html10n.get('pad.chat.nUsers', {n: 0});  // "No users"
```

复数规则函数通过语言代码查表获取 [html10n.ts#L72-L468](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/vendors/html10n.ts#L72-L468)，覆盖了 200+ 种语言的 CLDR 复数规则（英文是规则 3：`n==1` 用 one，其余 other）。

### 5.4 参数插值（{{variable}}）

`substArguments()` 按正则 `/\{\{\s*([a-zA-Z\.]+)\s*\}\}/` 扫描并替换：

翻译 JSON：
```json
{
  "pad.userlist.welcome": "Welcome, {{userName}}! You are editing {{padName}}."
}
```

调用：
```javascript
html10n.get('pad.userlist.welcome', {userName: 'Alice', padName: 'MyPad'});
```

查找参数值的优先级：
1. `args` 对象中传入的值
2. `this.translations` 翻译字典中同名的 key
3. 都找不到 → 打印 warning 并保留原占位符

### 5.5 无障碍支持（aria-label 自动填充）

[html10n.ts#L669-L682](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/vendors/html10n.ts#L669-L682) 实现了重要的 a11y 逻辑：

每次翻译元素时，如果：
- 元素**没有**自带的 `aria-label` 属性，或
- 该 `aria-label` 是上次翻译自动生成的（带 `data-l10n-aria-label="true"` 标记）

则将翻译后的文本同步写入 `aria-label`，保证屏幕阅读器能读出正确的本地化文本。切换语言时会被覆盖更新。

对于 `<select>`、`<input>`、`<textarea>` 这类表单控件——它们的可访问名称不来自 textContent 而是 aria-label——代码会走专用分支 [html10n.ts#L683-L692](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/vendors/html10n.ts#L683-L692)，只写 aria-label，避免误报 "找不到文本节点" 的 warning。

---

## 6. Admin SPA 的独立 i18n 体系（i18next）

Admin 面板（`/admin`）是 React + Vite 构建的独立 SPA，不走 html10n 路线，使用成熟的 i18next 生态。

### 6.1 初始化与语言检测

[admin/src/localization/i18n.ts#L55-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/admin/src/localization/i18n.ts#L55-L64)：

```typescript
i18n
  .use(LanguageDetector)   // i18next-browser-languagedetector
  .use(LazyImportPlugin)   // 自定义后端：Vite 动态 import
  .use(initReactI18next)   // React 绑定
  .init({
    ns: ['translation', 'ep_admin_pads', 'ep_admin_authors'], // 命名空间
    fallbackLng: 'en',     // 英文兜底
  });
```

`LanguageDetector` 内置的检测顺序：`querystring (?lng=) → cookie → localStorage → navigator → htmlTag lang → path → subdomain`。

### 6.2 懒加载后端（LazyImportPlugin）

[admin/src/localization/i18n.ts#L17-L53](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/admin/src/localization/i18n.ts#L17-L53) 自定义了一个 `BackendModule`：

```typescript
// 核心语言包：Vite import.meta.glob 静态扫描 + 代码分割
const coreLocales = import.meta.glob('../../../src/locales/*.json');
// => 每个语言打包成独立的 hashed chunk，按需懒加载

read: async (language, namespace, callback) => {
  if (namespace === 'translation') {
    // 核心翻译：动态 import
    const loader = coreLocales[`../../../src/locales/${language}.json`];
    const mod = await loader();
    callback(null, mod.default);
  } else {
    // 插件命名空间：运行时 fetch admin/public/<ns>/<lang>.json
    const res = await fetch(`${BASE_URL}/${namespace}/${language}.json`);
    callback(null, await res.json());
  }
}
```

好处：无需构建时复制文件、无需额外的 Express 路由、每个语言按需分块加载。

### 6.3 React 组件中的使用

遵循 AGENTS.md 中的强制约定：

```tsx
// JSX 文本：用 <Trans />（保留插值的 React 节点结构）
import { Trans } from 'react-i18next';
<h2><Trans i18nKey="admin_plugins.subtitle" /></h2>

// 属性值 / 编程式：用 t()
import { useTranslation } from 'react-i18next';
const { t } = useTranslation();
<button aria-label={t('admin.common.refresh')} title={t('admin.common.refresh')}>
  <RefreshCwIcon />
</button>

// 复数：用 _one / _other 后缀约定
t('admin_pads.count_pads', {count: pads.length});
// admin_pads.count_pads_one = "{{count}} pad"
// admin_pads.count_pads_other = "{{count}} pads"
```

---

## 7. 关键文件索引

| 功能 | 文件 | 核心函数/导出 |
|------|------|--------------|
| **后端语言构建** | [src/node/hooks/i18n.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/hooks/i18n.ts) | `getAllLocales()`、`generateLocaleIndex()`、`expressPreSession` 路由注册 |
| **后端语言协商（Social Meta）** | [src/node/utils/socialMeta.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/utils/socialMeta.ts) | `negotiateRenderLang()`、`resolveDescription()` |
| **后端路由渲染** | [src/node/hooks/express/specialpages.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/node/hooks/express/specialpages.ts) | `/p/:pad`、`/` 路由，`eejs.require()` 渲染模板 |
| **Pad 模板** | [src/templates/pad.html](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/templates/pad.html) | `<link rel="localizations">`、`renderLang`/`renderDir` |
| **首页模板** | [src/templates/index.html](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/templates/index.html) | 同上 |
| **时间轴模板** | [src/templates/timeslider.html](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/templates/timeslider.html) | 同上 |
| **Pad 启动入口** | [src/templates/padBootstrap.js](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/templates/padBootstrap.js) | `window.clientVars` 初始化，加载 `l10n.ts` |
| **客户端语言初始化** | [src/static/js/l10n.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/l10n.ts) | Cookie 读取 → `html10n.localize()` → 更新 `<html lang>` |
| **核心翻译库（Pad UI）** | [src/static/js/vendors/html10n.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/vendors/html10n.ts) | `Html10n` 类、`Loader` 类、复数规则、DOM 翻译 |
| **Pad 语言控制** | [src/static/js/pad.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/static/js/pad.ts) | `getParameters`、`applyLanguage()`、`getMyViewOverrides()` |
| **核心语言资源（英文基准）** | [src/locales/en.json](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/src/locales/en.json) | 所有翻译 key 的基准定义 |
| **Admin SPA i18n** | [admin/src/localization/i18n.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/76-etherpad-lite/admin/src/localization/i18n.ts) | i18next 初始化 + LazyImportPlugin |
