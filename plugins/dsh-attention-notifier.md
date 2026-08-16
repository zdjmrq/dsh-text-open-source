# 【文字开源】dsh-attention-notifier — DSH 的"需要你介入 / 一轮完成"注意力提醒判定端

> **这是什么**：本文件是「文字开源」描述，不是代码。把本文件全文粘贴给你 DSH 里的 AI 会话，
> AI 即可为你复刻出功能相同的插件；你也可以阅读本文件理解其原理，并自行微调需求。
>
> **对应源码**：https://github.com/zdjmrq/dsh-attention-notifier （描述基于 commit
> `719d8b38f1eb1689b87ff3509f754f6591b51cf1`）｜许可证：MIT（Copyright (c) 2026 zdjmrq）
>
> **最近更新**：2026-08-16

## 1. 插件概述

本插件给 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（DSH）增加"微信式"任务栏注意力提醒的**判定端**：它监听每个会话的审批请求、向用户的提问和工作状态，把"需要你介入"与"一轮工作完成"两类信号聚合到一个 JSON 端点，供桌面壳轮询呈现。插件**只做判定，不碰任何 UI，不发布任何服务**；呈现（任务栏闪烁等）交给配套的 [dsh-shell](https://github.com/zdjmrq/dsh-shell) 桌面壳——一个负责"判"、一个负责"显"，是成对设计，但拆开也各自可用。适用场景：后台同时跑多个会话时不必一直盯屏，只有真正需要人介入或一轮干完时才被提醒。它是一个**持久化**的 Cordis 插件（宿主半），以宿主组合层 patch 方式挂载，随 DSH 重启自动生效，一个实例服务所有预设的所有会话。

## 2. 功能规格

用户/呈现端可感知的行为清单（可验收）：

- **介入判定——审批请求挂起**：某根 agent 作用域内发出 `approval/request` 事件（waterfall）→ 插件记录该请求起始时刻、`stats.approvals +1`、打印日志，然后**继续调用 `next()` 放行，不拦截、不代答** → 从请求发出起持续 ≥1000ms 仍未结束（`next()` 返回的 Promise 尚未 settle），则端点 `intervention` 变为 `true`；请求 settle（无论成败）后立即移除该计时，若无其他挂起项则 `intervention` 回落为 `false`。机器秒答的审批（远小于 1 秒）不会误报。
- **介入判定——向用户提问挂起**：`tools/execute` 事件中 `exec.name` 含 `ask_user_question`（子串匹配；不含则该事件被直接放行、完全不计数）→ 记录起始时刻、`stats.questions +1`、打印日志，调用 `next()` 放行 → 执行持续 ≥1000ms 则 `intervention=true`；工具调用 settle 后移除计时。用于捕捉 `ask_user_question` 这类等待人回答的挂起。
- **一轮完成判定**：根 agent 的 `agent/status` 事件 `status === 'running'` → 置该会话 running；`status === 'idle'` → **若此前 running 为 true**，则 `completedId +1`、`completedAt = now`、`stats.completions +1` 并打印日志，随后 running 置 false（idle 时本来就不是 running——如会话刚恢复——不计数，避免重复计数）。端点据此暴露"有新完成"。
- **运行状态聚合**：任一被跟踪根 agent 处于 running → 端点 `running=true`；全部 idle → `false`。
- **状态端点输出**：`GET /dsh-attention`（挂在宿主 `webServer` 上，精确路径）返回 JSON：
  - `intervention`（bool）：是否有审批/提问挂起 ≥1 秒；
  - `running`（bool）：是否有一轮工作正在进行；
  - `completedId`（int）：完成轮次自增计数（DSH 重启后归零；多会话聚合时取 `completedAt` 最新者的计数）；
  - `completedAt`（int）：最近一次完成的 epoch 毫秒时间；
  - `stats`（object）：自诊断计数 `{ sessions, approvals, questions, completions }`，跨会话求和。
  响应头 `Content-Type: application/json; charset=utf-8`、`Cache-Control: no-store`；聚合抛错时返回 500 `{"error":"aggregate failed"}`。
- **多会话聚合**：一个实例覆盖所有预设的所有会话——每个根 agent 一份独立状态，`intervention`/`running` 取或，`stats` 求和，`completedId`/`completedAt` 取最新完成者。
- **会话生命周期自动管理**：每秒轮询 `agents.roots()`：新出现的根 agent（新会话/恢复会话都会产生新 agent 对象）自动挂上监听；已销毁的（`agents.get(id) !== agent`）自动解除监听并清理状态，无需重启。
- **插件生命周期纪律**：插件卸载（或 DSH 停止）时统一解除全部 agent 监听器与路由注册，不留任何跨插件生命周期的注册。
- **明确不做的事**：不闪烁、不弹窗、不发声；不注入页面、无 Client 半、无 UI；不发布 Service/Event/Tool；不代答审批、不拦截工具——所有 waterfall 一律 `next()` 并返回其结果，只观察不干预。

## 3. 技术路线

- **插件形态**：单文件 ESM（`attention-plugin.mjs`，222 行）、**仅 Host 半** 的 Cordis 插件；无 Client 半。模块导出固定三件：`export const name = 'attention-notifier'`、`export const inject = ['webServer', 'agents', 'timer']`、`export function apply(ctx)`。
- **加载与集成方式**：
  - **推荐——宿主组合层 patch**：把 mjs 放入 `~/.dsh/plugins/`，在 `~/.dsh/profiles/web/cordis.patch.yml` 用 `- insert:` 追加一行组合（见第 6 节）。DSH 启动即自动加载，**所有预设、所有会话自动生效**，无需在每个预设里加行、无任何"启用"操作。
  - **备选——按预设安装**：把 mjs 复制进某预设目录，在 `agent.cordis.yml` 末尾追加一行（见第 6 节），仅该预设生效（端点仍挂在宿主 `webServer` 上）。
- **依赖的核心 Service**（`inject` 声明为硬依赖，加载器会等这些服务就绪后才激活本插件，消除启动时序竞态；运行期用 `ctx.get(...)` 防御性读取）：
  - `webServer` — `register({ kind: 'exact', path: '/dsh-attention', handler(req, res) })`，返回路由拆解函数；用于挂状态端点。
  - `agents` — `roots()` 取全部根 agent；`get(id)` 按 id 查存活；用于补线新会话与清理已销毁会话。
  - `timer` — `interval(fn, 1000)` 每秒心跳；驱动补线轮询与路由注册重试。
- **依赖的核心 Event**（全部注册在**根 agent 自己的 ctx** 上，`target.on(...)`）：
  - `agent/status`（payload.status 为 `'running'`/`'idle'`）— 完成判定；
  - `approval/request`（`(req, next)` waterfall）— 审批介入判定；
  - `tools/execute`（`(exec, next)` waterfall，匹配 `exec.name` 含 `ask_user_question`）— 提问介入判定。
- **作用域原理（本插件最关键的设计点）**：agent 级事件沿作用域链**向上**投递、绝不向下；监听器必须挂在 agent 自己的作用域上。插件经 `agents.roots()` 取得根 agent 的 ctx 并在其上注册监听，天然只收到该根 agent 的事件——受管子 agent（subagent）是下一级作用域，其事件不会冒泡到根，故**子代理事件不会误报**。
- **关键外部依赖**：无任何 npm 运行时依赖；只消费 DSH 宿主服务；不依赖桌面壳（壳只是可选消费者）。呈现侧提到的 Electron 能力（`win.flashFrame` 等）属于 dsh-shell，不在本插件内。
- **与其他 dsh-* 仓库的配套关系**：与 [dsh-shell](https://github.com/zdjmrq/dsh-shell) 成对——本插件负责"判"（状态聚合到端点），dsh-shell 负责"显"（注入页面的轮询器每秒读端点，结合窗口焦点与活动检测判定"你不在"，经 preload 桥上报主进程触发任务栏闪烁）。壳侧内部细节（桥名 `window.dshShell.setAttention`、`flashFrame` 行为等）以 dsh-shell 仓库为准，本仓库未含其代码（**待确认**：以 dsh-shell 实际源码核实）。不装壳时插件独立可用，任何自制的呈现端（浏览器脚本、桌面组件、他人打包的桌面壳）都能消费同一 JSON。

## 4. 结构设计

仓库文件布局：

| 路径 | 职责 |
| --- | --- |
| `attention-plugin.mjs` | 全部插件逻辑（单文件 222 行）：状态、事件监听、聚合、路由、生命周期 |
| `README.md` | 用户文档：原理、安装、端点契约、与 dsh-shell 配套、呈现端最小实现、验证方法 |
| `LICENSE` | MIT 许可证（c）2026 zdjmrq |

`attention-plugin.mjs` 内部模块划分（按出现顺序）：

```
顶层：常量 + 模块级状态
  HOSTAGE_MS = 1000                    // 审批/提问挂起多少毫秒后视为"需要介入"
  liveAgents: Map<Agent, {local, off}> // 已接线的根 agent → 本地状态 + 监听拆解函数
  wiredAgents: WeakSet                 // 防重复接线
  routeDispose / routeReady            // 路由注册的拆解函数与就绪标志

aggregate()                 // 遍历 liveAgents，聚合出端点 JSON
mark(list)                  // 计时条目入队，返回出队函数
observe(promise, done)      // 观察 waterfall next() 的结果，settle 后出队
registerListeners(target, local)  // 在 agent.ctx 上挂 3 个监听器，返回统一 off()
wireAgent(agent)            // 接线一个新根 agent（建 local、注册监听、入 map）
detachAgent(agent)          // 解线（调 off、删 map，容错）
sweep(agents)               // 清理已销毁的 agent
ensureRoute()               // 在 webServer 上注册路由（未就绪则跳过，由 tick 重试）
tick()                      // 每秒心跳：ensureRoute → sweep → 逐个 wireAgent
apply(ctx)                  // ctx.effect 注册 timer.interval(tick, 1000) + 立即 tick；返回清理函数
```

调用关系（ASCII）：

```
timer.interval ──► tick ──► ensureRoute ──► webServer.register('/dsh-attention')
                     │            └───────────► aggregate()（每次 HTTP 请求时调用）
                     ├──► sweep ──► detachAgent（清理已销毁会话）
                     └──► agents.roots() ──► wireAgent ──► registerListeners（挂在根 agent ctx）
                                                              │
              agent/status · approval/request · tools/execute（事件 → 写入 local 状态）
```

## 5. 关键实现细节

### 核心逻辑步骤化

**计时条目机制（mark / observe）**
1. `mark(list)` 生成 `{ since: Date.now() }` 压入对应数组（`approvals` 或 `questions`），返回一个"出队函数"；
2. `observe(promise, done)`：若 `next()` 的返回值是 thenable，则 `promise.then(done, done)`（无论成败，settle 即出队）；否则立即 `done()`（同步返回视为瞬时结束，不进入挂起判定）。

**intervention 判定（aggregate 内）**：对每个 agent，取其 `approvals` 与 `questions` 数组中最旧的 `since`；任一数组满足 `now - since >= HOSTAGE_MS(1000)` → `intervention = true`。注意判定发生在**端点被请求的时刻**，反映的是当下状态，不是持续推送。

**agent/status 处理**：`status === 'running'` → `local.running = true`；`status === 'idle'` → 若 `local.running` 为 true，则 `completedAt = now()`、`completedId += 1`、`stats.completions += 1`、打印 `[attention] turn completed id=...`，然后 `local.running = false`。用"前一刻是否为 running"防止 idle 重复计数。

**approval/request 处理（waterfall）**：`mark(approvals)`、`stats.approvals += 1`、打印日志；`const outcome = next()`；`observe(outcome, done)`；`return outcome`（把结果原样传回宿主）。若 `next()` 同步抛错：`done()`（立即出队）后 `throw err`（不吞异常）。

**tools/execute 处理（waterfall）**：先取 `exec.name`（非字符串视为空串）；`name !== 'ask_user_question' && name.indexOf('ask_user_question') < 0` 时直接 `return next()`——**不计数、不记录**；命中才 `mark(questions)`、`stats.questions += 1`、打印日志、放行并返回结果。

**会话补线与清理（tick / sweep / wireAgent）**：`sweep` 遍历 `liveAgents`，`try { alive = agents.get(agent.id) === agent } catch { alive = false }`，不活则 `detachAgent`（`try { entry.off() } catch {}` 后删 map——监听器可能已随 agent ctx 销毁被框架清理，二次清理报错直接忽略）。随后遍历 `agents.roots()` 逐个 `wireAgent`；`wiredAgents`（WeakSet）保证同一 agent 对象只接线一次。启动时 agent 尚未创建，靠每秒 tick 持续补线新出现的根 agent。

**路由注册重试（ensureRoute）**：`routeReady` 为 false 且 `ctx.get('webServer')` 存在且 `typeof register === 'function'` 时才注册，成功后置 `routeReady = true`，注册返回的拆解函数存入 `routeDispose`。README 明确指出的坑：**webServer 未就绪时路由注册会静默失败且无重试**，因此必须由每秒 tick 兜底重试。

**生命周期（apply）**：`ctx.effect(() => { timer.interval(tick, 1000); tick(); return () => { for (const agent of [...liveAgents.keys()]) detachAgent(agent); if (routeDispose) { routeDispose(); routeDispose = null } routeReady = false } })`——每个 agent 的拆解函数随条目保存，agent 销毁与插件卸载时统一执行。

### 重要边界处理与坑

- **机器秒答不误报**：`HOSTAGE_MS = 1000` 阈值 + settle 即出队，双保险。
- **子代理不误报**：监听器挂在根 agent 自身 ctx（作用域向上投递），subagent 是受管子 agent，其事件到不了根。
- **启动时序三重兜底**：inject 硬依赖（等 webServer/agents/timer 就绪才激活）+ 每秒 tick 补线 + 路由注册重试。
- **Windows 导入坑（必踩）**：组合行 `name` 必须写 `file:///C:/Users/<你的用户名>/.dsh/plugins/attention-plugin.mjs`；写 `C:/...` 形式会被 Node ESM 当作 scheme 为 `c:` 的 URL 而拒绝导入，DSH 直接启动失败。
- **Waterfall 纪律**：每个事件都必须调用 `next()` 并返回其结果（或抛错时 rethrow），只观察不拦截，保证宿主审批/工具行为与未装插件时一致。
- **同步/异步兼容**：`observe` 同时处理同步返回与 Promise 返回，出队逻辑统一。

### 配置项/常量

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `HOSTAGE_MS` | 1000 | 审批/提问挂起多少毫秒后视为"需要介入" |
| 轮询间隔 | 1000 | `timer.interval(tick, 1000)` 心跳周期（毫秒） |
| 端点路径 | `/dsh-attention` | 挂载于宿主 webServer 的精确路径 |

无环境变量、无配置文件、无其他可调项；阈值调整需改源码。（呈现侧的"8 秒闲置""1 秒轮询"等阈值在壳/呈现端一侧，不属于本插件。）

### 与 DSH 版本相关的耦合点（随版本漂移风险）

- `agents.roots()` / `agents.get(id)` 的签名与语义；
- `webServer.register({ kind: 'exact', path, handler })` 的路由注册契约（handler 收到 `(req, res)`，返回拆解函数）；
- `timer.interval(fn, ms)` 与 `ctx.effect` / `ctx.get` 的 Cordis 生命周期 API；
- 三个事件的事件名与 payload 形状（`agent/status` 的 `payload.status`；`approval/request` 与 `tools/execute` 的 `(payload, next)` waterfall 签名）；
- `ask_user_question` 工具名——用子串匹配（`indexOf`）而非精确匹配，对名称带前缀/后缀有一定宽容；这是否为刻意设计**待确认**。

以上 API 均以本仓库代码为准；README 与代码均未声明最低 DSH 版本（**待确认**）。宿主升级导致改名/改签名时需同步适配。

## 6. 集成与安装

从拿到本描述到插件生效的完整步骤：

1. **复刻代码**：把本文件全文粘贴给 DSH 里的 AI 会话，要求"按第 5 节规格产出单文件 `attention-plugin.mjs`（ESM、Cordis 插件、`name='attention-notifier'`、`inject=['webServer','agents','timer']`、`apply(ctx)` 实现第 2 节全部行为）"。产出应为单个 `.mjs` 文件，无其他依赖。
2. **放置文件**（推荐宿主层）：
   ```powershell
   New-Item -ItemType Directory -Path "$env:USERPROFILE\.dsh\plugins" -Force
   Copy-Item attention-plugin.mjs "$env:USERPROFILE\.dsh\plugins\"
   ```
3. **挂载组合**：编辑 `~/.dsh/profiles/web/cordis.patch.yml`，把内容 `[]` 替换为：
   ```yaml
   - insert:
       - id: attention-notifier
         name: 'file:///C:/Users/<你的用户名>/.dsh/plugins/attention-plugin.mjs'
   ```
   ⚠ Windows 下 `name` 必须用 `file:///` 绝对 URL（见第 5 节坑）。
4. **重启 DSH**（关壳重开，或重启 `pnpm dsh web`）——无其他启用步骤。
5. **验证**：
   ```powershell
   Invoke-WebRequest http://127.0.0.1:3080/dsh-attention
   # {"intervention":false,"running":false,"completedId":0,"completedAt":0,"stats":{...}}
   ```
   - `stats.sessions` 应为 1（本会话已挂载）；
   - 提问/审批挂起时 `intervention` 变为 `true`，`stats.questions/approvals` 递增；
   - 一轮工作结束后 `completedId` 递增；DSH 日志出现 `[attention] wired root agent ...`、`[attention] turn completed id=...`。
6. **接入呈现端**（可选，推荐 dsh-shell）：按 dsh-shell 的 README 打包运行 `DSH Desktop.exe`（或开发模式 `npm start`），确认其 `config.json` 的 `port` 与 DSH web 端口一致（默认都是 3080）——不一致时壳读不到端点，提醒不会出现。之后"需要介入/一轮完成"且你不在关注（窗口失焦/最小化，或聚焦但 >8 秒无操作）时任务栏闪烁；回到对话立即熄灭。自定义呈现端按第 2 节端点契约轮询即可（README 给出的最小实现范式：首读记基线 → 轮询 → 判"你不在" → flash/clear；平台呈现：Windows `win.flashFrame(true/false)`、macOS `app.dock.bounce('informational')`、Linux urgency hint）。

备选——**按预设安装**：把 `attention-plugin.mjs` 复制进该预设目录，在 `agent.cordis.yml` 末尾追加：
```yaml
- id: attention-notifier
  name: './attention-plugin.mjs'
```
仅该预设的会话生效；端点仍挂在宿主 webServer 上，该预设会话运行时即生效，同样兼容 dsh-shell。

## 7. 已知边界与注意事项

- **只判不显**：不装任何呈现端时，插件只是把状态聚合在端点里（浏览器直接打开 `http://127.0.0.1:3080/dsh-attention` 即可查看），**不会自行闪烁**——别期望"装上就有提醒"。
- **端点无鉴权**：`/dsh-attention` 挂在 DSH 的 web 服务上、无认证；README 称其"仅 DSH 本机 web 服务"。若 DSH 端口暴露到局域网/公网，该端点可被任何人读取（内容仅为聚合状态，无会话内容；是否只绑定 127.0.0.1 **待确认**）。
- **completedId 重启归零**：呈现端必须首读记基线，否则重启后的计数会被当"新完成"误闪（README 示例已处理）。
- **阈值硬编码**：`HOSTAGE_MS` 与轮询间隔写死在源码里，无配置入口；想调需改码。
- **计数为进程内自诊断**：`stats` 与 `completedId` 重启清零，不持久化。
- **依赖宿主 API 现状**：agents/webServer/timer 与三个事件的名字、签名以编写时的 DSH 为准，宿主升级可能漂移（见第 5 节）。
- **工具名子串匹配**：若未来 DSH 给提问工具改名（不再含 `ask_user_question`），提问介入将不再被识别（审批路径不受影响）。
- **仅跟踪根 agent**：running→idle 完成判定只对根 agent 生效；子代理不参与判定（也因此不误报）。
- **日志噪音**：每次审批/提问/完成都会打 `console.log`（前缀 `[attention]`），高频场景日志量会增加。
- **重复安装叠加**：若宿主层与某预设层同时安装，会有两个实例同时监听与计数（README 未讨论此叠加情形，行为**待确认**）。
- **多会话 completedId 语义**：聚合取 `completedAt` 最新者的计数，多个会话并发完成时计数反映"最近完成者"，而非总和——呈现端如需区分会话需自行扩展（当前端点不含会话维度信息）。

## 8. 复刻验收标准（给复刻 AI 的检查单）

- [ ] 产出单文件 ESM `attention-plugin.mjs`，导出 `name = 'attention-notifier'`、`inject = ['webServer', 'agents', 'timer']`、`apply(ctx)`；无 Client 半、无 UI、无 Tool、不发布任何 Service。
- [ ] 挂载后 `GET /dsh-attention` 返回 `{intervention, running, completedId, completedAt, stats}`，响应头含 `Cache-Control: no-store` 与 `application/json; charset=utf-8`；`stats.sessions` 等于当前根会话数。
- [ ] 触发一次审批请求并挂起 ≥1 秒：端点 `intervention` 变 `true`、`stats.approvals` 递增；请求 settle 后恢复 `false`，计数保留。
- [ ] 触发一次含 `ask_user_question` 的工具执行并挂起 ≥1 秒：`intervention` 变 `true`、`stats.questions` 递增；不含该子串的工具执行完全不影响计数；两者均未拦截宿主行为（审批可正常通过、工具可正常执行）。
- [ ] 完成一轮工作（running→idle）：`completedId` 递增 1、`completedAt` 更新、`stats.completions` 递增；连续 idle 不重复计数。
- [ ] 新开会话后无需重启，`stats.sessions` 自动 +1；销毁会话后自动回落（每秒轮询清理）。
- [ ] 插件卸载/DSH 停止后：无残留监听器、`/dsh-attention` 路由消失（404）、无报错。
- [ ] 所有 waterfall（`approval/request`、`tools/execute`）都调用了 `next()` 并返回其结果，宿主审批/工具行为与未装插件时一致。
- [ ] 宿主层安装时组合行使用 `file:///` 绝对 URL，DSH 能正常启动并生效；按预设安装时 `agent.cordis.yml` 用相对路径 `./attention-plugin.mjs` 亦正常。
- [ ] 与 dsh-shell 配套时，`config.json` 端口与 DSH web 端口一致后任务栏闪烁正常出现与熄灭。
