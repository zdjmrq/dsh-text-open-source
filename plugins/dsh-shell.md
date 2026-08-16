# 【文字开源】dsh-shell — 把 DeepSeek Harness Web UI 装进原生桌面窗口的一键桌面壳

> **这是什么**：本文件是「文字开源」描述，不是代码。把本文件全文粘贴给你 DSH 里的 AI 会话，
> AI 即可为你复刻出功能相同的桌面壳；你也可以阅读本文件理解其原理，并自行微调需求。
>
> **形态提醒（重要）**：dsh-shell **不是 Cordis 插件**，而是一个独立的 **Electron 桌面应用
> （Windows 专属）**。它不注册任何 Service / Event / Tool，不进 `cordis.yml` / agent preset，
> 不挂载在 DSH 进程里；它通过 **HTTP 端口（复用/拉起 DSH web 服务）+ 页面 DOM 注入** 与 DSH
> 交互。复刻时请按 Electron 应用形态实现，不要硬套 Cordis 插件框架。
>
> **对应源码**：https://github.com/zdjmrq/dsh-shell（描述基于 commit `539e866794aeb02c46f1b1062f3bb2db81c299ff`；
> 2026-08-16 起在体验增强层新增**智能右键菜单**与**消息重新编辑**两项能力，见第 2 / 5.15 / 5.16 节）｜许可证：MIT
>
> **最近更新**：2026-08-16

## 1. 插件概述

dsh-shell 是一个 Windows 桌面壳：把 DeepSeek Harness 的 Web UI（默认 `http://127.0.0.1:3080`）
包进一个原生桌面窗口，双击即用，解决"每次都要开终端敲 `pnpm dsh web` 再开浏览器"的痛点。
它的核心定位是**轻量壳、不碰官方 UI**——只注入窗口边框层（自绘顶栏：拖动条 + 全屏/最小化/
最大化/关闭按钮），不修改 DSH 的任何页面结构或 UI 面板，皮肤、侧边栏、会话插件等全部照常工作。
在边框层之外，它还带一层**体验增强**（智能右键菜单、用户消息重新编辑），同样是 DOM 注入实现、
不修改 DSH 任何源码。启动时自动探测端口：已有 DSH 服务在跑就直接复用开窗（关窗不影响原服务），
没有则按配置自动拉起服务、就绪后开窗，关窗只杀掉自己启动的进程。它还作为 dsh-attention-notifier
的**呈现端**：消费配套插件的 `GET /dsh-attention` 契约，用 Windows 任务栏按钮闪烁（微信新消息同款）
提醒"需要介入 / 一轮工作完成"。

## 2. 功能规格

用户可感知的行为清单（可验收）：

- **一键启动 / 复用 DSH 服务**：双击 exe → 探测 3080 端口（可在 `config.json` 改）：
  - 已有服务在跑 → 直接以窗口打开，关闭窗口**不会**影响原服务；
  - 没有服务 → 按 `config.json` 的 `start.command`（默认 `pnpm dsh web`）自动拉起 DSH，
    就绪后打开窗口；**关闭窗口时只杀掉自己启动的服务**（taskkill 整棵进程树），不留后台进程。
- **单实例**：重复双击 exe 只把已有窗口 restore + show + focus 带到前台，不会开第二个窗口。
- **无边框自绘顶栏**：默认无边框窗口，顶部一条 36px 自绘顶栏（高度可配 `overlayHeight`），
  可**拖动窗口**；右上角依次是**全屏 / 最小化 / 最大化 / 关闭**四个按钮（内联 SVG 图标）。
  按钮颜色使用 DSH 的 CSS 变量，**换皮肤、切深浅色时自动跟随页面变色**。
- **全屏纯享**：全屏按钮（四角方框图标）长悬停显示「全屏纯享」提示，点击进入完全全屏
  （隐藏任务栏）；全屏后按钮组淡出隐藏，**鼠标靠近屏幕顶部（≤60px）时柔和淡入**，可点按钮
  退出全屏，全屏时也可直接点关闭。退出全屏：再点全屏按钮、按 **Esc** 或 **F11** 均可；
  F11 也可进入全屏。
- **回退原生标题栏**：把 exe 旁 `config.json` 的 `window.overlay` 改为 `false` → 回到传统
  原生标题栏（此时 F11/Esc 全屏仍可用；注入顶栏与注意力上报桥随之停用）。
- **任务栏注意力提醒（需 DSH 侧配套插件，推荐 dsh-attention-notifier）**：
  - **需要介入**（审批/提问挂起）或**一轮工作完成**后，只要"你不在"——窗口失焦/最小化，
    或聚焦但超过 8 秒没有任何操作——任务栏按钮就闪烁（闪几轮后常驻淡红，微信同款）；
  - **回到对话**（窗口聚焦，或窗口内任意鼠标移动/点击/滚轮/键盘操作）立即熄灭；
  - 完成事件若发生在你正活跃地看着窗口时，视为已看到，**不闪**。
- **智能右键菜单（体验增强）**：页面内右键弹出原生菜单，按上下文动态生成：
  - **有选中文字** → `复制`、`剪切`（可编辑处）+ 分隔线 + `发送到新对话`（在当前工作区
    **新建会话**，把选中文字写入输入框并聚焦，**不自动发送**）；
  - **无选中、点在输入框** → `粘贴`、`剪切`、`复制`、`撤销`（按当前是否可操作动态取舍）；
  - **无选中、非输入框** → `复制`（可复制时）或不弹菜单。
- **消息重新编辑（体验增强）**：每条**用户消息**的操作条（复制键旁边）新增「重新编辑」按钮，
  点击把该条消息正文**召回输入框**并聚焦、光标置尾（原消息保留，编辑后重发）；打断进行中的任务后，
  即可直接对最后一条消息改发，省去复制粘贴。按钮复用 DSH 操作键样式，随主题变色。
- **启动诊断**：所有启动/停止细节记录在 exe 旁边的 `dsh-shell.log`；找不到 DSH checkout /
  服务命令启动失败 / 服务提前退出 / 等待就绪超时（120 秒）→ 自绘错误页给出 config 位置、
  排查命令与日志路径。
- **外链处理**：页面内点击非 `BASE_URL` 的 http/https 链接 → 用系统默认浏览器打开，壳内
  不导航、不新开窗口。
- **打包分发**：`npm run dist:dir` 出免安装绿色版；`npm run dist` 同时出绿色版 + NSIS 安装器
  （自动创建桌面/开始菜单快捷方式、免管理员、带卸载程序）；`npm run dist:zip` 绿色版 + 发布 zip。

## 3. 技术路线

- **应用形态**：**Electron 桌面应用（Windows 专属），不是 Cordis 插件**。单进程、单 `BrowserWindow`，
  三部分：主进程 `main.js`（Node.js，承载全部业务逻辑）、`preload.js`（contextBridge 桥）、
  页面注入脚本（纯 DOM/JS，注入进 DSH 页面运行）。无 TypeScript / bundler，纯 CommonJS。
- **与 DSH 的集成方式**：**不加载进 DSH 进程**，不进 `cordis.yml` / agent preset / patch 层。
  通过两个薄契约交互：
  1. **HTTP 层**：探测 / 复用 / 拉起 `http://127.0.0.1:3080`；消费 `GET /dsh-attention`
     （由配套插件 dsh-attention-notifier 挂在 DSH 宿主 `webServer` 上）；
  2. **页面 DOM 层**：`dom-ready` 后 `insertCSS` + `executeJavaScript` 注入顶栏、注意力轮询器
     与体验增强（右键菜单入口、消息重新编辑按钮），不修改 DSH 任何源码。
- **进程模型与安全基线**：`contextIsolation: true`、`nodeIntegration: false`、
  `backgroundThrottling: false`（最小化/遮挡时页面定时器不节流，注意力轮询必须照常跑）；
  preload 只暴露 `window.dshShell` 窗口控制桥，不暴露任何 Node 能力；渲染进程经
  `ipcMain` 发窗口操作请求，主进程执行。
- **核心机制**：端口探测用 `net.connect`（600ms 超时）；服务启动用
  `spawn('cmd.exe', ['/d','/s','/c', command], { windowsHide: true })`；停止用
  `taskkill /pid <pid> /T /F` 杀整棵进程树；单实例用 `app.requestSingleInstanceLock()`；
  全屏/最大化/焦点状态经 `webContents.send` 推给页面；任务栏提醒用 `win.flashFrame(true/false)`
  （Windows 上有限次数闪烁 → 按钮停留高亮，即"常驻淡红"）。体验增强：右键菜单用
  `webContents.on('context-menu')` + `Menu.buildFromTemplate`（复制/剪切/粘贴/撤销用内置 `role`
  原生执行，"发送到新对话" 经 `executeJavaScript` 调注入函数）；消息重新编辑由注入脚本操作
  DSH 页面 DOM（受控 textarea 用原生 value setter 写入）。
- **关键外部依赖**：`electron` ^42.0.0、`electron-builder` ^26.0.0（devDependencies）；
  运行时系统能力：`cmd.exe`、`taskkill`、Windows 任务栏 `flashFrame`、NSIS 安装器；
  启动命令要求 `pnpm` 在 PATH（默认 `pnpm dsh web`）。
- **与其他 dsh-\* 仓库的配套关系**：与 [dsh-attention-notifier](https://github.com/zdjmrq/dsh-attention-notifier)
  成对（**判/显分离**）：插件在 DSH 侧只做判定、聚合状态到 `GET /dsh-attention`；壳注入页面的
  轮询器每秒读取该端点 + 活动检测判定"你不在"，经 preload 桥上报主进程闪烁。壳**只消费
  `GET /dsh-attention` 这一个契约**（字段见第 5 节），任何实现同一契约的提醒插件都能配合，
  不绑定具体实现；没装这类插件时壳不探测 DSH 页面状态、任务栏也不闪。
- **版本敏感点**：electron/electron-builder 为声明版本范围；`flashFrame` 是 Windows 平台行为；
  页面注入依赖 DSH 的 CSS 变量 `--dsw-alias-label-secondary`（带兜底色）与根布局
  `html/body/#root{height:100%}` 假设——DSH 若改动这些会漂移（详见第 5 节）。

## 4. 结构设计

文件/目录布局（仓库根目录，不含 node_modules/release）：

| 路径 | 职责 |
| --- | --- |
| `main.js` | 主进程全部逻辑（~750 行）：配置加载与解析、日志、端口探测、服务拉起/停止、窗口创建、顶栏 CSS/JS 注入、注意力提醒主进程侧、智能右键菜单、快捷键、外链策略、应用生命周期 |
| `preload.js` | `contextBridge` 暴露 `window.dshShell` 最小窗口控制桥（21 行）：minimize / toggleMaximize / close / toggleFullscreen / onFullscreenChange / onMaximizeChange / setAttention / onFocusChange |
| `config.json` | 运行时配置：`port`、`start.cwd`、`start.command`、`window.overlay`、`window.overlayHeight`；打包时作为 `extraFiles` 放到 exe 旁供用户改 |
| `package.json` | 元信息 + electron-builder 打包配置（win: dir + nsis，x64；appId `local.dsh.shell`，productName `DSH Desktop`） |
| `assets/icon.png` | 应用窗口/任务栏/快捷方式图标（打包 icon 与窗口 icon） |
| `assets/icon-attn.png` | **存在但当前代码未引用**（待确认：疑似任务栏叠加图标预留，现实现用 flashFrame 且 `setOverlayIcon(null,'')` 恒清除） |
| `make-zip.ps1` | 把 `release\win-unpacked` 复制为临时目录再 `Compress-Archive` 成发布 zip |
| `publish.ps1` | 一键发布 GitHub：建仓库 → 打 `dsh-plugin` 主题标签 → 推代码 → 创建 Release 并上传 zip |
| `README.md` | 面向用户的使用/打包/配置文档 |
| `LICENSE` | MIT 许可 |
| `.gitignore` | 忽略 `node_modules/`、`release/`、`*.log` 等 |

模块调用关系（ASCII 图）：

```
┌─────────────────────────── Electron 壳（独立进程，Windows） ───────────────────────────┐
│  main.js（主进程, Node.js）                                                             │
│   ├─ 配置加载/日志/端口探测(net.connect)/服务拉起(cmd.exe)/停服(taskkill)/单实例锁        │
│   ├─ BrowserWindow（frame:false, contextIsolation:true, backgroundThrottling:false）     │
│   │    ├─ dom-ready → insertCSS + executeJavaScript(顶栏/注意力轮询/体验增强注入)        │
│   │    ├─ context-menu → Menu(复制/剪切/粘贴/撤销 + 发送到新对话)                        │
│   │    ├─ ipcMain 收: dsh:minimize / toggle-maximize / close / toggle-fullscreen        │
│   │    │              dsh:attention → flashFrame(true|false)                            │
│   │    └─ webContents.send 推: fullscreen-changed / maximized-changed / focus-changed    │
│   └─ loadURL ─────────► http://127.0.0.1:3080（DSH Web UI，内容与浏览器完全一致）          │
│                               ▲                                        │                │
│                    GET /dsh-attention（配套插件契约）              preload.js（bridge）    │
│                               │                                        │ window.dshShell │
│            DSH 宿主 webServer（dsh-attention-notifier）◄── 注入脚本每秒轮询 + 活动检测     │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

## 5. 关键实现细节

### 5.1 配置加载（`configPath` / `loadConfig` / `resolveStartCwd`）

- **configPath()**：打包版（`app.isPackaged`）且 exe 旁存在 `config.json` → 读 exe 旁；
  否则读项目根（`npm start` 开发模式）。
- **loadConfig()** 默认值合并：`port=3080`；`start.command` 缺失 → `'pnpm dsh web'`；
  `window.overlay=true`、`window.overlayHeight=36`；JSON 解析失败 → 用默认值并在日志记录。
- **resolveStartCwd()（DSH checkout 目录解析，按优先级）**：
  1. `config.start.cwd` 非空：绝对路径直接用；相对路径以 **config 所在目录** 为基准 `path.resolve`；
  2. 环境变量 `DSH_CHECKOUT`（用 `isDshCheckout` 验证 `dir/package.json` 存在）；
  3. 相对 config 目录探测常见并排布局：`deepseek-harness`、`../deepseek-harness`、
     `../../deepseek-harness`、`../../../deepseek-harness`（开发态 / win-unpacked / 安装目录都能命中）。
  全部落空 → 返回 `null` → boot 弹错误页引导编辑 config.json。

### 5.2 日志

`appendFileSync` 追加，行格式 `[ISO时间] 内容`；打包版写 exe 旁 `dsh-shell.log`，开发版写项目根；
写入失败静默忽略。服务 stdout/stderr 按 `\n` 拆行（去尾部 `\r`、残留 buffer 在 `end` 时补打），
带 `[out]` / `[err]` 前缀。

### 5.3 服务探测与启动 / 停止

- **portInUse()**：`net.connect({host:'127.0.0.1', port})`，600ms 超时 destroy → false；
  connect → true；error → false。
- **waitForServer()**：`START_TIMEOUT_MS=120000` 截止 + `POLL_INTERVAL_MS=700` 轮询；
  三态 `ready` / `exited` / `timeout`。`exited` 判定：startServer 的 `onExit(code)` 回调把
  `serverExited` 置为 code，轮询里 `getServerExited()` 为 truthy 即判 exited。
- **startServer()**：`spawn('cmd.exe', ['/d','/s','/c', cfg.start.command], { cwd, windowsHide:true,
  stdio:['ignore','pipe','pipe'], env: process.env 拷贝 })`。child `error` 事件（如 cwd 无效/命令
  不存在）→ reject；`exit` → 置空 `spawnedProc` 并调 `onExit`。
- **stopServer()**：`spawnedProc` 非空才停（即**只停自己拉起的服务**）；`execFile('taskkill',
  ['/pid', pid, '/T', '/F'])` 杀整棵进程树；`stopping` 标志防重入；挂载在 `window-all-closed`
  与 `before-quit`。

### 5.4 boot 主流程

`portInUse()` → 已有服务 → 直接 `win.loadURL(BASE_URL)`；没有服务 → 校验 cwd（空则错误页）→
`startServer` → `waitForServer`：`exited` → 错误页"服务进程启动后立即退出（exit code N）"；
`timeout` → `stopServer()` + 错误页"等待服务就绪超时（120 秒）"；整体 catch → 错误页 + 日志。

### 5.5 启动页与错误页

`data:text/html;charset=utf-8` + `encodeURIComponent`；`escapeHtml` 转义用户可见文本防注入；
自绘 spinner（CSS 旋转动画）；错误页红标题 + `<pre>` 详情 + 右上角关闭按钮（`window.close()`）。
窗口 `show:false`，`ready-to-show` 才 `win.show()`（避免白屏闪烁）。初始即加载启动画面 data: 页，
服务就绪后再 `loadURL(BASE_URL)`。

### 5.6 顶栏注入（overlay 模式）

- **触发**：`dom-ready` 时，仅当 `win.webContents.getURL()` 以 `BASE_URL` 开头（避免注入到
  启动 data: 页或外链），且 `overlay` 开启。
- **CSS**：
  - `body{box-sizing:border-box;padding-top:<overlayHeight>px !important}` 给页面顶部让位
    （利用 DSH 根布局 `html/body/#root{height:100%}` + border-box，padding 不裁内容）；
  - `#dsh-shell-bar`：fixed 顶部、高 **36px（写死，未与 overlayHeight 联动，见第 7 节）**、
    `z-index: 2147483000`（压在 DSH 面板之上）、`-webkit-app-region: drag` 整条可拖、
    按钮 `no-drag`；
  - 按钮颜色用 `var(--dsw-alias-label-secondary, #8b98a5)` 跟随 DSH 主题；hover 用
    `color-mix(in srgb, currentColor 12%, transparent)`；关闭按钮 hover 红底 `#c42b1c` 白字。
- **JS（OVERLAY_JS，IIFE）**：
  - 幂等守卫 `if (document.getElementById('dsh-shell-bar')) return`；`window.dshShell` 不存在则
    return（overlay:false 时 preload 未加载，天然不注入）；
  - 建四按钮（内联 SVG，`stroke="currentColor"`）：全屏（进入/退出两套图标按状态切换）/最小化/
    最大化（含还原图标）/关闭；点击 → `api.*` → ipcMain → 主进程窗口操作；
  - `api.onFullscreenChange` / `api.onMaximizeChange` 回调切图标、`title`、`aria-label`。
- **全屏 UI**：`body.dsh-fs` 加 `padding-top:0 !important`（全屏内容顶到屏幕最顶）；bar
  `opacity:0; pointer-events:none` 淡出；`mousemove` 且 `clientY ≤ 60` → 加 `dsb-shown` 淡入，
  离开后 450ms（hideTimer 防抖）淡出。

### 5.7 注意力提醒（页面侧，注入脚本内）

- **前置**：`api.setAttention` 存在才启动 `setInterval(pollAttention, 1000)` + 立即执行一次。
- **状态变量**：`attnKind`（'none' | 'intervention' | 'done'，**变化才上报**）；`attnFocused`
  （`api.onFocusChange` 回填，聚焦时重置 `attnLastActivity`）；`attnBaseline`（首读标记）；
  `attnLastCompletedId`；`attnPendingCompletion{id, at}`；`attnAckedCompletion`；`attnLastActivity`；
  `attnIdleMs = 8000`。
- **活动检测**：document 级 `mousemove / mousedown / keydown / wheel / touchstart`（passive）
  刷新 `attnLastActivity`。
- **轮询逻辑（pollAttention，伪代码级）**：
  1. `fetch('/dsh-attention', {cache:'no-store'})`；非 200 或响应非对象 → 忽略；
  2. **首读**：记 `attnBaseline=true`，`attnLastCompletedId = state.completedId`，return
     （旧完成不误闪）；
  3. `away = !attnFocused || (Date.now() - attnLastActivity > 8000)`；
  4. `state.intervention`：away → `setAttn('intervention')`，否则 `setAttn('none')`，return；
  5. `state.completedId !== attnLastCompletedId` → 更新 `attnLastCompletedId` 并记
     `attnPendingCompletion = {id, at: state.completedAt}`；
  6. 若 `state.running` → 清 `attnPendingCompletion`（一轮还在跑，不算完成）；
  7. pending 存在且 `age ≥ 2500`（完成确认窗口）：`!away` → 记 acked 清 pending（正看着 = 已看到，
     不闪）；`away` 且该 id 未 ack 过 → `setAttn('done')` + 记 acked 清 pending；
  8. 收尾：`attnKind ∈ {intervention, done}` 且 `!away` → `setAttn('none')`（回到对话立即熄灭）。
- **上报**：`setAttn(kind)` 仅在 kind 变化时调 `api.setAttention({kind})`。

### 5.8 注意力提醒（主进程侧）

- **ipc `dsh:attention`**：校验 `kind ∈ {none, intervention, done}`（否则忽略），存 `attention.kind`
  并 `renderAttention()`。
- **renderAttention()**：先 `stopAttentionFx()`（`flashFrame(false)` + `setOverlayIcon(null,'')`）；
  kind 为 'none' 返回；否则 log + `win.flashFrame(true)`。代码注释说明：**完全信任页面的"你不在"
  判定，不再按窗口聚焦自行拦截**——旧拦截会把"聚焦发呆"场景吞掉（历史演进）。
- **窗口 focus** → send `dsh:focus-changed true` + 若 kind 非 none 再 renderAttention（页面收到
  焦点后 ≤1s 内会因 `!away` 上报 none 熄灭）；**blur** → send false。
- **flashFrame 机制**：Windows 上 `flashFrame(true)` 是有限次数闪烁，闪几轮后任务栏按钮停留
  高亮态（微信"常驻淡红"），直到 `flashFrame(false)` / 窗口激活才恢复。

### 5.9 快捷键与窗口状态

`before-input-event` 拦截 keyDown：`F11` → preventDefault + `setFullScreen(!isFullScreen())`；
`Escape` 且全屏 → 退出全屏。`enter/leave-full-screen`、`maximize/unmaximize` 事件 →
`webContents.send` 推送页面（顶栏据此切图标/淡入淡出）。

### 5.10 导航与外链

`will-navigate`：`data:` 前缀放行；非 `BASE_URL` 前缀 → preventDefault，http(s) 走
`shell.openExternal`。`setWindowOpenHandler`：`BASE_URL` → allow；http(s) → openExternal + deny。
`did-fail-load`：忽略非主 frame 与 code -3（ERR_ABORTED），否则错误页。

### 5.11 生命周期与单实例

`app.requestSingleInstanceLock()` 失败 → `app.quit()`；`second-instance` → 最小化则 restore，
再 show + focus。`app.setAppUserModelId('local.dsh.shell')`（Windows 任务栏行为所需）。
`window-all-closed` → stopServer + quit；`before-quit` → stopServer；`closed` → **仅置空
`win=null`**（'closed' 触发时窗口已销毁，不能再调 `flashFrame`/`setOverlayIcon` 等窗口方法，
否则抛 `Object has been destroyed`）。`stopAttentionFx` / `renderAttention` 及全部 ipc 窗口操作
均带 `win.isDestroyed()` 守卫；另注册 `process.on('uncaughtException')` 兜底——异常写入
`dsh-shell.log` 而不弹 Electron 崩溃对话框。

### 5.12 打包配置

electron-builder：`files` 仅 `main.js / preload.js / assets/**/*`；`extraFiles` 把 `config.json`
放 exe 旁；win target = dir + nsis（x64）；nsis：`oneClick:false`、`perMachine:false`、
`allowToChangeInstallationDirectory:true`、`createDesktopShortcut`、`createStartMenuShortcut`、
`shortcutName: 'DSH Desktop'`、`deleteAppDataOnUninstall:false`。窗口 icon 与打包 icon 均
`assets/icon.png`。窗口默认参数：宽 1320、高 860、最小宽 940、最小高 620、`title: 'DeepSeek Harness'`、
`autoHideMenuBar:true`、`backgroundColor:'#101418'`、`show:false`（`ready-to-show` 才显示）。

### 5.13 发布脚本

`publish.ps1`：确认账号 → 建仓库（422 复用）→ topics 打 `dsh-plugin` → `git push`
（remote URL 内嵌一次性 `x-access-token`，推完 `set-url` 擦除）→ 创建 Release（422 复用既有 tag）
→ `curl.exe --ssl-no-revoke` 上传 zip（规避 Windows schannel 证书吊销检查误报）。发布完提示删除 Token。
`make-zip.ps1`：复制 `release\win-unpacked` → 临时目录 → `Compress-Archive` →
`release\DSH-Desktop-<ver>-win-x64.zip`。

### 5.14 与 DSH 版本的耦合点 / 漂移风险

- CSS 变量 `--dsw-alias-label-secondary`（含兜底色 `#8b98a5`，缺失仍可用但颜色固定）；
- 根布局假设 `html/body/#root{height:100%}` + border-box，`padding-top` 让位方案依赖它；
- `GET /dsh-attention` 契约字段（`intervention` / `running` / `completedId` / `completedAt`）
  由配套插件定义，DSH 本身不提供该端点；
- 默认端口 3080 与启动命令 `pnpm dsh web`（DSH 启动方式变化时需改 config.json）；
- `backgroundThrottling:false` 仅在 overlay 模式设置（注意力轮询需要）；
- **体验增强依赖 DSH 页面 DOM 锚点**（实测）：`placeholder="给智能体发消息"`（输入框）、
  `aria-label="新建会话"`、`aria-label="复制"`（消息操作键）与哈希类名（`gdEzaW_userRow` /
  `gdEzaW_bubble` / `p-xYUq_actions` / `p-xYUq_action` / `osXY9a_root` 等）；DSH 改 UI 或构建哈希
  变化时对应功能**静默失效**（渐进增强，不报错，需人工对齐锚点）。

### 5.15 智能右键菜单（主进程）

- **触发**：`win.webContents.on('context-menu')`，仅当 `getURL()` 以 `BASE_URL` 开头（启动/错误
  data: 页不弹）。
- **菜单构建 `buildContextMenu(params)`**：依据 `params.selectionText`（选中文字）、
  `params.isEditable`（是否点在可编辑区）、`params.editFlags`（canCut / canCopy / canPaste /
  canUndo）动态生成模板：
  - 可编辑区 → `粘贴 / 剪切 / 复制 / 撤销`（各按 editFlags 可用性取舍）；
  - 非可编辑且有选中 → `复制`；
  - 有选中文字 → 追加分隔线 + `发送到新对话`。
- 复制/剪切/粘贴/撤销用 `Menu` 内置 `role`（Chromium 原生执行，与系统行为一致）；无可用项或
  无选中时返回 null（不弹菜单）；`menu.popup({ window })` 弹出。
- **「发送到新对话」**：`executeJavaScript("window.__dshShellNewSession(<JSON 字符串>)")` 调页面
  注入函数（见 5.16），失败仅 log，不弹错。

### 5.16 消息重新编辑与发送到新对话（页面注入 OVERLAY_JS）

- **DOM 锚点（实测，DSH 当前构建）**：输入框 `textarea[placeholder="给智能体发消息"]`（React 受控
  组件）；「新建会话」`[aria-label="新建会话"]`；用户消息行 `.gdEzaW_userRow`，正文
  `.gdEzaW_bubble`，操作条 `.p-xYUq_actions`，复制键 `button[aria-label="复制"]`；助手消息行
  `.osXY9a_root`。锚点优先用 aria-label / placeholder 等稳定属性，哈希类名仅作辅助。
- **fillComposer(text)**：受控 textarea 必须用**原生 value setter**
  （`Object.getOwnPropertyDescriptor(HTMLTextAreaElement.prototype,'value').set`）写入，再派发
  `input` / `change` 事件，不能只设 `.value`；随后聚焦 + 光标置尾。
- **newSessionThenFill(text)**：点「新建会话」→ 轮询（120ms × 最多 40 次）等新输入框就绪
  （值为空）→ `fillComposer(text)`；暴露为 `window.__dshShellNewSession` 供主进程右键菜单调用。
- **installEditButtons()**：遍历 `.gdEzaW_userRow`，在操作条内复制键**前**插入「重新编辑」按钮
  （复用 `.p-xYUq_action` 类跟随 DSH 操作键样式/主题，`aria-label="重新编辑"` 作幂等标记）；
  点击 → 读 `.gdEzaW_bubble` 正文 → `fillComposer`（原消息保留）。
- **动态渲染**：防抖（400ms）`MutationObserver` 监听 body 子树，为新消息增量安装按钮。
- **幂等与降级**：已带 `.dsh-edit-btn` 的行跳过；锚点缺失 → 对应功能静默禁用，不影响壳本体。

## 6. 集成与安装

从拿到本描述到壳生效的完整步骤：

1. **复刻**：把本文件全文粘贴给 AI，要求按"**Electron 桌面应用（Windows）**"形态产出
   （不是 Cordis 插件）：`main.js`、`preload.js`、`config.json`、`package.json`
   （含 electron ^42、electron-builder ^26 与第 5.12 节 build 配置）、`assets/icon.png`；
   `make-zip.ps1`、`publish.ps1` 为可选的发布工具。
2. **安装依赖**：在产物目录执行 `npm install`。
3. **开发试跑**：`npm start`（读项目根 config.json）；**日常使用**：`npm run dist:dir` →
   `release\win-unpacked\DSH Desktop.exe`（绿色版，双击即用），或 `npm run dist` 出 NSIS 安装器。
4. **配置**：默认 `config.json` 即可。DSH checkout 自动探测失败时，填 `start.cwd` 绝对路径
   （或相对 exe 的路径），或设置环境变量 `DSH_CHECKOUT`；`pnpm` 需在 PATH。
5. **验证基础功能**：停掉所有占用 3080 的实例后双击 exe → 应自动拉起 `pnpm dsh web` 并开窗；
   再双击只聚焦不新开；关窗后 `tasklist` 确认无残留 DSH 进程；浏览器打开 `127.0.0.1:3080`
   内容与壳内完全一致。
6. **验证顶栏**：拖动窗口、四按钮可用；换皮肤/切深浅色按钮颜色跟随；全屏 → 按钮淡出 → 鼠标
   靠顶淡入 → Esc/F11 退出；`overlay:false` 回退原生标题栏。
7. **（可选）任务栏提醒**：按 dsh-attention-notifier 的 README 把插件装进 DSH（宿主层
   `cordis.patch.yml` 一行即可，所有预设/会话生效），重启 DSH 后验证
   `Invoke-WebRequest http://127.0.0.1:3080/dsh-attention` 返回 JSON；然后失焦窗口跑一轮工作，
   观察任务栏闪烁；回到对话应立即熄灭。
8. **（推荐）验证体验增强**：选中一段文字右键 → 菜单含 `复制` + `发送到新对话`，点击后者 →
   当前工作区新建会话且输入框已填入选中文字（未自动发送）；点在输入框右键 → 出现 `粘贴` 等；
   任意用户消息复制键旁有「重新编辑」按钮，点击 → 正文召回输入框；打断任务后直接改发。
9. **排查**：启动异常看 exe 旁 `dsh-shell.log`（开发模式看项目根）。

## 7. 已知边界与注意事项

- **Windows 专属**：依赖 `cmd.exe`、`taskkill`、Windows 任务栏 `flashFrame`、NSIS；macOS/Linux 未支持。
- **不含任何 DSH 代码**：DSH 更新照常（checkout 里 git pull 等），壳无需重建；但 DSH 改动
  CSS 变量/根布局/启动命令时壳的注入与启动逻辑可能漂移（见 5.14）。
- **端口冲突即复用是特性**：想让壳"自己启动服务"，需先停掉所有占用 3080 的 DSH 实例。
- **任务栏提醒强依赖配套插件**：没装 dsh-attention-notifier（或同契约实现）时任务栏不闪，
  壳不探测 DSH 页面状态——这是"轻量壳"定位的刻意取舍。
- **顶栏高度写死 36px** 在 OVERLAY_CSS 中，与 `overlayHeight` 配置未联动：改配置需同步改代码
  （待确认：可能是简化取舍）。
- **`assets/icon-attn.png` 未被代码引用**（待确认：疑似任务栏叠加图标预留，现实现用 flashFrame
  且恒清除 overlay icon）。
- **绿色版不自动建快捷方式**（README 提供手动步骤）；移动项目文件夹后快捷方式失效需重建。
- **未签名 NSIS 安装包**首次运行有 Windows SmartScreen 提示（"更多信息 → 仍要运行"）。
- **config.json 损坏** → 回退默认值（可能连到错误的端口/命令），日志中有记录。
- **安全**：contextIsolation + 无 nodeIntegration，preload 仅暴露窗口控制；外链一律系统浏览器。
  注意注入脚本以 `executeJavaScript` 运行在 DSH 页面上下文，页面若被 XSS，壳的窗口控制可被调用
  （风险随 DSH 本身，壳未额外隔离）。
- **不自动重试**：服务启动失败/超时只给错误页与日志，需用户手动关闭重开。
- **完成提醒的确认窗口**：完成事件发生 2.5 秒内若新一轮已开始（`running` 为 true），该次完成
  的闪烁会被丢弃（页面侧实现细节，属边界行为）。
- **体验增强为 DOM 层增强，随 DSH 前端更新漂移**（见 5.14）：锚点失效时功能静默禁用，不报错、
  不影响壳本体；需人工对齐锚点。
- **右键菜单仅壳内生效**：它是 Electron 原生菜单，浏览器直接打开 `127.0.0.1:3080` 时没有此菜单。
- **重新编辑召回的是渲染后文本**：消息正文含 Markdown（代码块/加粗）时从 DOM 召回会丢失格式；
  纯文本 prompt 无影响。

## 8. 复刻验收标准（给复刻 AI 的检查单）

- [ ] 双击 exe（或 `npm start`）在无服务时自动拉起 `pnpm dsh web` 并打开 3080 页面；页面内容
      与浏览器打开 `http://127.0.0.1:3080` **完全一致**（不修改 DSH 页面结构）
- [ ] 3080 已占用时直接开窗，关窗不杀外部服务；壳自己拉起的服务关窗即 taskkill 整树、无残留进程
- [ ] 重复双击只聚焦已有窗口（单实例）
- [ ] 无边框 + 自绘顶栏：可拖动；全屏/最小化/最大化/关闭四按钮可用；按钮颜色随 DSH 主题
      （换皮肤/切深浅色）变化
- [ ] 全屏：按钮组淡出，鼠标靠顶（≤60px）淡入，Esc/F11 退出；全屏时按钮可关闭窗口
- [ ] `window.overlay:false` 回退原生标题栏，F11/Esc 全屏仍可用
- [ ] 错误路径：无 checkout / 命令失败 / 服务提前退出 / 120s 超时 → 错误页含 config 位置、
      排查命令与日志路径；`dsh-shell.log` 有对应记录
- [ ] 装 dsh-attention-notifier 后：失焦窗口 + 介入/完成 → 任务栏闪烁（有限次后常驻淡红）；
      聚焦或窗口内操作 → 立即熄灭；正看着窗口时完成不闪
- [ ] 右键：选中文字 → 菜单含 `复制` + `发送到新对话`；点击 → 当前工作区新建会话且输入框已
      填入选中文字（未自动发送）
- [ ] 右键输入框（无选中）→ 菜单含 `粘贴 / 剪切 / 复制 / 撤销`
- [ ] 每条用户消息复制键旁有「重新编辑」按钮；点击 → 正文召回输入框、聚焦、原消息保留
- [ ] 打断任务后对最后一条用户消息点「重新编辑」→ 输入框出现原 prompt，可直接改发
- [ ] 注入幂等：页面刷新/HMR 后编辑按钮不重复出现
- [ ] 外链在系统默认浏览器打开，壳内不导航、不新开窗口
- [ ] 结构/集成点与描述一致：main/preload/config 职责划分、`GET /dsh-attention` 契约字段、
      config 解析优先级（start.cwd → DSH_CHECKOUT → 并排自动探测）与默认值
