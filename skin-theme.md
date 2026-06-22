# Etherpad Skin 与 UI 主题装配机制解析

本文档系统地梳理 Etherpad 项目中 Skin（皮肤）与 UI 主题的配置、资源选择和界面渲染之间的完整连接链路。

---

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        主题配置层 (Server-Side)                      │
│  settings.json → Settings.ts (验证/默认值) → getPublicSettings()    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        资源选择层 (Server-Side)                      │
│  skinName → 锁定 skins/<name>/ 目录 → 选择对应 CSS/JS/图片/图标     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        模板渲染层 (Server-Side, EJS)                 │
│  pad.html / index.html / timeslider.html                             │
│  → 注入 skinVariants class 到 <html>                                  │
│  → 输出 <link rel="stylesheet"> (核心CSS + 皮肤CSS + 变体CSS)        │
│  → 输出 <meta name="theme-color"> (移动端地址栏配色)                  │
│  → 输出预加载 Dark Mode 脚本 (防白闪)                                 │
│  → 输出 <script src=pad.js> / padbootstrap-xxx.min.js                │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        客户端初始化 (Browser)                         │
│  padBootstrap.js → pad.ts init()                                     │
│  → 读取 clientVars.enableDarkMode / skinVariants                     │
│  → matchMedia('(prefers-color-scheme: dark)') 自动切换               │
│  → 加载 skin_variants.ts，把 classes 传播到所有 iframe                │
│  → 用户点击 Dark Mode Toggle → localStorage 持久化                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、主题配置层

### 2.1 配置文件入口

用户在 `settings.json`（或 `settings.json.template`）中定义三个核心配置项：

| 配置项 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `skinName` | `string \| null` | `null`（运行时回退为 `"colibris"`） | 皮肤目录名，必须位于 `src/static/skins/<skinName>/` |
| `skinVariants` | `string` | `"super-light-toolbar super-light-editor light-background"` | 空格分隔的变体类名列表 |
| `enableDarkMode` | `boolean` | `true` | 是否允许客户端自动暗黑模式 |

配置文件片段见 [settings.json.template#L147-L183](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/settings.json.template#L147-L183) 和 [settings.json.template#L837-L843](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/settings.json.template#L837-L843)。

### 2.2 环境变量替换

配置文件支持 `${ENV_VAR}` / `${ENV_VAR:default}` 语法，例如：

```json
"skinName": "${SKIN_NAME:colibris}",
"enableDarkMode": "${ENABLE_DARK_MODE:true}"
```

解析逻辑位于 [Settings.ts#L93-L168](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/Settings.ts#L93-L168) 的 `parseSettings()` → `lookupEnvironmentVariables()` → `coerceValue()` 函数链中，会自动把 `"true"/"false"` 转成布尔、数字字符串转成 number。

### 2.3 默认值与运行时验证

所有默认值定义在 [Settings.ts#L375-L883](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/Settings.ts#L375-L883) 的 `settings` 对象中：

```typescript
skinName: null,  // 故意设为 null，用于检测老旧配置文件
skinVariants: 'super-light-toolbar super-light-editor light-background',
enableDarkMode: true,
```

`reloadSettings()` 函数（[Settings.ts#L1172-L1298](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/Settings.ts#L1172-L1298)）在启动时执行四层安全校验：

1. **缺失 skinName 警告**：若 `skinName == null`，输出 WARN 日志并回退到 `"colibris"`
2. **路径穿越防护**：`skinName` 必须是单层目录名（不含 `path.sep`），且必须是 `skins/` 的子目录（防止 `..` 攻击）
3. **目录存在性**：`src/static/skins/<skinName>/` 必须真实存在于磁盘
4. **日志确认**：最终打印 `Using skin "<name>" in dir: <path>`

### 2.4 公开给客户端的配置子集

`settings.getPublicSettings()`（[Settings.ts#L865-L881](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/Settings.ts#L865-L881)）只把安全的字段暴露给前端：

```typescript
{
  title, skinVariants, randomVersionString, skinName, toolbar,
  exposeVersion, gitVersion, enableDarkMode,
  enablePadWideSettings, enablePluginPadOptions, privacyBanner
}
```

这些字段在渲染 `pad.html` 时作为 `settings` 上下文传入模板，并通过 `entrypoint` 脚本中的 `window.clientVars` 进一步传递。

---

## 三、资源选择层

### 3.1 Skin 目录规范

每个 Skin 是 `src/static/skins/<skinName>/` 下的一组约定文件：

```
src/static/skins/colibris/
├── index.js              # 首页 (/) 运行
├── index.css             # 首页样式
├── pad.js                # 文档页 (/p/:pad) 运行
├── pad.css               # 文档页样式（入口，@import 子文件）
├── timeslider.js         # 时间轴 iframe 运行
├── timeslider.css        # 时间轴样式
├── favicon.ico           # 可选：覆盖默认图标
├── robots.txt            # 可选：覆盖默认 robots
├── images/               # 可选：图片资源
└── src/                  # colibris 特有：CSS 模块化子目录
    ├── general.css
    ├── layout.css
    ├── pad-editor.css
    ├── pad-variants.css  # ★ 皮肤变体规则核心
    ├── components/*.css
    └── plugins/*.css
```

规范文档在 [skins.md](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/doc/skins.md)。

### 3.2 CSS 资源加载顺序（以 pad 页为例）

在 [pad.html#L89-L97](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L89-L97) 中由 EJS 模板输出：

```html
<!-- 1. 核心基础样式（所有 skin 共享） -->
<link href="../static/css/pad.css?v=2123" rel="stylesheet">

<!-- 2. Skin 专属样式（customStyles eejs block，插件可通过钩子注入） -->
<link href="../static/skins/colibris/pad.css?v=2123" rel="stylesheet">

<!-- 3. 动态语法高亮槽位（运行时 JS 填充） -->
<style title="dynamicsyntax"></style>
```

**关键机制**：所有静态资源路径都附加 `?v=randomVersionString` 查询参数。`randomVersionString` 由 Etherpad 版本号 + git SHA 确定性生成（[Settings.ts#L1404-L1420](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/Settings.ts#L1404-L1420)），保证升级后浏览器强制刷新缓存，同时多副本部署之间 hash 一致。

### 3.3 colibris pad.css 的模块化导入

[colibris/pad.css](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/skins/colibris/pad.css) 不是单文件，而是通过 `@import` 顺序串联 20+ 子 CSS：

1. 基础层：`general.css` → `layout.css` → `pad-editor.css`
2. 组件层：`scrollbars.css`、`buttons.css`、`popup.css`、`chat.css`、`toolbar.css` 等
3. 插件适配层：`plugins/brightcolorpicker.css`、`plugins/comments.css` 等
4. **变体层**：`pad-variants.css`（★ 最关键，见 §4）

这种结构让变体规则最后加载，优先级最高，正确覆盖基础颜色变量。

### 3.4 Favicon 与 robots.txt 的 Skin 覆盖链

在 [specialpages.ts#L62-L107](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/hooks/express/specialpages.ts#L62-L107) 中实现了降级优先级：

**favicon.ico 查找顺序**：
1. `settings.favicon`（绝对路径，也可以是 http URL → 直接 302 重定向）
2. `src/static/skins/<skinName>/favicon.ico`
3. `src/static/favicon.ico`（兜底）

**robots.txt 查找顺序**：
1. 若 `skinName` 未设置 → 直接发默认 `src/static/robots.txt`
2. 先尝试 `src/static/skins/<skinName>/robots.txt`，不存在则 fallback 到默认

### 3.5 静态资源服务管道

所有 `/static/*` 请求通过 [static.ts#L34-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/hooks/express/static.ts#L34-L38) 的 `minify` 中间件：

```typescript
app.all('/static/*filename', minify);
```

`minify` 中间件负责：按需压缩 JS/CSS、为 ace/require-kernel 等特殊模块做内容改写、设置 `Cache-Control: max-age` 缓存头。

---

## 四、皮肤变体（Skin Variants）工作原理

### 4.1 变体 token 的语法

`skinVariants` 是一个**空格分隔**的 token 列表，每个 token 遵循 `<亮度>-<容器>` 格式：

| 维度 | 可选值 |
|---|---|
| **亮度** | `super-light` / `light` / `dark` / `super-dark` |
| **容器** | `toolbar`（工具栏）/ `background`（页面背景）/ `editor`（编辑区） |
| **特殊** | `full-width-editor`（编辑器占满宽度，无左右边距） |

示例（默认值）：`"super-light-toolbar super-light-editor light-background"`

### 4.2 变体规则的定义位置

所有 CSS 变体规则集中在 [colibris/src/pad-variants.css](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/skins/colibris/src/pad-variants.css)，通过 CSS 变量（Custom Properties）机制实现"换皮"：

```css
/* 4 种 Toolbar × 2 种特殊组合 */
.super-light-toolbar .toolbar { --bg-color: var(--super-light-color); --text-color: var(--super-dark-color); }
.light-toolbar       .toolbar { --bg-color: var(--light-color);        --text-color: var(--super-dark-color); }
.dark-toolbar        .toolbar { --bg-color: var(--dark-color);         --text-color: var(--super-light-color); }
.super-dark-toolbar  .toolbar { --bg-color: var(--super-dark-color);   --text-color: var(--super-light-color); }

/* 4 种 Background */
.super-light-background #editorcontainerbox { --bg-color: ...; }
...

/* 4 种 Editor */
.super-light-editor #outerdocbody iframe { --bg-color: ...; }
...

/* Full Width */
.full-width-editor #outerdocbody iframe { max-width: none; border-radius: 0; }
...

/* 根画布配色（修复 iOS Safari 安全区白条 #7606） */
html.super-light-toolbar { background-color: #fff;  color-scheme: light; }
html.super-dark-toolbar  { background-color: #485365; color-scheme: dark; }
```

要点：**不在每个类里硬编码颜色值**，而是通过重新设置 CSS 变量改变全局配色。`pad.css` 顶部的 `:root` 块（[colibris/pad.css#L33-L56](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/skins/colibris/pad.css#L33-L56)）定义了 4 个基础色变量，变体规则再把这些变量重映射到 `--bg-color` / `--text-color` 等"语义变量"，后续所有组件样式都只引用语义变量。

### 4.3 类名注入：服务端 → `<html>` 根元素

在 [pad.html#L22](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L22) EJS 模板直接把 `skinVariants` 字符串写进 `<html class>`：

```html
<html lang="en" dir="ltr" translate="no" class="pad <%=pluginUtils.clientPluginNames().join(' '); %> <%=settings.skinVariants%>">
```

这样浏览器在解析 CSS 之前类名已经就位，首次渲染不会出现颜色闪烁（Dark Mode 另有处理，见 §五）。

### 4.4 theme-color：与移动端地址栏同步

为了让 iOS Safari / Android Chrome 的地址栏颜色和 Toolbar 一致（用户视觉一体感），模板在 `<head>` 输出 `<meta name="theme-color">`。

颜色解析逻辑在两处共享同一份真值表：

1. **真值表**：[skin_toolbar_colors.ts#L13-L18](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_toolbar_colors.ts#L13-L18)

   ```typescript
   export const TOOLBAR_COLORS_IN_CSS_ORDER = [
     ['super-light-toolbar', '#ffffff'],
     ['light-toolbar',       '#f2f3f4'],
     ['super-dark-toolbar',  '#485365'],
     ['dark-toolbar',        '#576273'],
   ];
   ```

   顺序很重要：`toolbarColorForTokens()` 按顺序遍历，最后一个命中的 token 获胜，与 CSS 级联规则完全相同。

2. **服务端**：[SkinColors.ts#L10-L31](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/SkinColors.ts#L10-L31)
   - `configuredToolbarColor()`：根据当前 `skinVariants` 计算亮配色
   - `darkToolbarColor()`：强制 `super-dark-toolbar` 计算深配色（用于 media 查询）

3. **模板输出**：[pad.html#L18-L19, #L57-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L18-L19)

   ```html
   <!-- 亮配色（或无 dark mode 时的唯一样式） -->
   <meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
   <!-- 深配色（仅当 enableDarkMode 为 true 时输出） -->
   <meta name="theme-color" content="#485365" media="(prefers-color-scheme: dark)">
   ```

   之所以用双 `<meta>` + `media` 属性，是因为 **iOS Safari 在 HTML 解析时就会锁定地址栏颜色**，后续 JS 修改 `<meta>` 不可靠（见 #7606 issue）。

---

## 五、界面渲染的完整流水线

### 5.1 服务端启动时：构建 JS Bundle

在 [specialpages.ts#L301-L444](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/hooks/express/specialpages.ts#L301-L444) 的 `expressCreateServer` 钩子中：

1. 用 eejs 渲染三个 bootstrap 模板为字符串：
   - `padBootstrap.js` → pad 页入口（含所有插件模块 require）
   - `indexBootstrap.js` → 首页入口
   - `timeSliderBootstrap.js` → 时间轴 iframe 入口
2. 调用 esbuild 的 `buildSync()` 打包为单文件（目标 ES2020，生产模式压缩 + 去除 sourcemap）
3. 根据 esbuild 输出的 `hash` 生成文件名 `padbootstrap-<hash>.min.js`
4. 注册 Express 路由：`/padbootstrap-<hash>.min.js` → 返回打包好的 JS 内容
5. 把文件名作为 `entrypoint` 变量传给 HTML 模板

**开发模式**（`NODE_ENV !== "production"`）：改用 `handleLiveReload()`，通过 chokidar 监听 `src/static/js/` 下的文件变更，变更时自动重新 esbuild 并通过 socket.io 广播 `liveupdate` 消息通知浏览器刷新。

### 5.2 请求到达时：EJS 渲染 HTML

以 `/p/:pad` 为例（[specialpages.ts#L381-L406](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/hooks/express/specialpages.ts#L381-L406)）：

```
1. 检查权限 → isReadOnly
2. hooks.callAll('padInitToolbar') → 允许插件修改工具栏
3. 渲染 socialMeta（og:title, twitter:card 等 Open Graph 标签）
4. eejs.require('templates/pad.html', {
     req, toolbar, isReadOnly,
     entrypoint: "../padbootstrap-AbC.min.js",
     settings: settings.getPublicSettings(),  // ← 包含 skinName, skinVariants, enableDarkMode
     socialMetaHtml, proxyPath
   })
```

### 5.3 pad.html 中与主题相关的输出顺序

```
<head>
  ① <meta name="theme-color"> (亮+暗 双标签)
  ② 【内联脚本】Dark Mode 预加载（仅 enableDarkMode=true 时输出）
     → 在 <link rel="stylesheet"> 之前运行
     → 检查 localStorage.ep_darkMode === 'false' 或 matchMedia(dark)
     → 如果命中，先把 <html> 加上 super-dark-editor/dark-background/super-dark-toolbar
     → 这样 CSS 加载完成立刻应用暗色，不闪白
  ③ <link> 基础 pad.css
  ④ <link> 皮肤 pad.css (变体规则在里面)
</head>
<body>
  ...
  <script src="entrypoint"></script>        ← padBootstrap JS
  <script src="skins/colibris/pad.js"></script> ← 皮肤自定义 JS (customStart 钩子)
</body>
```

**Dark Mode 预加载脚本**的内联版本位于 [pad.html#L62-L87](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L62-L87)。

### 5.4 客户端初始化：Dark Mode 自动切换

padBootstrap 加载后进入 `pad.ts init()`，在 [pad.ts#L761-L770](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L761-L770) 执行：

```typescript
// 条件：不是 #skinvariantsbuilder 调试模式 && enableDarkMode 开启
//       && 系统偏好暗色 && 用户没显式选 "强制亮色"
if (hash !== '#skinvariantsbuilder'
    && clientVars.enableDarkMode
    && matchMedia('(prefers-color-scheme: dark)').matches
    && !skinVariants.isWhiteModeEnabledInLocalStorage()) {
  skinVariants.updateSkinVariantsClasses([
    'super-dark-editor', 'dark-background', 'super-dark-toolbar'
  ]);
}

// 显示设置面板里的 Dark Mode 开关
if (clientVars.enableDarkMode) {
  $('#theme-toggle-row').prop('hidden', false);
  $('#options-darkmode').prop('checked', skinVariants.isDarkMode());
}
```

### 5.5 运行时切换：skin_variants.ts

[skin_variants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts) 导出 5 个 API：

| API | 作用 |
|---|---|
| `updateSkinVariantsClasses(newClasses)` | 把新的 variant classes 应用到**所有相关 DOM 根** |
| `isDarkMode()` | 当前是否处于 dark（检查 `html.super-dark-editor`） |
| `setDarkModeInLocalStorage(bool)` | 写入 `localStorage.ep_darkMode` |
| `isDarkModeEnabledInLocalStorage()` | 读 `ep_darkMode === 'true'` |
| `isWhiteModeEnabledInLocalStorage()` | 读 `ep_darkMode === 'false'`（用户显式选了亮） |

**传播 iframe 的关键点**（[skin_variants.ts#L28-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L28-L59)）：

```
domsToUpdate = [
  $('html'),                                          // 主文档
  ace_outer iframe → contents → html,                  // 编辑器外层
  ace_outer → ace_inner iframe → contents → html,      // 编辑器正文（真正的内容区）
  #history-frame iframe → html,                        // 时间轴模式下的历史副本
  #history-frame → ace_outer → html,
  #history-frame → ace_outer → ace_inner → html,
]
```

每个操作都是：先 remove 所有 4×3=12 个可能的旧 variant class，再 add 新 class，最后同步更新 `<meta name="theme-color">` 的 content 值（这样手动切换时 Android Chrome 地址栏颜色立即变化）。

### 5.6 皮肤自定义 JS 钩子：window.customStart()

每个 Skin 的 `pad.js` / `index.js` / `timeslider.js` 都应该定义 `window.customStart = () => { ... }`。由 padBootstrap 和框架保证在 DOM Ready + 插件 Ready 后被调用。

典型用途：
- [colibris/pad.js](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/skins/colibris/pad.js)：工具栏按钮按下态视觉反馈、最近 pad 列表 localStorage 维护
- [colibris/index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/skins/colibris/index.js)：首页最近 pad 列表渲染、输入框 placeholder 同步

### 5.7 Skin Variants Builder（开发者工具）

访问任意 `/p/test#skinvariantsbuilder`，会弹出内置的"变体构造器"弹窗（代码在 [pad.html#L658-L687](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L658-L687) + [skin_variants.ts#L79-L114](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L79-L114)）：

```
三个下拉框：toolbar / background / editor  ×  super-light/light/dark/super-dark
一个复选框：Full Width Editor
一个只读 input：实时输出可直接粘贴到 settings.json 的 skinVariants 字符串
```

用户操作的每个 change 事件都会实时调用 `updateSkinVariantsClasses()`，所见即所得。

---

## 六、三条页面的差异对照表

| | 首页 `/` | Pad 页 `/p/:pad` | Timeslider `/p/:pad/timeslider?embed=1` |
|---|---|---|---|
| **模板** | [index.html](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/index.html) | [pad.html](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html) | [timeslider.html](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/timeslider.html) |
| **入口 JS** | `indexBootstrap.js` | `padBootstrap.js` | `timeSliderBootstrap.js` |
| **<html class>** | （无 variant class） | `pad <skinVariants>` | `pad <skinVariants>` |
| **CSS 加载** | `static/skins/<n>/index.css` | `static/css/pad.css` + `static/skins/<n>/pad.css` | `+ static/css/timeslider.css` + `+ static/skins/<n>/timeslider.css` |
| **皮肤 JS** | `skins/<n>/index.js` | `skins/<n>/pad.js` | `skins/<n>/timeslider.js` |
| **Dark 预加载** | 无 | 有（内联 IIFE） | 有（与 pad.html 相同逻辑） |
| **theme-color** | 无 | 亮 + 暗双 meta | 亮 + 暗双 meta |

---

## 七、关键代码文件索引

| 职责 | 文件路径 |
|---|---|
| **配置定义与验证** | [Settings.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/Settings.ts) |
| **配置模板** | [settings.json.template](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/settings.json.template) |
| **路由 + Bundle + 模板调用** | [specialpages.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/hooks/express/specialpages.ts) |
| **静态资源 minify 中间件** | [static.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/hooks/express/static.ts) |
| **Toolbar 颜色解析（共享真值）** | [skin_toolbar_colors.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_toolbar_colors.ts) |
| **Server 端颜色包装器** | [SkinColors.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/SkinColors.ts) |
| **Pad 页面模板（核心）** | [pad.html](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html) |
| **Pad 入口脚本模板** | [padBootstrap.js](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/padBootstrap.js) |
| **客户端 Dark Mode + 变体切换** | [skin_variants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts) |
| **Pad 初始化（Dark Mode 自动开关）** | [pad.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts) |
| **Colibris 皮肤 CSS 入口** | [colibris/pad.css](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/skins/colibris/pad.css) |
| **Colibris 变体 CSS 规则** | [colibris/src/pad-variants.css](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/skins/colibris/src/pad-variants.css) |
| **Colibris 自定义 JS** | [colibris/pad.js](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/skins/colibris/pad.js) · [index.js](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/skins/colibris/index.js) |
| **官方 Skins 文档** | [skins.md](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/doc/skins.md) |

---

## 八、典型数据流动示例

**场景**：用户在 `settings.json` 设置 `"enableDarkMode": true`，用 macOS 深色模式打开 `/p/test`。

```
(1) 启动：reloadSettings()
    └─ settings.skinName = "colibris"
       settings.skinVariants = "super-light-toolbar super-light-editor light-background"
       settings.enableDarkMode = true

(2) esbuild 构建 padBootstrap-xxx.min.js

(3) GET /p/test → Express 路由命中 specialpages.ts /p/:pad
    └─ eejs.render(pad.html, { settings: getPublicSettings() })
       │
       ├─ <html class="pad super-light-toolbar super-light-editor light-background">
       ├─ <meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
       ├─ <meta name="theme-color" content="#485365" media="(prefers-color-scheme: dark)">
       │
       ├─ ★ 内联 Dark Mode 预加载脚本运行：
       │     matchMedia(dark).matches → true
       │     localStorage.ep_darkMode → null (未显式选)
       │     → <html>.classList.add('super-dark-editor','dark-background','super-dark-toolbar')
       │
       ├─ <link> pad.css + colibris/pad.css （此时 <html> 已是 dark classes → 首帧不闪白）
       │
       └─ <script src=padbootstrap-xxx.min.js>
          └─ <script src=skins/colibris/pad.js>

(4) pad.ts init() 运行
    └─ 再次确认 enableDarkMode + matchMedia + !localStorage white-mode
       → 调用 skinVariants.updateSkinVariantsClasses(dark classes)
         → 把 classes 同步到 ace_outer / ace_inner / 潜在的 #history-frame
         → document.querySelectorAll('meta[name=theme-color]').setAttribute(content, '#485365')
       → 显示 #options-darkmode checkbox，checked = true

(5) 用户点击 Dark Mode 开关（取消勾选）
    └─ setDarkModeInLocalStorage(false)
       updateSkinVariantsClasses(light classes)
       下次访问：预加载脚本看见 localStorage.ep_darkMode === 'false' → 不自动切暗
```

---

## 九、自定义皮肤（非 colibris）下的实际行为全链路分析

**核心矛盾**：整个主题装配系统的设计以 colibris 为"一等公民"实现，大量分支里隐含 `skinName == 'colibris'` 的硬编码判断。当你把 `settings.json` 里 `skinName` 改成 `"no-skin"` 或任何自定义名字时，**四块主题功能出现了不同的降级/保留行为，且彼此不一致**。

### 9.1 四块功能的降级/保留速查表

| 功能块 | colibris 皮肤 | 非 colibris 皮肤 | 行为类型 | 根源判断 |
|---|---|---|---|---|
| **theme-color meta** | ✅ 亮+暗双标签 | ❌ 完全不输出任何 meta | **降级（完全消失）** | [SkinColors.ts#L14](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/SkinColors.ts#L14) + [SkinColors.ts#L29](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/SkinColors.ts#L29) |
| **favicon.ico** | ✅ skins/colibris/favicon.ico → 默认 | ✅ skins/<name>/favicon.ico → 默认 | **完全保留** | [specialpages.ts#L90-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/hooks/express/specialpages.ts#L90-L104) 不检查 skinName |
| **robots.txt** | ✅ skins/colibris/robots.txt → 默认 | ✅ skins/<name>/robots.txt → 默认 | **完全保留** | [specialpages.ts#L62-L75](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/hooks/express/specialpages.ts#L62-L75) 不检查 skinName |
| **Dark Mode 自动开关（服务端预加载脚本）** | ✅ 内联 IIFE 防白闪 | ❌ 不输出任何脚本 | **降级（无防白闪）** | [pad.html#L19](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L19) + [pad.html#L61](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L61) 通过 `darkColor == null` 间接实现 |
| **Dark Mode 自动开关（pad.ts 客户端自动切）** | ✅ 自动切换 + 生效 | ⚠️ 逻辑仍执行但视觉无效果 | **保留逻辑但失效** | [pad.ts#L764](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L764) 不检查 skinName |
| **Dark Mode Toggle（设置面板 checkbox）** | ✅ 显示 + 生效 | ⚠️ 显示但视觉无效果 | **保留逻辑但失效** | [pad.ts#L767](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L767) 不检查 skinName |
| **Skin Variants Builder 弹窗（#skinvariantsbuilder）** | ✅ 渲染 DOM | ❌ 不渲染 DOM | **降级（完全消失）** | [pad.html#L658](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L658) 显式 `if (settings.skinName == 'colibris')` |
| **<html> 的 skinVariants classes 注入** | ✅ 写入 + 生效 | ✅ 写入但可能无效 | **保留写入** | [pad.html#L22](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L22) 通用，不检查 skinName |
| **skins/<name>/pad.css + pad.js 加载** | ✅ colibris 路径 | ✅ <name> 路径 | **完全保留** | [pad.html#L93](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L93) 通用 `encodeURI(settings.skinName)` 拼接 |

### 9.2 theme-color：完全降级的完整调用链

**非 colibris 时的追踪**：

```
GET /p/test
  │
  ├─ specialpages.ts 渲染 pad.html，传入 settings.skinName = "no-skin"
  │
  ├─ pad.html#L18:
  │    configuredColor = skinColors.configuredToolbarColor("no-skin", "super-light-toolbar ...")
  │
  ├─ SkinColors.ts#L10-L16 configuredToolbarColor():
  │    if (skinName !== 'colibris') return null;   ←★ 命中，返回 null
  │    configuredColor → null
  │
  ├─ pad.html#L19:
  │    darkColor = settings.enableDarkMode ? skinColors.darkToolbarColor("no-skin") : null
  │                (假设 enableDarkMode = true)
  │
  ├─ SkinColors.ts#L26-L31 darkToolbarColor():
  │    if (skinName !== 'colibris') return null;   ←★ 命中，返回 null
  │    darkColor → null
  │
  ├─ pad.html#L57:  <% if (configuredColor) { %>...<% } %>
  │                 configuredColor = null → 跳过 → 无 theme-color meta
  │
  └─ pad.html#L58:  <% if (darkColor) { %>...<% } %>
                    darkColor = null → 跳过 → 无 dark theme-color meta
```

**最终 HTML 头部**：完全没有任何 `<meta name="theme-color">` 标签。移动端浏览器（iOS Safari、Android Chrome）会使用各自的默认背景色（通常是白色），可能和自定义皮肤的实际 toolbar 背景色反差很大。

**timeslider.html 完全相同**（[timeslider.html#L7-L10](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/timeslider.html#L7-L10) + [#L45-L46](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/timeslider.html#L45-L46)）：同一个 `SkinColors` 包装器 + 同一套条件渲染。

**设计意图**：代码注释（[SkinColors.ts#L5-L9](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/SkinColors.ts#L5-L9)）明确说了——只有 colibris 的 toolbar 颜色映射表是已知的（见 `skin_toolbar_colors.ts` 的 `TOOLBAR_COLORS_IN_CSS_ORDER`）。对于自定义皮肤，框架**猜不出 toolbar 颜色**，所以与其输出一个误导值（比如错误的白色，而自定义皮肤 toolbar 可能是紫色），不如直接不输出 meta。

### 9.3 favicon & robots：无条件保留

这两块的路由代码**完全不区分 colibris / 非 colibris**，是真正的"皮肤无关"设计。

#### favicon 的三级 fallback 链（通用，任何皮肤适用）

代码 [specialpages.ts#L78-L107](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/hooks/express/specialpages.ts#L78-L107)：

```
GET /favicon.ico
  │
  ├─ ① 如果 settings.favicon 是 http(s) URL → 302 重定向
  │
  └─ 否则按顺序尝试磁盘文件，第一个可读的即返回：
       1. settings.favicon（如果有）→ resolve 到绝对路径
       2. src/static/skins/<settings.skinName>/favicon.ico
       3. src/static/favicon.ico（兜底）
```

关键点：第 2 步的 `path.join(settings.root, 'src', 'static', 'skins', settings.skinName, 'favicon.ico')` 没有任何 `skinName == 'colibris'` 判断。只要自定义皮肤目录下放了 `favicon.ico` 就能覆盖。

#### robots.txt 的两级 fallback 链（通用）

代码 [specialpages.ts#L62-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/hooks/express/specialpages.ts#L62-L76)：

```
GET /robots.txt
  │
  ├─ 特殊分支：if (!settings.skinName) → 直接发默认 src/static/robots.txt
  │     （这是遗留分支，Settings.ts 已保证 skinName 不可能为空）
  │
  └─ 正常：先尝试 skins/<skinName>/robots.txt
            ↓ 文件不存在时走 err 回调
            发默认 src/static/robots.txt
```

### 9.4 Dark Mode 自动开关：三层不同的命运

这是整个主题装配系统中**最不直观**的部分——三个相关组件在非 colibris 下有三种完全不同的行为。

#### 9.4.1 层一：服务端内联预加载脚本 → 完全不输出

**触发链**（以 pad.html 为例）：

```
pad.html#L18: configuredColor = configuredToolbarColor("no-skin", ...) → null
pad.html#L19: darkColor      = darkToolbarColor("no-skin")            → null
              ↓
pad.html#L61: <% if (darkColor) { %> ... 内联 IIFE ... <% } %>
              ↑ darkColor 为 null → 整个 <script> 块从 HTML 中彻底消失
```

**结果**：
- 没有防白闪脚本
- 非 colibris + enableDarkMode + 系统深色模式的用户，首帧会看到浏览器默认白底色（因为在 `<html>` 上改 class 的脚本根本不存在）
- iOS Safari 地址栏颜色也没法靠 meta 联动（见 §9.2）

timeslider.html 的逻辑完全镜像（[timeslider.html#L10](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/timeslider.html#L10) + [#L48-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/timeslider.html#L48-L68)）。

#### 9.4.2 层二：pad.ts init() 自动切暗 → 逻辑执行，但视觉上"空操作"

代码 [pad.ts#L764-L770](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L764-L770) 的判断条件：

```typescript
if (window.location.hash.toLowerCase() !== '#skinvariantsbuilder'
    && window.clientVars.enableDarkMode          // ← 只看 settings 开关
    && (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches)
    && !skinVariants.isWhiteModeEnabledInLocalStorage()) {
  skinVariants.updateSkinVariantsClasses([
    'super-dark-editor', 'dark-background', 'super-dark-toolbar'
  ]);   // ← 非 colibris 也照常调用
}
```

**重点**：这 4 个条件里**完全没有** `skinName !== 'colibris'` 的判断。`clientVars.skinName` 虽然通过 `getPublicSettings()` 暴露给了前端，但这里没用到。

进入 `updateSkinVariantsClasses()`（[skin_variants.ts#L28-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L28-L59)）后发生什么？

1. 收集 3~6 个 DOM 根（html、ace_outer、ace_inner、潜在 history-frame 三层）
2. 从每个 DOM 根 remove 所有 12 个可能的旧 variant class
3. 给每个 DOM 根 add 新的 `super-dark-editor dark-background super-dark-toolbar`
4. 调用 `updateThemeColorMeta()` → 由于 `meta[name="theme-color"]` 不存在（§9.2），`metas.length == 0` → 提前 return，什么也不做

**最终效果**：`<html>` 等节点的 classList 确实被改成了暗色值，但——

> **自定义皮肤的 pad.css 里有没有定义 `.super-dark-toolbar .toolbar { --bg-color: ... }` 这样的规则？**
>
> 如果没有（比如默认的 no-skin/pad.css 只有一行 `/* intentionally empty */`），这些 class 对视觉完全**零影响**。

#### 9.4.3 层三：设置面板的 Dark Mode Toggle → checkbox 可见，但切换后"空操作"

代码 [pad.ts#L767-L770](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L767-L770)：

```typescript
if (window.clientVars.enableDarkMode) {   // ← 同样不检查 skinName
  $('#theme-toggle-row').prop('hidden', false);
  $('#options-darkmode').prop('checked', skinVariants.isDarkMode());
}
```

- `#theme-toggle-row`（[pad.html#L303](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L303)）原本是 `hidden` 的，所以非 colibris + enableDarkMode 时，用户**看得见**这个 checkbox
- `#options-darkmode` 的 checked 状态由 `isDarkMode()`（检查 `html.super-dark-editor`）决定，而这个 class 在前一步自动切暗时已经被加上了，所以 checkbox 会显示为"已勾选"
- 用户点击 checkbox → 触发 [pad_editor.ts#L140-L150](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad_editor.ts#L140-L150) 的绑定 → 调用 `updateSkinVariantsClasses()` → 同 §9.4.2，**视觉无效果除非自定义皮肤自己写了 variant CSS**

### 9.5 Skin Variants Builder → 硬编码 colibris 判断，彻底不渲染

在 [pad.html#L658](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L658)：

```ejs
<% if (settings.skinName == 'colibris') { %>
<div id="skin-variants" class="popup" ...>
  <!-- 三个下拉框 + Full Width checkbox + 结果输入框 -->
</div>
<% } %>
```

**非 colibris 行为**：整个 `<div id="skin-variants">` 不出现在 DOM 中。即使用户手动访问 `/p/test#skinvariantsbuilder`，[skin_variants.ts#L80-L84](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L80-L84) 会尝试 `$('#skin-variants').addClass('popup-show')`，但因为 DOM 元素不存在，jQuery 静默失败，用户看不到任何弹窗。

### 9.6 对照基准：no-skin 皮肤到底是什么

官方提供的 `no-skin` 皮肤位于 `src/static/skins/no-skin/`，只有 6 个文件：

| 文件 | 内容 | 非 colibris 的典型行为参考 |
|---|---|---|
| `pad.css` | `/* intentionally empty */`（空文件） | variant class 完全无 CSS → Dark Mode 切换视觉零效果 |
| `pad.js` | `window.customStart = () => { /* 空 */ }` | 没有任何自定义行为 |
| `index.css` | 60+ 行，覆盖首页 form/wrapper/button 样式（纯布局调整，没有皮肤配色） | 首页仍然有样式，但不靠 variant class |
| `index.js` | `customStart` 里用 MutationObserver 同步 input placeholder | 功能性代码，和主题无关 |
| `timeslider.css` | 只有顶部注释 `/* custom css files are loaded after core css files... */` | 空样式 |
| `timeslider.js` | `window.customStart = () => { /* 空 */ }` | 无行为 |

**关键结论**：no-skin 并不意味着"完全没样式"——核心 `static/css/pad.css` 仍然会加载（[pad.html#L90](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L90)，这一步与 skinName 无关）。no-skin 真正"no"的是：
1. 没有 colibris 那种皮肤特有的布局/配色 CSS
2. 没有 `pad-variants.css` 这种变体规则
3. 没有皮肤 JS 功能（按钮反馈、最近列表等）

### 9.7 典型数据流对比：colibris vs no-skin

**假设条件**：`enableDarkMode=true`，系统 macOS 深色模式，访问 `/p/test`

```
              colibris 皮肤                              no-skin 皮肤
──────────────────────────────────────   ──────────────────────────────────────

(1) HTML 模板渲染:
    <meta theme-color> × 2 输出 ✅         <meta theme-color> 完全没有 ❌
    内联 Dark Mode IIFE 输出 ✅            内联 Dark Mode IIFE 不输出 ❌
    <html class="pad super-light-toolbar   <html class="pad super-light-toolbar
              super-light-editor light               super-light-editor light
              -background">                           -background">

(2) CSS 加载后首帧:
    IIFE 已把 class 改成 dark → 暗色 ✅    没有 IIFE，class 仍是 settings 里的
                                            亮配色 → 白屏/亮底首帧 ⚠️
                                            （除非自定义皮肤本来就是暗色）

(3) pad.ts init() 运行:
    enableDarkMode + matchMedia →          enableDarkMode + matchMedia →
    updateSkinVariantsClasses(dark)        updateSkinVariantsClasses(dark)
    .toolbar → --bg-color 重映射为暗色 ✅   <html classList 被改动了 ✅
                                            但 pad.css 没有 .*-toolbar
                                            规则 → 视觉零变化 ⚠️

(4) 设置面板:
    Dark Mode toggle 显示 + 生效 ✅         Dark Mode toggle 显示 ✅
    Skin Builder 弹窗访问 #xxx → 出现 ✅    Skin Builder 弹窗 → DOM 不存在 ❌
    切 toggle → 配色即时变化 ✅              切 toggle → classList 变
                                            但视觉零变化 ⚠️

(5) theme-color (Android Chrome):
    首帧 2 个 meta 中命中 dark 那一个 ✅    无 meta → 浏览器默认白 ❌
    用户手动切亮时 JS 改写 meta content ✅   无 meta，JS 什么都不做 ❌

(6) favicon / robots:
    标准 fallback 链 ✅                     标准 fallback 链 ✅（完全一致）
```

### 9.8 给自定义皮肤开发者的工程建议

如果你打算开发非 colibris 的皮肤，基于以上分析有 **4 个必须处理的要点**：

1. **补上变体 CSS**：要让 Dark Mode / skinVariants 真的生效，必须在你的 `pad.css`（或通过 `@import` 引入的子文件）里定义和 `colibris/src/pad-variants.css` 等价的 12+1 条规则（3 容器 × 4 亮度 + full-width-editor）。class 名必须完全一致，因为 `skin_variants.ts` 里的 token 是硬编码的（[skin_variants.ts#L6-L7](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L6-L7)）。

2. **扩展 SkinColors 或自写 meta**：`<meta theme-color>` 依赖 `skin_toolbar_colors.ts` 的硬编码真值表，非 colibris 一律不给。你有两条路：
   - 修改 `SkinColors.ts`，去掉 `skinName !== 'colibris'` 的守卫，让自定义皮肤名也能走 `toolbarColorForTokens()`（前提是你的配色和 colibris 用相同的 4 种亮度色值）
   - 或者在你的 `pad.js` 的 `customStart()` 里自己 `document.createElement('meta')` 注入

3. **考虑补写防白闪脚本**：如果你自己实现了深色配色，又想避免首帧白闪——由于 `darkColor` 间接守卫了内联 IIFE 的输出，你需要在 SkinColors 里让 `darkToolbarColor()` 对你的皮肤返回非 null，或者把防白闪逻辑改写到你自己的 skin 模板里（但目前模板系统不支持 skin 覆盖 pad.html）。

4. **处理 Dark Mode Toggle 的误导性**：非 colibris 但 enableDarkMode=true 时，用户仍然**看得见** Dark Mode checkbox（[pad.ts#L767](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L767)）。如果你的皮肤根本不支持暗配色，最直接的做法是把 `enableDarkMode` 设成 `false`，checkbox 就会彻底隐藏，避免给用户"选项存在但没用"的挫败感。
