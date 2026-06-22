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

---

## 十、enableDarkMode × skinName 的四种组合全链路交叉分析

主题装配系统有两个相互独立但有交叉影响的配置开关：

| 开关 | 取值空间 | 说明 |
|---|---|---|
| **`enableDarkMode`** | `true` / `false`（默认 `true`） | 服务端 Settings 字段；控制"Dark Mode 自动切换"整套功能是否启用 |
| **`skinName`** | `"colibris"` / 任意自定义名称（默认 `"colibris"`） | 皮肤目录名；控制 CSS/JS 资源路径 + toolbar 颜色真值表是否匹配 |

两个开关 × 两个取值 = **4 种组合**，下面按 theme-color、内联预加载脚本、客户端自动切暗、设置面板 checkbox 四个观测点逐一拆解。

### 10.1 enableDarkMode × skinName 的代码决策树

在进入四种组合之前，先梳理两个开关**各自**能切断哪些链路，这样组合行为就可以通过布尔乘法推导：

#### enableDarkMode 切断的代码路径（按执行顺序）

```
(1) 服务端 EJS 模板渲染 pad.html
    │
    ├─ [pad.html#L19] darkColor = settings.enableDarkMode
    │                    ? skinColors.darkToolbarColor(...)
    │                    : null
    │
    ├─ [pad.html#L57] if (configuredColor)
    │                  <meta theme-color content=...
    │                    if (darkColor) media="(prefers-color-scheme: light)"
    │                  ↑ darkColor=null 时，亮 meta 不带 media 属性
    │
    ├─ [pad.html#L58] if (darkColor)
    │                  <meta theme-color content=... media="(prefers-color-scheme: dark)"
    │                  ↑ darkColor=null 时，暗 meta 彻底不输出
    │
    └─ [pad.html#L61] if (darkColor)
                       <script> 内联 IIFE（防白闪）</script>
                       ↑ darkColor=null 时，整段脚本不输出

(2) 客户端 CLIENT_VARS 消息组装 [PadMessageHandler.ts#L1340]
    └─ enableDarkMode: settings.enableDarkMode  原样传给浏览器

(3) 客户端 pad.ts init()
    ├─ [pad.ts#L764] 自动切暗：必须同时满足 clientVars.enableDarkMode 为 true
    └─ [pad.ts#L767] 显示 checkbox：必须满足 clientVars.enableDarkMode 为 true
```

#### skinName 切断的代码路径

```
(1) 服务端 SkinColors 颜色解析
    ├─ [SkinColors.ts#L14]  configuredToolbarColor()：非 colibris → null
    └─ [SkinColors.ts#L29]  darkToolbarColor()：非 colibris → null
    （这一步的 null 会通过 darkColor 变量和 §10.1.1 的 enableDarkMode 守卫叠加）

(2) 服务端 pad.html 静态资源路径拼接
    ├─ [pad.html#L93]  <link href="static/skins/<skinName>/pad.css">
    └─ [pad.html#末尾] <script src="static/skins/<skinName>/pad.js">
    （通用拼接，不做 colibris 判断，skinName 只影响路径）

(3) 服务端 Skin Variants Builder 弹窗 DOM
    └─ [pad.html#L658] if (settings.skinName == 'colibris') → 仅 colibris 输出

(4) 客户端视觉效果
    └─ 取决于 <skinName>/pad.css 是否定义了 .*-toolbar / .*-background / .*-editor
       等 variant class 的 CSS 规则（不涉及 enableDarkMode）
```

### 10.2 四种组合速查表

假设条件：`skinVariants = "super-light-toolbar super-light-editor light-background"`，系统 macOS 深色模式。

| | **组合 A**<br>`colibris` + `enableDarkMode=true` | **组合 B**<br>`colibris` + `enableDarkMode=false` | **组合 C**<br>自定义（no-skin） + `enableDarkMode=true` | **组合 D**<br>自定义（no-skin） + `enableDarkMode=false` |
|---|---|---|---|---|
| **① theme-color meta** | ✅ 亮 `#ffffff` + `media="(prefers-color-scheme: light)"`<br>✅ 暗 `#485365` + `media="(prefers-color-scheme: dark)"` | ✅ 亮 `#ffffff`<br>❌ **不带 media**<br>❌ 暗 meta 不输出 | ❌ 完全无 meta | ❌ 完全无 meta |
| **② 内联预加载 IIFE** | ✅ 输出（防白闪） | ❌ 不输出 | ❌ 不输出 | ❌ 不输出 |
| **③ 客户端自动切暗** | ✅ 切暗 + 视觉生效 | ❌ 逻辑短路，不执行 | ⚠️ 逻辑执行，classList 变<br>但 CSS 无规则，视觉不变 | ❌ 逻辑短路，不执行 |
| **④ 设置面板 Dark Mode checkbox** | ✅ 显示 + checked + 生效 | ❌ 隐藏 | ⚠️ 显示 + checked<br>但切换无视觉效果 | ❌ 隐藏 |
| **⑤ #skinvariantsbuilder 弹窗** | ✅ DOM 存在 + 显示 + 生效 | ✅ DOM 存在 + 显示 + 生效（不依赖 enableDarkMode） | ❌ DOM 不存在 | ❌ DOM 不存在 |
| **⑥ favicon / robots** | ✅ 标准 fallback 链 | ✅ 标准 fallback 链 | ✅ 标准 fallback 链 | ✅ 标准 fallback 链 |
| **⑦ CSS / JS 资源加载** | ✅ `skins/colibris/pad.css|js` | ✅ `skins/colibris/pad.css|js` | ✅ `skins/no-skin/pad.css|js`（pad.css 空文件） | ✅ `skins/no-skin/pad.css|js` |
| **⑧ 首帧视觉效果** | 暗色（IIFE 先改 class，CSS 随后加载） | 亮配色（设置里的 super-light-* 生效） | 亮配色（无 IIFE，默认 class 生效；CSS 空） | 亮配色（同上） |

### 10.3 组合 A：colibris + enableDarkMode=true（默认出厂配置）

完整流程在 §八 和 §九已经详细拆解，这里只给最简洁的链路：

```
Server:
  enableDarkMode=true, skinName=colibris
    → configuredColor = "#ffffff", darkColor = "#485365"
    → pad.html 输出双 <meta theme-color> + 内联 IIFE

Client:
  CLIENT_VARS.enableDarkMode = true
    → pad.ts#L764 条件满足：updateSkinVariantsClasses(dark classes)
    → pad.ts#L767 条件满足：checkbox 显示 + checked
  切 checkbox → pad_editor.ts#L140 事件绑定 → 正常亮/暗切换
  #skinvariantsbuilder → pad.html#L658 条件满足 → DOM 渲染 + 生效
```

**关键点**：双 `<meta>` + `media` 属性 + 内联 IIFE 三者配合，iOS Safari 首帧即正确暗色，零白闪。

### 10.4 组合 B：colibris + enableDarkMode=false

这是**最容易被忽略的组合**，因为 enableDarkMode=false 时 theme-color 的输出不是"完全没了"，而是**降级为单 meta**。

#### 10.4.1 theme-color：单 meta，不带 media

决策链：

```
pad.html#L18: configuredColor = configuredToolbarColor("colibris", skinVariants)
              → "#ffffff"  (因为 colibris，能查表)
pad.html#L19: darkColor      = settings.enableDarkMode ? darkToolbarColor("colibris") : null
                           = false ? "#485365" : null
                           = null

pad.html#L57: if (configuredColor) → true
                <meta name="theme-color" content="#ffffff"
                <% if (darkColor) { %> media="(prefers-color-scheme: light)"<% } %>
                ↑ darkColor=null → 不输出 media 属性 →
                最终：<meta name="theme-color" content="#ffffff">

pad.html#L58: if (darkColor) → false
                暗 meta 完全不输出
```

**结果**：只输出一个 `<meta name="theme-color" content="#ffffff">`，没有任何 `media` 属性。无论系统偏好是亮还是暗，移动端地址栏都是白色。

**和组合 A 的区别**：
- 组合 A 有两个 meta，iOS Safari 在 HTML 解析时根据 `prefers-color-scheme` 二选一
- 组合 B 只有一个 meta，iOS Safari 无条件用 `#ffffff`

#### 10.4.2 内联预加载 IIFE：完全不输出

```
pad.html#L61: <% if (darkColor) { %> ... IIFE ... <% } %>
              darkColor=null → 整段脚本不输出
```

**结果**：没有防白闪脚本。即使系统是深色模式，`<html>` 在 CSS 加载之前保持服务端写入的亮配色 class（`super-light-toolbar super-light-editor light-background`），首帧亮底。

#### 10.4.3 客户端自动切暗：逻辑短路

[pad.ts#L764](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L764) 的四个条件里 `window.clientVars.enableDarkMode` 是 `false` → `&&` 短路 → 整段不执行。`updateSkinVariantsClasses()` 不会被调用。

**结果**：`<html>` class 始终等于服务端写入的默认值，不会被自动切暗。

#### 10.4.4 设置面板 checkbox：隐藏

[pad.ts#L767](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L767) 的 `if (window.clientVars.enableDarkMode)` 为 false → 不执行 `.prop('hidden', false)` → `<p id="theme-toggle-row">` 保持模板里默认的 `hidden` 属性（[pad.html#L303](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L303)）。

**结果**：用户完全看不到 Dark Mode 的开关。

#### 10.4.5 Skin Variants Builder：完全不受影响

`#skinvariantsbuilder` 弹窗的 DOM 守卫在 [pad.html#L658](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L658)：

```ejs
<% if (settings.skinName == 'colibris') { %>
```

这层判断**只看 skinName**，和 enableDarkMode 完全无关。所以即使用户关了 enableDarkMode，只要 skinName 是 colibris，访问 `/p/test#skinvariantsbuilder` 依然能看到弹窗，且弹窗的下拉框切换依然能调用 `updateSkinVariantsClasses()` 改变配色。

**这是一个有趣的交叉行为**：管理员关了 enableDarkMode（不希望普通用户自动切暗），但 Skin Variants Builder 仍然可以手动选择 dark 变体。两者在代码里是完全独立的通道：

| 通道 | 守卫开关 |
|---|---|
| 自动切暗 + checkbox | `enableDarkMode` |
| Skin Variants Builder 手动选变体 | `skinName == 'colibris'` |

### 10.5 组合 C：自定义皮肤 + enableDarkMode=true（最"迷惑"的组合）

核心现象：enableDarkMode 打开了，但因为 skinName 非 colibris 导致 `configuredColor` 和 `darkColor` 全为 null，**theme-color + 预加载 IIFE 都消失了**；而客户端的 `clientVars.enableDarkMode` 又为 true，**自动切暗逻辑照常执行 + checkbox 照常显示**。

#### 10.5.1 theme-color + 预加载 IIFE：双双消失

```
pad.html#L18: configuredColor = configuredToolbarColor("no-skin", ...)
              → SkinColors.ts#L14: "no-skin" !== "colibris" → return null
pad.html#L19: darkColor      = enableDarkMode=true ? darkToolbarColor("no-skin") : null
                            = darkToolbarColor("no-skin")
                            → SkinColors.ts#L29: "no-skin" !== "colibris" → return null

pad.html#L57: configuredColor=null → 亮 meta 不输出
pad.html#L58: darkColor=null → 暗 meta 不输出
pad.html#L61: darkColor=null → IIFE 不输出
```

**结果**：移动端地址栏用浏览器默认色（通常白色），首帧无防白闪。

#### 10.5.2 客户端自动切暗：逻辑执行但视觉空操作

服务端在组装 CLIENT_VARS 时（[PadMessageHandler.ts#L1333-L1340](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/handler/PadMessageHandler.ts#L1333-L1340)）：

```typescript
const clientVars = {
  skinName: settings.skinName,          // "no-skin"
  skinVariants: settings.skinVariants,
  randomVersionString: ...,
  accountPrivs: ...,
  enableDarkMode: settings.enableDarkMode, // true
  ...
};
```

`enableDarkMode` 字段**完全独立于 skinName** 赋值，不做任何交叉判断。所以客户端收到的 `clientVars.enableDarkMode` 还是 `true`。

接下来的客户端逻辑和组合 A 完全相同：

```
pad.ts#L764 条件全满足 → updateSkinVariantsClasses(dark classes)
  → 6 个 DOM 根 remove 所有 variant class
  → 6 个 DOM 根 add 'super-dark-editor dark-background super-dark-toolbar'
  → updateThemeColorMeta(dark classes)
     → document.querySelectorAll('meta[name="theme-color"]')
     → 长度为 0 → 提前 return，什么都不做
```

**最终效果**：`html` 等节点的 classList 确实被改成了暗色值，但 `no-skin/pad.css` 是空文件，没有任何 `.super-dark-toolbar .toolbar { ... }` 规则 → **视觉零变化**。

#### 10.5.3 设置面板 checkbox：显示但切换无效

```
pad.ts#L767: if (window.clientVars.enableDarkMode) → true
  → $('#theme-toggle-row').prop('hidden', false) → checkbox 显示
  → $('#options-darkmode').prop('checked', skinVariants.isDarkMode())
  → isDarkMode() 检查 html.super-dark-editor → 前一步自动切暗已加了 class → true
```

用户点击 checkbox：

```
pad_editor.ts#L140-L150: 事件已绑定（pad_editor.ts 在 padBootstrap.js 里 require 加载，
                        整个模块不做 enableDarkMode 判断，事件绑定无条件执行）
  → setDarkModeInLocalStorage(true/false)  写入 localStorage
  → updateSkinVariantsClasses(...)         同 10.5.2，视觉零效果
```

**最终效果**：checkbox 看得见、能点、checked 状态会变、localStorage 也会写，但页面颜色**完全不变**。用户体验是"这个开关坏了"。

#### 10.5.4 Skin Variants Builder：DOM 不存在

由 [pad.html#L658](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L658) 的硬编码守卫控制，和 enableDarkMode 无关。skinName 不是 colibris → 整个 `<div id="skin-variants">` 不写入 DOM。

### 10.6 组合 D：自定义皮肤 + enableDarkMode=false（最干净的组合）

这是组合 C 的"修复版"——关掉 enableDarkMode 后，客户端自动切暗逻辑被短路、checkbox 被隐藏，消除了组合 C 的 UX 陷阱。

#### 10.6.1 theme-color + 预加载 IIFE：仍然双双消失

和组合 C 完全相同。因为 `configuredColor` / `darkColor` 是否为 null，决定于 `SkinColors.ts` 里的 `skinName !== 'colibris'` 守卫，和 enableDarkMode 无关：

```
configuredColor = configuredToolbarColor("no-skin", ...) → null
darkColor      = enableDarkMode=false ? ... : null      → null

→ 两个 meta 都不输出
→ IIFE 不输出
```

**注意**：enableDarkMode 从 true→false 的切换**不会改变 theme-color 的输出结果**，因为 configuredColor 已经是 null 了。这里 enableDarkMode 的作用被短路。

#### 10.6.2 客户端自动切暗：逻辑短路

```
clientVars.enableDarkMode = false
→ pad.ts#L764 的 && 短路 → 不执行 updateSkinVariantsClasses()
```

`<html>` 的 class 保持服务端写入的默认值。

#### 10.6.3 设置面板 checkbox：隐藏

```
clientVars.enableDarkMode = false
→ pad.ts#L767 的 if 不满足 → #theme-toggle-row 保持 hidden
```

用户完全看不到 Dark Mode 开关。组合 C 的 UX 陷阱被消除。

#### 10.6.4 Skin Variants Builder：DOM 不存在

同组合 C，由 skinName 控制。

### 10.7 enableDarkMode 的完整传播链（从 settings.json 到浏览器）

```
settings.json
  └─ "enableDarkMode": true/false
       │
       ▼
Settings.ts 定义默认值 + 加载
  ├─ [Settings.ts#L435]    默认值 = true
  ├─ [Settings.ts#L177]    类型声明 enableDarkMode: boolean
  └─ [Settings.ts#L876]    getPublicSettings() 暴露给模板的字段
       │
       ├───────────────────────────────────────────────┐
       │                                               │
       ▼                                               ▼
服务端模板渲染                                        CLIENT_VARS 消息组装
  ├─ pad.html#L19: darkColor = enableDarkMode           [PadMessageHandler.ts#L1340]
  │                  ? darkToolbarColor(...) : null        clientVars.enableDarkMode =
  ├─ pad.html#L57: 双 meta 的 media 属性                   settings.enableDarkMode (原封不动)
  ├─ pad.html#L58: 暗 meta 是否输出
  ├─ pad.html#L61: 内联 IIFE 是否输出                      │
  └─ timeslider.html 同理                                  ▼
                                                        pad.ts 客户端逻辑
                                                          ├─ L764: 自动切暗守卫
                                                          └─ L767: checkbox 显示守卫
```

### 10.8 关键交叉点的代码注释解读

为什么 `enableDarkMode` 的行为会和 `skinName` 出现这么复杂的交叉？因为代码里有两处设计意图的碰撞：

1. **SkinColors 的 colibris 守卫**（[SkinColors.ts#L5-L9](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/node/utils/SkinColors.ts#L5-L9)）：

   > "Only the colibris skin has a known mapping... For any other skin we cannot derive the toolbar color server-side and return null so callers can omit the meta rather than emit a misleading value."

   这是一种"**宁可不输出也不输出错误值**"的防御性设计。

2. **pad.html 用 darkColor 同时守卫两件事**（[pad.html#L15-L19](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L15-L19) 注释）：

   > "The dark variant is only emitted when enableDarkMode is on, since that is what gates the client-side auto-switch..."

   darkColor 的存在性同时意味着：
   - (a) skinName 是 colibris（由 SkinColors 保证）
   - (b) enableDarkMode 是 true（由三元表达式保证）

   所以它被拿来同时当**暗 meta 的守卫**和**防白闪 IIFE 的守卫**。这种"一个变量兼两职"的写法让非 colibris + enableDarkMode=true 的组合出现了看似"enableDarkMode 没生效"的错觉——实际上 enableDarkMode=true 了，但 darkColor 被 skinName 的守卫短路成了 null。

3. **客户端完全不看 skinName**（[pad.ts#L764-L770](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L764-L770)）：

   客户端的 4 个判断条件里完全没有 `clientVars.skinName`。这意味着：
   - 服务端可以通过 `SkinColors` 决定"不输出 theme-color 和 IIFE"
   - 但客户端依然会忠实执行自动切暗 + 显示 checkbox 的逻辑
   
   这个设计分割就是组合 C 出现"checkbox 显示但切换无效"的根本原因。

### 10.9 给运维的配置建议（四种组合怎么选）

| 场景 | 推荐组合 | 理由 |
|---|---|---|
| **默认，啥都不改** | A: colibris + enableDarkMode=true | 官方预期行为，体验最佳 |
| **组织内网、强制统一亮色、不希望用户切暗** | B: colibris + enableDarkMode=false | checkbox 隐藏，用户看不到选项；但 theme-color 仍有亮 meta |
| **自定义皮肤 + 该皮肤自带完整 variant CSS** | C-修复: 自定义 + enableDarkMode=true **+** 自行扩展 SkinColors 让 theme-color/IIFE 输出 + 补全 variant CSS | 不建议裸用组合 C，会有 UX 陷阱 |
| **自定义皮肤 + 该皮肤完全不支持暗配色** | D: 自定义 + enableDarkMode=false | 最干净：checkbox 隐藏、自动切暗不触发，消除所有 UX 陷阱 |
| **自定义皮肤 + 该皮肤想自己实现暗模式机制（不依赖 variant class）** | D: 自定义 + enableDarkMode=false | 关闭系统默认机制，自己在 pad.js 的 customStart() 里实现；同时建议自写 theme-color meta |

---

## 十一、`localStorage.ep_darkMode` 三态 × 双开关的优先级与最终表现

`localStorage.ep_darkMode` 这个键看似只有"true/false"布尔，实际在代码里是**三态**的：

| 状态 | `localStorage.getItem('ep_darkMode')` 返回值 | 典型场景 |
|---|---|---|
| **unset（未设置）** | `null`（JS 原生返回值，不是字符串） | 首次访问 / 清了浏览器缓存 / 从不使用 Dark Mode 开关 |
| **显式 true** | 字符串 `"true"` | 用户勾选了 Dark Mode checkbox |
| **显式 false** | 字符串 `"false"` | 用户取消勾选了 Dark Mode checkbox（覆盖系统暗色偏好） |

三种状态在四个观测节点上的行为完全不同，且会和 `enableDarkMode`、`skinName` 两个开关交叉叠加。下面按执行顺序逐节点拆解。

### 11.1 四个观测节点的执行顺序（时间线）

一次典型的 Pad 页面加载，主题相关代码按以下顺序触发：

```
T1  服务端 EJS 渲染 pad.html
      ├─ 计算 configuredColor / darkColor（受 skinName + enableDarkMode 影响）
      ├─ 输出 <meta theme-color>（0/1/2 个，视 darkColor 而定）
      └─ 若 darkColor != null，输出内联 IIFE（含 localStorage 判断）
         ↑ T2 执行点

T2  浏览器解析到 <head> 里的内联 IIFE（如果存在）
      └─ 检查 hash、localStorage.ep_darkMode、matchMedia
         → 命中条件则提前把 <html> 改成暗色 class
         ↑ 防白闪的关键，发生在 CSS 加载之前

T3  CSS 加载、DOM Ready、padBootstrap 执行 → pad.ts init()
      ├─ pad.ts#L764: 客户端自动切暗逻辑
      │              （再次检查 localStorage + matchMedia + enableDarkMode）
      └─ pad.ts#L767: checkbox 显示/隐藏 + 初始 checked 状态
                      （checked 取决于 isDarkMode()，即 html.super-dark-editor）

T4  用户手动点击 #options-darkmode checkbox（如果显示）
      └─ pad_editor.ts#L140: 事件绑定
         ├─ setDarkModeInLocalStorage(true/false)  → 写入 localStorage
         └─ updateSkinVariantsClasses(...)           → 改 DOM class + 调 updateThemeColorMeta
```

**关键点**：T1（服务端）→ T2（内联脚本）→ T3（客户端 init）→ T4（用户交互）是严格时序。T2 早于 CSS 加载，所以它能真正"防白闪"；T3 晚于 DOM Ready，主要负责 iframe 同步和 checkbox 状态；T4 只在用户操作时发生。

### 11.2 localStorage 三态在各节点的判断逻辑

先逐个节点看 localStorage 三个取值分别如何被判定。

#### 节点 T2：内联预加载 IIFE（仅 darkColor != null 时才会出现在 HTML 中）

代码位置：[pad.html#L69-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L69-L85)（timeslider 同构）

```javascript
(function () {
  try {
    if (window.location.hash.toLowerCase() === '#skinvariantsbuilder') return;
    if (localStorage.getItem('ep_darkMode') === 'false') return;   // ★ 三态判断 1
    if (!(window.matchMedia &&
          window.matchMedia('(prefers-color-scheme: dark)').matches)) return;
    // ... 改 <html> 为暗色 class
  } catch (e) { /* fall back to light */ }
})();
```

| `ep_darkMode` 值 | `getItem() === 'false'` | 结果 |
|---|---|---|
| `null`（unset） | `false`（null !== 'false'） | 不 return，继续判断 matchMedia |
| `"true"` | `false`（"true" !== 'false'） | 不 return，继续判断 matchMedia |
| `"false"` | `true` | 直接 return → 保持亮色，不切暗 |

**关键洞察**：这个判断只检查 **`=== 'false'`**，不检查 `=== 'true'`。也就是说——

> **内联 IIFE 只尊重"用户显式选了亮"这一种覆盖**。`"true"` 和 unset 都走"跟随系统偏好"。

这是有意设计：用户若从没碰过开关（unset）或主动勾选了暗色（`"true"`），两种情况都应该让 `matchMedia` 决定。只有用户**主动取消勾选**（`"false"`）才要强制覆盖系统偏好。

#### 节点 T3：pad.ts 客户端自动切暗

代码位置：[pad.ts#L764-L766](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L764-L766)

```typescript
if (window.location.hash.toLowerCase() !== '#skinvariantsbuilder'
    && window.clientVars.enableDarkMode                                 // ★ 守卫 1
    && (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches)
    && !skinVariants.isWhiteModeEnabledInLocalStorage()) {              // ★ 三态判断 2
  skinVariants.updateSkinVariantsClasses([
    'super-dark-editor', 'dark-background', 'super-dark-toolbar'
  ]);
}
```

其中 `isWhiteModeEnabledInLocalStorage()` 定义在 [skin_variants.ts#L75-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L75-L77)：

```typescript
const isWhiteModeEnabledInLocalStorage = () => {
  return localStorage.getItem('ep_darkMode') === 'false';
};
```

和 T2 的内联 IIFE **完全相同的判断逻辑**（也只检查 `=== 'false'`）。这是刻意保持一致的镜像设计，T2 和 T3 的结果始终一致，不会出现"T2 切了暗但 T3 又切回来"的情况。

| `ep_darkMode` 值 | `isWhiteModeEnabledInLocalStorage()` | `!该值`（即 if 条件的第四项） |
|---|---|---|
| `null`（unset） | `false` | `true` → 通过 |
| `"true"` | `false` | `true` → 通过 |
| `"false"` | `true` | `false` → 不通过，保持亮色 |

**另外**，T3 比 T2 多了 `clientVars.enableDarkMode` 这个守卫——T2 存在的前提是 darkColor != null（间接要求 enableDarkMode=true），而 T3 直接用 enableDarkMode 守卫。

#### 节点 T3：checkbox 显示与初始 checked 状态

代码位置：[pad.ts#L767-L770](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad.ts#L767-L770)

```typescript
if (window.clientVars.enableDarkMode) {            // 只看 enableDarkMode
  $('#theme-toggle-row').prop('hidden', false);    // 显示 checkbox
  $('#options-darkmode').prop(
    'checked',
    skinVariants.isDarkMode()                      // ★ 不直接读 localStorage
  );
}
```

这里的设计比较微妙：**checked 状态不直接读 localStorage.ep_darkMode**，而是读 `isDarkMode()`（[skin_variants.ts#L62-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L62-L64)）：

```typescript
const isDarkMode = () => {
  return $('html').hasClass('super-dark-editor');
};
```

也就是检查 `<html>` 上的 DOM class，而这个 class 是 T2（内联 IIFE）或 T3（自动切暗）加上去的。最终效果：

| 场景 | `isDarkMode()` | checkbox checked |
|---|---|---|
| unset + 系统暗色 + enableDarkMode=true + colibris | `true`（T2 或 T3 加了 class） | ✅ checked |
| unset + 系统亮色 | `false`（没人加 class） | ⬜ 未勾选 |
| `"true"` + 系统暗色 | `true` | ✅ checked |
| `"true"` + 系统亮色 | `false`（T2/T3 的 matchMedia 条件不满足，不切暗） | ⬜ 未勾选 |
| `"false"`（任何系统偏好） | `false`（T2/T3 都被 return） | ⬜ 未勾选 |
| enableDarkMode=false | 不执行这段 | hidden，用户看不到 |

**注意 `ep_darkMode="true"` + 系统亮色这个边界情况**：用户显式勾选了暗色，但当前系统是亮色。由于 T2/T3 都要求 `matchMedia(dark).matches` 才会切暗，结果是 `<html>` 保持亮色 class → checkbox 显示未勾选。这个看似反直觉，但实际符合设计——`"true"` 的语义是"跟随系统暗色偏好"，不是"永远强制暗色"。代码里**没有**任何地方提供"强制覆盖系统为暗色"的持久化开关。

#### 节点 T4：用户手动切换 checkbox

代码位置：[pad_editor.ts#L140-L150](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/pad_editor.ts#L140-L150)

```typescript
padutils.bindCheckboxChange($('#options-darkmode'), () => {
  const isDark = padutils.getCheckbox($('#options-darkmode'));
  skinVariants.setDarkModeInLocalStorage(isDark);          // ★ 写 localStorage
  if (isDark) {
    skinVariants.updateSkinVariantsClasses([
      'super-dark-editor', 'dark-background', 'super-dark-toolbar']);
  } else {
    skinVariants.updateSkinVariantsClasses(
      ['super-light-toolbar super-light-editor light-background']);
  }
});
```

`setDarkModeInLocalStorage()` 定义在 [skin_variants.ts#L67-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L67-L69)：

```typescript
const setDarkModeInLocalStorage = (isDark) => {
  localStorage.setItem('ep_darkMode', isDark ? 'true' : 'false');
};
```

**这个节点是 localStorage 三态里 unset → "true"/"false" 的唯一写入点**。注意：
- 没有 `removeItem()`，永远不会写回 unset
- 用户第一次点 checkbox 之后，localStorage 状态就**永久锁定**在 `"true"` 或 `"false"`，除非手动清浏览器数据
- 同时调用 `updateSkinVariantsClasses()` 立即改 DOM，不等下次刷新

#### 节点 T2/T3/T4 共用：updateThemeColorMeta

代码位置：[skin_variants.ts#L20-L25](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L20-L25)

```typescript
const updateThemeColorMeta = (newClasses: string[]) => {
  const metas = document.querySelectorAll('meta[name="theme-color"]');
  if (!metas.length) return;                                                 // ★ 守卫 1
  const color = toolbarColorForTokens(
    newClasses.join(' ').split(/\s+/).filter(Boolean));
  metas.forEach((meta) => { meta.setAttribute('content', color); });         // ★ 全部 meta 都改
};
```

这个函数只有**唯一的调用点**：`updateSkinVariantsClasses()` 的最后一行（[skin_variants.ts#L58](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L58)）。所以只要 DOM class 变化，theme-color meta 就会尝试同步。

关键行为：

1. **非 colibris 皮肤下静默失败**：`metas.length == 0`（因为 SkinColors 返回 null，T1 根本没输出 meta）→ 直接 return，不报错
2. **colibris + enableDarkMode=true 时覆盖两个 meta 的 content**：服务端 T1 输出了两个 meta（各带 `media` 属性），`toolbarColorForTokens()` 根据当前 variant class 算出颜色后，**两个 meta 的 content 都被改成同一个值**。这样用户手动切亮/切暗时，不管浏览器当前匹配哪个 media 查询，都能看到正确颜色
3. **colibris + enableDarkMode=false 时只改一个 meta**：T1 只输出了一个不带 media 的亮 meta，改它就够了

`toolbarColorForTokens()`（[skin_toolbar_colors.ts#L24-L31](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_toolbar_colors.ts#L24-L31)）的逻辑是：遍历 `TOOLBAR_COLORS_IN_CSS_ORDER` 真值表（顺序和 CSS 级联一致），最后一个匹配的 *-toolbar token 获胜；如果没有匹配则返回默认 `#ffffff`。

### 11.3 localStorage 三态 × enableDarkMode × skinName 的完整 24 格矩阵

假设系统偏好为暗色（`matchMedia('(prefers-color-scheme: dark)').matches === true`），其他默认值，按执行顺序看最终表现：

#### 组 A：skinName = colibris

| `enableDarkMode` | `ep_darkMode` | T1 meta 输出 | T2 内联 IIFE | T3 自动切暗 | T3 checkbox 状态 | T4 可交互？ | T4 后 `updateThemeColorMeta` |
|---|---|---|---|---|---|---|---|
| **true** | `null`（unset） | 亮 `#ffffff` media=light<br>暗 `#485365` media=dark | ✅ 切暗（class + CSS 变量生效） | ✅ 再次切暗（幂等） | 显示 + checked | ✅ 可点，点后生效 | ✅ 双 meta content 同步改 |
| **true** | `"true"` | 同上 | ✅ 切暗 | ✅ 切暗 | 显示 + checked | ✅ 可点，点后生效 | ✅ 同步改 |
| **true** | `"false"` | 同上 | ❌ return，不切暗 | ❌ 第四项为 false，不切暗 | 显示 + 未 checked | ✅ 可点，点后生效 | ✅ 同步改 |
| **false** | `null`（unset） | 亮 `#ffffff`，**无 media** | ⬜ IIFE 不存在（darkColor=null） | ❌ enableDarkMode=false，短路 | 隐藏 | ❌ 不可见 | ⬜ 无 T4 交互 |
| **false** | `"true"` | 同上 | ⬜ IIFE 不存在 | ❌ 短路 | 隐藏 | ❌ 不可见 | ⬜ 无 T4 交互 |
| **false** | `"false"` | 同上 | ⬜ IIFE 不存在 | ❌ 短路 | 隐藏 | ❌ 不可见 | ⬜ 无 T4 交互 |

**组 A 观察点**：

- enableDarkMode=true 时，`"false"` 是真正有区分度的状态——它强制保持亮，而 unset 和 `"true"` 都跟随系统暗色
- enableDarkMode=false 时，localStorage 的三个值**对视觉完全没有影响**，因为 T2 IIFE 不输出、T3 自动切暗被短路、T4 checkbox 隐藏，localStorage 根本读不到
- enableDarkMode=false 时 T1 只输出单 meta 不带 media，**且永远不会被 JS 改写**（因为 checkbox 隐藏，没有 T4 调用 `updateSkinVariantsClasses`，也就不会触发 `updateThemeColorMeta`）

#### 组 B：skinName = 自定义（no-skin）

| `enableDarkMode` | `ep_darkMode` | T1 meta 输出 | T2 内联 IIFE | T3 自动切暗 | T3 checkbox 状态 | T4 可交互？ | T4 后 `updateThemeColorMeta` |
|---|---|---|---|---|---|---|---|
| **true** | `null`（unset） | ❌ 无 meta（SkinColors 返回 null） | ⬜ IIFE 不存在（darkColor=null） | ✅ 改 classList，但 CSS 空 → 视觉零效果 | 显示 + checked（基于 class，和视觉不一致） | ✅ 可点，但切换无视觉变化 | ⬜ metas.length=0，静默 return |
| **true** | `"true"` | ❌ 无 meta | ⬜ IIFE 不存在 | ✅ 改 classList，视觉零效果 | 显示 + checked | ✅ 可点，无视觉变化 | ⬜ 静默 return |
| **true** | `"false"` | ❌ 无 meta | ⬜ IIFE 不存在 | ❌ isWhiteMode 为 true，不改 | 显示 + 未 checked | ✅ 可点，无视觉变化 | ⬜ 静默 return |
| **false** | `null`（unset） | ❌ 无 meta | ⬜ IIFE 不存在 | ❌ enableDarkMode=false，短路 | 隐藏 | ❌ 不可见 | ⬜ 无 T4 交互 |
| **false** | `"true"` | ❌ 无 meta | ⬜ IIFE 不存在 | ❌ 短路 | 隐藏 | ❌ 不可见 | ⬜ 无 T4 交互 |
| **false** | `"false"` | ❌ 无 meta | ⬜ IIFE 不存在 | ❌ 短路 | 隐藏 | ❌ 不可见 | ⬜ 无 T4 交互 |

**组 B 观察点**：

- **T2 内联 IIFE 在自定义皮肤下永远不存在**，因为 `SkinColors.darkToolbarColor()` 对非 colibris 返回 null，darkColor 永远是 null → 守卫 `if (darkColor)` 永远不通过。这意味着在自定义皮肤 + enableDarkMode=true 组合下，防白闪机制完全失效
- T3 的 checkbox checked 状态会出现"视觉与 checked 不一致"：`ep_darkMode=null` 时 T3 自动切暗把 `<html>` 加成 `super-dark-editor` → `isDarkMode()` 返回 true → checkbox 显示 checked，但因为 no-skin/pad.css 是空文件，视觉上是亮色
- `updateThemeColorMeta` 在所有非 colibris 场景下都是**静默空操作**，因为 T1 没输出 meta，`metas.length === 0` 直接 return

### 11.4 localStorage 三态的优先级总结

从所有场景提炼出的**决策优先级**（从高到低）：

```
优先级 1：skinName !== 'colibris'
           → theme-color: 无
           → 内联 IIFE: 不输出（防白闪失效）
           → updateThemeColorMeta: 空操作
           （colibris 是整个系统的前置条件）

优先级 2：enableDarkMode === false
           → 内联 IIFE: 不输出（darkColor 三元返回 null）
           → T3 自动切暗: 短路（&& 判断 clientVars.enableDarkMode）
           → checkbox: 隐藏（用户根本碰不到）
           （enableDarkMode 是客户端所有暗模式逻辑的总开关）

优先级 3：localStorage.ep_darkMode === 'false'
           → T2 IIFE: 直接 return，不切暗
           → T3 自动切暗: !isWhiteModeEnabledInLocalStorage() 为 false，不切暗
           （用户显式覆盖：我不管系统是不是暗色，我就要亮）

优先级 4：matchMedia('(prefers-color-scheme: dark)').matches
           → true: 切暗
           → false: 保持亮
           （只有前三个优先级都放行，才跟随系统）

优先级 5：localStorage.ep_darkMode === 'true'
           → 不影响当前会话的切暗决策（没有任何代码检查 === 'true'）
           → 只在下次刷新、且 matchMedia 已经是 dark 时才"生效"
           → 实际效果和 unset 完全相同
```

**最重要的两个反直觉结论**：

1. **`"true"` 和 unset 语义完全相同**。代码里**没有任何地方**检查 `ep_darkMode === 'true'`，只检查 `=== 'false'`。`"true"` 的存在仅仅是为了"让 checkbox 在下次打开时保持选中状态"——但那个 checked 状态其实是靠 `isDarkMode()`（读 DOM class）间接算出来的，`"true"` 本身不驱动任何切暗逻辑。

2. **`"true"` 不能强制亮系统为暗**。如果你在系统亮色时勾选了 Dark Mode checkbox：
   - T4 会立刻调用 `updateSkinVariantsClasses(dark)` 改 DOM（这次生效）
   - 同时写入 `ep_darkMode = "true"`
   - 下次刷新时，T2（如果存在）和 T3 都会先检查 `matchMedia(dark).matches`，系统亮 → 不切暗
   - 结果是你**刷新后又变回亮色**，即使 localStorage 里是 `"true"`

### 11.5 localStorage 的所有读写点汇总

| 位置 | 操作 | 触发时机 |
|---|---|---|
| [pad.html#L74](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/pad.html#L74) | `getItem('ep_darkMode') === 'false'`（只读） | T2 内联 IIFE，CSS 加载前（仅 colibris + enableDarkMode=true） |
| [timeslider.html#L55](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/templates/timeslider.html#L55) | 同上 | T2 内联 IIFE，timeslider 页 |
| [skin_variants.ts#L75-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L75-L77) | `getItem('ep_darkMode') === 'false'`（isWhiteModeEnabledInLocalStorage） | T3 pad.ts 自动切暗守卫 |
| [skin_variants.ts#L71-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L71-L73) | `getItem('ep_darkMode') === 'true'`（isDarkModeEnabledInLocalStorage） | **导出但整个代码库零调用**（死代码？） |
| [skin_variants.ts#L67-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L67-L69) | `setItem('ep_darkMode', 'true' | 'false')`（setDarkModeInLocalStorage） | T4 用户点 checkbox 时写入 |

**特别注意**：`isDarkModeEnabledInLocalStorage()`（检查 `=== 'true'`）这个函数在 [skin_variants.ts#L119](file:///d:/fz/0601-2/solo-dogfeeding/code/77-etherpad-lite/src/static/js/skin_variants.ts#L119) 被导出，但整个代码库里**没有任何 import 或调用**——它是遗留或预留接口，目前所有判断都只看 `=== 'false'`。这进一步印证了结论 1：`"true"` 在当前版本里不起任何直接驱动作用。

### 11.6 完整状态流转图（从用户首次访问起）

```
首次访问（localStorage 无 ep_darkMode 键）
  │
  ▼
unset 状态
  │
  ├─ colibris + enableDarkMode=true + 系统暗
  │    → T2 IIFE 切暗（防白闪）
  │    → T3 再切暗（幂等）+ checkbox 显示为 checked
  │    → 用户如果保持不动：一直是 unset
  │    │
  │    ├─ 用户取消勾选 checkbox
  │    │    → ep_darkMode = "false"
  │    │    → DOM 立刻改亮 + meta 同步改
  │    │    → 下次刷新：T2/T3 都命中 "==='false'" 判断，保持亮
  │    │
  │    └─ 用户先不操作，后来系统切亮又切回暗
  │         → 不触发重渲染，保持当前状态
  │         → 刷新后：重新走 unset 分支
  │
  ├─ colibris + enableDarkMode=false
  │    → 无 IIFE，无自动切暗，checkbox 隐藏
  │    → ep_darkMode 永远保持 unset（除非手动 F12 改 localStorage）
  │
  └─ 非 colibris（无论 enableDarkMode）
       → 无 IIFE，无 meta
       → enableDarkMode=true 时 checkbox 显示但视觉不一致
       → 勾选 checkbox 写入 "true"，但刷新后系统暗才生效（系统亮则白写）
```

### 11.7 给开发者/运维的 localStorage 相关注意事项

1. **如果要强制所有用户亮模式**：只设 `enableDarkMode=false` 就够了。不需要去清用户的 `ep_darkMode` localStorage，因为 enableDarkMode=false 时所有 localStorage 读取代码路径都被短路，值写了也白写。

2. **如果要强制所有用户暗模式**：仅靠现有代码做不到。需要改造以下两处之一：
   - 去掉 T2 和 T3 的 `matchMedia(dark).matches` 守卫
   - 或在服务端把 `skinVariants` 默认值改成 `super-dark-toolbar dark-background super-dark-editor`，并把 enableDarkMode=false（这样 T1 就把暗色 class 写进 `<html>`，同时不触发任何自动切换逻辑）

3. **排查"我明明勾了暗色但刷新又变亮"**：检查系统 `prefers-color-scheme`。当前亮系统下，勾选暗色只在本次会话生效；刷新后 T2/T3 的 `matchMedia` 条件不满足，会回到亮。这是**按设计行为**不是 bug。

4. **自定义皮肤 + enableDarkMode=true 的 UX 修复**：
   - 如果你的皮肤有 variant CSS，记得同时扩展 `SkinColors.ts` 让 `configuredToolbarColor` / `darkToolbarColor` 返回非 null——这样 T1 才会输出 meta 和 IIFE，防白闪 + 地址栏颜色才能正常工作
   - 如果你的皮肤没有 variant CSS，务必把 `enableDarkMode=false`，否则用户会看到"有 checkbox 但点了没用"的坏体验

5. **调试 localStorage 状态**：在 DevTools Console 里执行：
   ```javascript
   localStorage.getItem('ep_darkMode')  // null / "true" / "false"
   document.documentElement.className   // 看当前实际生效的 variant class
   document.querySelectorAll('meta[name="theme-color"]')  // 看输出了几个 meta
   ```
