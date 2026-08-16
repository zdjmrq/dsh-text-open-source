# 【文字开源】dsh-usage-balance — DeepSeek Harness 侧边栏「用量 / 余额」插件

> **这是什么**：本文件是「文字开源」描述，不是代码。把本文件全文粘贴给你 DSH 里的 AI 会话，
> AI 即可为你复刻出功能相同的插件；你也可以阅读本文件理解其原理，并自行微调需求。
>
> **对应源码**：https://github.com/zdjmrq/dsh-usage-balance（描述基于 commit `3c64872352728801b82ebe2dc085da7ded91e6ec`）｜许可证：MIT
>
> **最近更新**：2026-08-16

## 1. 插件概述

dsh-usage-balance 是一个 DeepSeek Harness（DSH）Web 界面的侧边栏插件：在侧边栏底部「设置」按钮上方（`sidebar.footer.action` 槽位）渲染一行简洁的「用量 | 余额」标签，鼠标悬停时在右侧展开一张玻璃拟态详情卡，展示会话运行时长、Token 用量、按官方价格表估算的金额，以及调用 DeepSeek 官方接口获取的真实账户余额。它解决"想知道这轮对话烧了多少钱、账户还有多少余额"的观测需求，全程不离开界面、无需打开外部控制台。适用场景：日常使用 DSH Web 时随手查看费用与余额；其金额估算与内置统计（tokenUsage 投影）同源口径，余额复用 `llm-deepseek` 适配器的同一把 API Key，不额外要求用户配置密钥。该插件与 `dsh-cost-meter` 有渊源（余额接口的拼法与 key 回退链参考了它），二者功能有重叠但本插件侧重侧边栏内嵌 UI；与仓库内的「Cordis 面板」按钮共存（纵向堆叠、互不遮挡）。

## 2. 功能规格

所有行为以仓库代码（`src/index.js`、`client/client.js`）为准，以下按可验收行为穷举：

- **标签行（宽侧边栏）**：侧边栏底部「设置」按钮上方显示一行，内容为 16px 仪表盘 SVG 图标 + 14px 文案「用量 | 余额」；行高 34px、圆角 12px、图标与文字间距 8px，悬停时背景变为 `--dsw-alias-interactive-bg-hover`。文案跟随界面中英文（见下）。悬停整行触发详情卡。
- **窄栏（rail，侧边栏折叠为竖条时）**：显示 36×36 圆形按钮 + 18px 同款仪表盘图标（与设置 rail 同规格），无文字；鼠标悬停同样展开详情卡，定位逻辑与宽栏完全同一套；此时图标自带 `title` 提示（显示完整用量/余额摘要文本，悬停详情卡打开时取消 title 以免遮挡）。
- **详情卡（悬停展开）**：零延迟展开（onMouseEnter 即开），关闭有 220ms 缓冲（鼠标移出后 220ms 才隐藏，便于滑向卡片）；玻璃拟态样式（`backdrop-filter: blur(16px) saturate(1.3)` + 半透明叠层 + `--dsw-shadow-lv3`），`position: fixed`、`z-index: 35`、min-width 224px / max-width 320px。卡内固定 6 行 + 可选错误行 + 底部开关：
  1. 运行时长：格式 `hmmss`/`mss`/`s`（有小时才显示 h）；含义是"每轮对话实际工作时间之和"（turn/start → turn/end 区间），**进行中的轮每秒跳动补算到当前时刻**；
  2. 输入·未命中（`uncachedInputTokens + cacheWriteTokens`）；
  3. 缓存命中（`cacheReadTokens`）；
  4. 输出（`outputTokens`）；
  5. 金额：`≈¥` + 估算费用（<1 元显示 3 位小数，否则 2 位），该项 hover 时 title 显示按计价时段拆分的明细（基础价期/高峰/空闲 各自的 ≈¥ 金额，只列出 >0 的时段）；
  6. 计价时段：当前时刻所属计价层（基础价 / 高峰 / 空闲）；
  7. 余额：`¥`（或对应货币符号）+ 官方 total_balance，悬停 title 显示错误详情（失败时）；
  8. 错误行（仅余额失败时）：红色小字显示错误信息（HTTP 状态、curl 输出或接口 error.message）。
- **Token 行显示规则**：输入/缓存命中/输出三行的值统一按 `fmtTokens` 缩写（<1K 原样、<1M 一位小数 K、≥1M 一位小数 M）；若三者全为 0 则显示「—」。
- **门帘式滑杆开关（卡底）**：60×22px 椭圆按钮，`role="switch"` + `aria-checked`；深灰色门帘（fill）默认只占左侧 18px，点击后从左向右拉满（宽度过渡 200ms）＝「固定」模式（详情卡常驻显示），再点收回＝「悬停」模式；开关上左右两侧分别显示「悬停 / 固定」（zh）或「Hover / Pinned」（en）微标签；按钮 title 提示「点击后始终显示详情 / 点击后仅悬停显示详情」。固定状态仅存于客户端内存，不持久化。
- **视口防溢出**：卡片默认贴在触发行右侧（`left = rect.right`）；若右侧放不下（估计宽 264px + 12px 边距超视口）则翻转到左侧（`left = rect.left - estW`）；仍放不下则夹到 `margin=12` 内；上下同理（估计高 264px，`top = rect.top - 6` 起算）。
- **数据刷新**：进入页面立即请求一次 `/dsh-usage-balance/state`，之后每 60 秒轮询一次（`timer.interval`）；运行时长跳动由每 1 秒一次的时钟驱动。请求带当前会话 id（`?sessionId=<id>`，来自槽位注入的 `useSessions` 响应式选择器）；无会话时 runtime 行显示「无对话 / No session」，其余行按空数据处理。
- **余额展示状态机**：无快照时显示「…」；`status=ok` 显示货币符号 + total_balance；`unavailable + reason=no-key` 显示「未配置 key」；`unavailable`（其他）显示「不可用 / N/A」；`error` 显示「获取失败 / Failed」并在卡内追加错误详情行。
- **中英文双语**：文案表 `TEXTS.zh` / `TEXTS.en` 全覆盖（用量、余额、运行时长、输入·未命中、缓存命中、输出、金额、计价时段、基础价/高峰/空闲、无对话、未配置 key、不可用、获取失败、悬停/固定、开关提示等）；判定依据 `locale.getSnapshot().active` 是否以 `zh` 开头（`locale` 服务不可用时默认中文）；监听 `locale.subscribe` 实时切换。
- **主题跟随**：全部颜色使用 `--dsw-*` 主题变量（label/bg/interactive/separator/border/overlay/shadow/error 等），跟随 DSH 全局亮/暗主题。
- **与 Cordis 面板共存**：一条精确的 `:has()` CSS 规则——当脚部动作区出现带 `aria-haspopup="dialog"` 的兄弟节点（Cordis 面板触发器）时，把动作区容器改为 `flex-direction: column`，使本行与 Cordis 面板按钮纵向堆叠、互不冲突。
- **Host 端路由**：`GET /dsh-usage-balance/state?sessionId=<id>` 返回 `{ usage, balance, now }`（JSON，`Cache-Control: no-store`）；sessionId 缺省或会话不存在时 `usage.hasSession=false`（UI 显示「无对话」），余额仍照常尝试（此时 cwd 提示为空，走默认路径）。请求失败时返回 `{ error: <message> }`。

## 3. 技术路线

- **插件形态**：Cordis 插件，双半结构——Host 半（`src/index.js`，提供 JSON 路由）＋ Client 半（`client/client.js`，`module-loader` bundle 格式，零构建）。单包单插件（一个 `usage-balance` 插件实例，两半通过宿主各自加载）。
- **加载与集成方式**：以 **bundle 插件包** 形式加载（源码注释自称"bundle 层挂载"）：
  - `package.json` 中 `dsh.bundle.patch: "./cordis.patch.yml"` —— 该 YAML 是**补丁层**，按 profile 层级应用，内容是向宿主组合插入一行 `{ id: usage-balance, name: dsh-usage-balance }`，从而挂载 Host 半；
  - `dsh.client.platform: "web"` —— 声明 Client 半挂到 Web 平台，由 DSH 的 module-loader 以 `window.__ModuleLoader__.load({ id, factory })` 方式注入浏览器。
  - 安装命令 `dsh plugin --profile web add github:zdjmrq/dsh-usage-balance`，安装后重启 `dsh web` 生效（见第 6 节）。
- **依赖的核心 Service / Event（名称以仓库代码为准，均为 DSH 内部服务，随版本漂移，见 §5）**：
  - Host 半（仅 `webServer` 声明为硬依赖 `inject: ['webServer']`；其余**全部在请求时经 `ctx.get` 惰性获取**）：`webServer.register({kind:'exact', path, handler})` 注册路由；`sessions.get(sessionId)` 取会话；`sessionProjections.snapshot(session)` 取 `values.tokenUsage` 投影（与内置统计同源）；`settings.get('llm-deepseek')` 读 `apiKeyEnv`/`baseURL`；`credentials.resolve(envName)` 解析密钥（返回 `{value}`）；`process.env` 兜底；`launchEnvironment.get(envName)` 读启动环境快照（进程 env / 项目 .env / 用户 .env）；`subprocess.resolveExecutable(name)` + `subprocess.spawn(...)` 执行 curl；`sandboxPolicy.workspaceRoot` 作 curl 的默认 cwd。
  - Client 半（`inject: ['slots', 'timer']`；`locale` 经 `ctx.get` 可选）：`slots.inject('sidebar.footer.action', () => slots.register({name:'sidebar.footer.action', id:'usage-balance', order:-100}, renderFn))` 挂到侧边栏脚部动作区（设置按钮上方）；`timer.interval(cb, ms)`（1s 时钟、60s 轮询）、`timer.timeout(cb, ms)`（220ms 关闭缓冲）均返回 disposer；`locale.getSnapshot().active` / `locale.subscribe`；`React`（经 module-loader 的 `require('react')` 获取，仅用 `React.createElement`，无 JSX）。
  - 用到的**会话事件日志**（`session.events` 数组，append-only、`seq` 连续）：`turn/start`、`turn/end`（`data.turn` 为轮次号）、`request/header`（`data.header.config.model` 为模型名）、`assistant/chunk`（`data.chunk.type === 'usage'` 时携带 usage）、`assistant/message`（`data.usage` 存在时携带完整 usage）；事件对象含 `seq`、`time`、`type`、`data`。
- **关键外部依赖**：运行时仅需 Node.js ≥ 20 与 DSH（dshhub 兼容声明 `dsh >= 0.1.0-rc.5`）；余额查询需要能访问 `https://api.deepseek.com` 的网络与有效 DeepSeek API Key；本机有 `curl`/`curl.exe` 可执行文件作为 Node fetch 失败时的回退（可选，无 curl 时返回 `balance-unreachable`）。无 npm 运行时依赖（package.json 无 dependencies）。
- **与其他 dsh-\* 仓库的配套关系**：与 `dsh-cost-meter` 同族（余额端点拼接、key 解析回退链"参考 dsh-cost-meter"）；本仓库自身即完整插件，不依赖其他 dsh-\* 仓库运行。

## 4. 结构设计

| 路径 | 职责 |
| --- | --- |
| `src/index.js` | Host 半：注册 `GET /dsh-usage-balance/state` 路由；会话用量快照（tokenUsage 投影 + 增量分层计价 + 实际工作时间累计）；官方余额抓取（key 解析链 + fetch/curl 双方案 + 60s 缓存） |
| `client/client.js` | Client 半：`window.__ModuleLoader__.load` 的 module-loader bundle；`sidebar.footer.action` 槽位组件（标签行/rail/详情卡/滑杆开关）；`--dsw-*` 样式（style 标签注入）；中英文案；60s 轮询与 1s 跳动 |
| `cordis.patch.yml` | bundle 补丁层：向宿主组合 `insert` 一行 `{ id: usage-balance, name: dsh-usage-balance }`，挂载 Host 半 |
| `package.json` | 包清单：`main` 指向 Host 半、`exports` 暴露 `.` 与 `./client`；`dsh.bundle.patch` / `dsh.client.platform`；dshhub 元数据（surfaces `["host","web"]`、capability `route:/dsh-usage-balance/state`、兼容声明、网络权限声明） |
| `README.md` / `LICENSE` / `.gitignore` | 文档与元数据（MIT） |

模块调用关系：

```
浏览器(Client 半 client.js)
  │  slots.inject('sidebar.footer.action') 注册组件
  │  timer.interval 1s/60s 驱动
  ▼
fetch GET /dsh-usage-balance/state?sessionId=<id>   (每 60s + 首帧)
  │
  ▼
Host 半 src/index.js: webServer.register(exact 路由)
  ├─ sessions.get(sessionId) → buildUsage(session)
  │    ├─ sessionProjections.snapshot(session).values.tokenUsage  → 四桶 Token 展示值
  │    └─ sweepBilling(session, now) 增量扫描 session.events → 分层金额/工作时长/模型
  └─ fetchBalance(cwdHint)  →  key 解析链 → GET {baseURL}/user/balance
       (Node fetch → 失败回退 subprocess+curl；60s 全局缓存)
  └─ 返回 { usage, balance, now }  →  Client 渲染详情卡
```

## 5. 关键实现细节

### 5.1 增量分层计价（Host 半核心算法）

内存缓存 `billing: Map<sessionId, {lastSeq, model, last, buckets, cost, turnStart, workMs}>`（进程内，重启即清）。`buckets` 为三层桶 `{legacy, peak, offpeak}`，每层含 `{miss, hit, out, amount}`。

- **增量扫描**：每次请求只从 `b.lastSeq` 扫到日志尾部（`O(新增事件)`），扫完置 `lastSeq = total`。事件 `time` 取 `ev.time`，缺失时用当前时间。
- **全量重扫触发**：`b.lastSeq > total`（日志被截断）或 `b.lastSeq > 0` 且 `events[b.lastSeq-1].seq !== b.lastSeq`（seq 不连续，如会话被其他进程写）→ 重置该会话的记账对象从 seq 0 重扫（注释称毫秒级）。
- **计价层判定**：每个 usage 事件按**该事件的时间戳**定层——`ms < PRICE_SWITCH_MS`（2026-08-17T00:00 北京时间，即 `Date.UTC(2026,7,16,16)`）为 `legacy`；之后按北京时间小时，`9≤h<12 || 14≤h<18` 为 `peak`，否则 `offpeak`（空闲价 = 高峰价的一半，即表内 offpeak 恒为 peak 的一半）。`beijingHour(ms) = new Date(ms + 8*3600000).getUTCHours()`。
- **计价公式**：`miss = inputTokens + cacheWriteTokens`，`hit = cacheReadTokens`，`out = outputTokens`；`amount = (miss*P.miss + hit*P.hit + out*P.output) / 1_000_000`（单价为 元/百万 tokens）。模型价格查不到时回退 `FALLBACK_MODEL = 'deepseek-v4-pro'`。
- **last-wins 去重（与内置 token-meter 投影一致）**：`b.last` 记录上一个已处理的 usage 事件 `{turn, step, tier, miss, hit, out, amount}`；若新事件与它 `turn` 与 `step` 都相同（usage chunk 后跟 message 的完整 usage，属同一 turn/step 的"后到样本替换先到样本"），先把旧样本从对应桶与 `cost` 中**减掉**再累加新样本。注意：`turn`/`step` 可能为 `undefined`（如无 turn 的事件），此时 `undefined === undefined`，连续同源事件也会走替换路径——按代码如此实现，复刻须保持一致。
- **模型跟踪**：`request/header` 事件更新 `b.model`（最新一次请求的模型，会话级单一值）；`buildUsage` 时若 `b.model` 为 null 再调 `detectModel(events)`（从事件尾部反向找最后一个带非空 `config.model` 的 `request/header`）。**坑**：所有 usage 事件按"扫描时刻的 `b.model`"计价——首条 `request/header` 之前的 usage 事件会以 null 模型计，落入 `priceFor` 的 `FALLBACK_MODEL`（v4-pro）单价；会话中途换模型时，早期事件也会按扫描时的最新模型计价。这是近似口径，非精确逐事件归因。
- **实际工作时间（workMs / running）**：`turn/start` 记 `turnStart.set(turn, time)`；`turn/end` 累加 `workMs += max(0, time - start)` 并删除该 turn；`buildUsage` 时对仍在 `turnStart` 中的进行中轮补算 `now - start`；`running = turnStart.size > 0`。UI 每秒用 `snap.now` 与本地时钟差值做线性外推，实现"每秒跳动"。
- **金额展示口径**：`costCny` 为该会话累计估算金额（元）；`costBreakdown` 为三层桶金额；`currentTier = priceTierAt(Date.now())` 仅用于「计价时段」行。

### 5.2 余额抓取（Host 半）

- **端点拼接** `balanceEndpoint(baseURL)`：去尾部斜杠 → 空则默认 `https://api.deepseek.com` → 剥掉 `/v1` 等版本路径（`/\/v\d+$/i`）→ 拼 `/user/balance`。
- **Key 解析链**（与 llm-deepseek 适配器同链，逐级回退）：
  1. `settings.get('llm-deepseek')` 的 `apiKeyEnv`（默认 `'DEEPSEEK_API_KEY'`）与 `baseURL`（无则默认官方）；
  2. `credentials.resolve(apiKeyEnv)` → `{value}`；
  3. `process.env[apiKeyEnv]`；
  4. `launchEnvironment.get(apiKeyEnv)` → `{value}`（进程 env / 项目 .env / 用户 .env 快照）。
  全部失败 → `{status:'unavailable', reason:'no-key'}`，UI 显示「未配置 key」。
- **请求方案**：方案 1 `fetch(url, {headers:{authorization:'Bearer '+key}, signal:AbortSignal.timeout(15000)})`；`response.ok` 才解析并写入缓存，非 ok 返回 `{status:'error', message:'HTTP '+status}`。方案 2（fetch 抛错时）：`subprocess.resolveExecutable('curl')` 再试 `'curl.exe'`，找到后用 `subprocess.spawn({argv:[curl,'-sS','--max-time','15','-H','Authorization: Bearer '+key, url], cwd, stdio:{stdin:'ignore', stdout:{maxBytes:65536}, stderr:{maxBytes:16384}}, graceMs:5000})`；`cwd` 取会话 `header.cwd` → `sandboxPolicy.workspaceRoot` → `'C:\'`；JSON 解析失败时错误信息取 stderr（截 200 字符）或 stdout 前 200 字符；spawn 失败返回 `spawn-failed: <msg>`；无 curl 返回 `balance-unreachable`。**坑**：curl 方案是为绕开 Windows 上可能故障的 bash/WSL 而设（注释原文）。
- **响应解析** `parseResponse(text)`：`JSON.parse` → 取 `balance_infos[0]`；有 `currency`（字符串）且 `total_balance != null` → `{status:'ok', currency, totalBalance, grantedBalance, toppedUpBalance}`（金额统一 `String()` 化，UI 显示字符串）；`is_available === false` → `unavailable / not-available`；否则 `{status:'error', message: parsed.error.message | 'bad-response'}`。
- **缓存**：全局单例缓存（**所有会话共享**，非按会话），TTL `BALANCE_CACHE_MS = 60000`；成功解析的结果（含 API 返回的 error 类结果，不含 HTTP 非 2xx 与 spawn 失败）写入缓存。**坑**：无逐出、无并发去抖——60s 窗口内的并发请求会各自继续执行（但都命中同一缓存值），复刻时可保持简单实现。

### 5.3 Client 半关键实现

- **加载形态**：`window.__ModuleLoader__.load({id:'dsh-usage-balance', factory})`；`factory(require)` 是 CommonJS 风格，返回 `{apply, inject}`。**零构建**：该文件即最终产物格式，无需打包。所有代码为纯 JS + `React.createElement`，无 JSX/TS。
- **样式注入**：27 条 CSS 字符串拼接后经 `<style data-plugin-css="dsh-usage-balance/style">` 注入 `document.head`；注入前查重（同名 data 属性已存在则跳过）。样式全部走 `--dsw-*` 主题变量。
- **槽位注册**：`slots.inject('sidebar.footer.action', () => slots.register({name:'sidebar.footer.action', id:'usage-balance', order:-100}, (props) => e(UsageBalanceBar, {...props, timer, locale})))`。`slots` 缺失时直接 return（静默跳过）。`order:-100` 为负优先级，使其排在脚部动作区靠前位置（设置按钮上方）。组件收到的 `props`（`wide`、`useSessions` 等）由宿主 shell 的槽位渲染器注入，**其确切来源不在本仓库内（待确认，复刻时以宿主实际传入为准）**。
- **响应式会话**：`useSessions((s) => s ? s.current : undefined)` 作为响应式选择器取当前会话 id，变化时重建 60s 轮询 effect（`sessionId` 在依赖数组里）。
- **悬停/定位**：`measureFly(rect)` 用估计尺寸 264×264、边距 12 做视口防溢出（见 §2）；`fly` 状态为 `{top, left}`，未测量时置 `-9999` 隐藏。关闭缓冲：`onMouseLeave` 时 `timer.timeout(..., 220)`，期间再次进入则 `clearHoverTimer()` 取消（`timer.timeout` 返回 disposer）。**坑**：若 `timer` 服务不可用（undefined），关闭缓冲退化为立即关闭、1s 时钟与 60s 轮询也不启动——组件对可选服务做了逐级容错。
- **时钟与轮询**：`timer.interval(() => setNow(Date.now()), 1000)`；数据 `load()` 立即执行一次 + `timer.interval(load, 60000)`；effect 清理时置 `cancelled` 标志防竞态、dispose 定时器。
- **金额文案**：`'≈¥' + (cost<1 ? cost.toFixed(3) : cost.toFixed(2))`（硬编码 ¥，因为估算本就是 CNY）；hover title 的时段拆分只列 >0 的时段，格式 `基础价 ≈¥x.xx · 高峰 ≈¥x.xx · 空闲 ≈¥x.xx`。
- **货币符号**：`CNY→¥`、`USD→$`、其他原样、缺省 `¥`。
- **与 Cordis 面板共存**：CSS 末尾 `div:has(> .ubar),div:has(.ubar):has(+ div [aria-haspopup="dialog"]){flex-direction:column;}`——命中"含 .ubar 且紧跟带 dialog 弹层属性的兄弟节点"的容器时改纵向堆叠，避免两按钮横向挤压。
- **无障碍**：开关为原生 `<button role="switch" aria-checked={pinned}>`；图标 SVG `aria-hidden`。

### 5.4 配置项 / 常量一览

| 常量/配置 | 位置 | 含义 |
| --- | --- | --- |
| `PRICE_SWITCH_MS = Date.UTC(2026,7,16,16)` | src/index.js | 峰谷定价切换时刻（2026-08-17T00:00 北京时间） |
| `PRICE_TABLE`（legacy/offpeak/peak × flash/pro，元/百万 tokens） | src/index.js | 官方价格表快照：legacy flash `{hit:0.02, miss:1, output:2}`、pro `{hit:0.025, miss:3, output:6}`；offpeak flash `{0.05,1.5,4.5}`、pro `{0.15,4.5,13.5}`；peak flash `{0.1,3,9}`、pro `{0.3,9,27}` |
| `FALLBACK_MODEL = 'deepseek-v4-pro'` | src/index.js | 未知模型的计价兜底 |
| `BALANCE_CACHE_MS = 60000` | src/index.js | 余额缓存 TTL |
| `apiKeyEnv`（默认 `DEEPSEEK_API_KEY`）、`baseURL` | `settings.get('llm-deepseek')` | 密钥环境变量名与 API 基址（复用 llm-deepseek 适配器配置，无需额外配置） |
| 15s 超时 / `graceMs:5000` / stdout 64KiB / stderr 16KiB | src/index.js | fetch 与 curl 的超时与采集上限 |
| 60s 轮询 / 1s 时钟 / 220ms 关闭缓冲 / 264×264 估算 / 12px 边距 / z-index 35 | client/client.js | UI 时序与布局常量 |

### 5.5 与 DSH 版本的耦合点（易随版本漂移）

- **服务名**：`webServer`、`sessions`、`sessionProjections`、`settings`、`credentials`、`subprocess`、`sandboxPolicy`、`launchEnvironment`（Host）；`slots`、`timer`、`locale`（Client）。均为 DSH 内部服务，签名以宿主版本为准（dshhub 兼容声明 `dsh >= 0.1.0-rc.5`）。
- **关键时序陷阱**：Host 半的 `apply` 可能早于 `credentials`/`settings` 等服务注册（启动时序），因此**所有服务一律在请求时 `ctx.get` 惰性获取，绝不在 apply 闭包中捕获**——否则会把 `undefined` 冻结进插件，导致永久 no-key。复刻时必须遵守这一条。
- **事件日志 schema**：`session.events` 的 `seq` 连续性与事件类型（`turn/start`、`turn/end`、`request/header`、`assistant/chunk`、`assistant/message`）属于宿主内部结构，随 DSH 版本漂移。
- **tokenUsage 投影形状**：`sessionProjections.snapshot(session).values.tokenUsage` 的 `uncachedInputTokens/cacheWriteTokens/outputTokens/cacheReadTokens` 与内置 token-meter 同源，字段名随统计层变化需同步。
- **Client 加载契约**：`window.__ModuleLoader__.load({id, factory})` 与 `slots.inject/register` 契约、槽位 `sidebar.footer.action` 的 props（`wide`/`useSessions`）由宿主 Web shell 提供（来源待确认）。
- **价格表与切换时刻**：是 2026-08-16 时的官方定价快照，随 DeepSeek 定价变更需手动更新（README 已注明金额为估算）。

## 6. 集成与安装

从拿到本描述到插件生效：

1. **复刻产出**：把本文件全文粘贴给 DSH 会话中的 AI，要求其按 §1–§5 生成一个同名仓库，至少包含 `src/index.js`、`client/client.js`、`cordis.patch.yml`、`package.json`、`README.md`、`LICENSE`。关键点核对：`package.json` 的 `dsh.bundle.patch`、`dsh.client.platform: "web"`、`exports` 的 `.` 与 `./client`；Host 半 `inject: ['webServer']` 且其余服务请求时惰性获取；Client 半 `window.__ModuleLoader__.load` 格式。
2. **放置**：把仓库放到本地（或发布为 GitHub 仓库后走命令安装）。两种方式：
   - 命令安装：`dsh plugin --profile web add github:zdjmrq/dsh-usage-balance`（或你发布的新地址）；
   - 本地调试：将仓库目录作为 bundle 包加入 profile（等价于该 patch 层生效）。
3. **生效**：重启 `dsh web`（`dsh plugin --profile web update dsh-usage-balance` 可更新；`remove` 卸载）。Host 半经 `cordis.patch.yml` 挂载，Client 半经 module-loader 注入 Web 页面。
4. **验证**：见 §8 检查单。其中余额功能需要：能访问 `api.deepseek.com`、`llm-deepseek` 已配置（或 `DEEPSEEK_API_KEY` 环境变量 / 凭据服务可解析）、本机有可用的 `curl`（作为回退）。

## 7. 已知边界与注意事项

- **金额为估算**：价格表为快照，且逐事件模型归因是近似（会话级单模型、首事件前按 v4-pro 兜底），实际账单以 DeepSeek 官方为准。
- **余额强依赖**：需要能访问 `api.deepseek.com` 与有效 API Key；无 key 时显示「未配置 key」，网络失败显示错误行；余额为全局缓存（60s），非实时刷新。
- **内存**：`billing` Map 按会话增长且**无逐出策略**，长跑进程会话极多时内存线性增长（未实现 LRU/清理）；进程重启后清空并从 0 全量重扫（冷启动 O(全部事件)，注释称毫秒级）。
- **固定状态不持久化**：滑杆的「固定」仅存在客户端内存，刷新页面即回「悬停」模式。
- **无会话时**：详情卡运行时长显示「无对话」，但余额仍会请求（cwd 走默认路径）。
- **并发**：余额请求无去抖/无锁，60s 窗口内多个会话各自触发一次 fetch（结果共享缓存）。
- **安全**：key 只用于 Host 侧请求官方接口，客户端通过同源 JSON 路由取余额快照，不会把 key 下发到浏览器；路由无鉴权，任何能访问 `dsh web` 端口的进程都可读该会话用量/余额快照（按仓库现状，未做访问控制，属可接受的局域网内插件约定，复刻时请知悉）。
- **UI 细节**：详情卡行高/间距为固定 CSS 值；`title` 工具提示与详情卡并存；`--dsw-*` 变量缺失的旧版主题下样式可能回退（README 未承诺兼容旧主题）。

## 8. 复刻验收标准（给复刻 AI 的检查单）

- [ ] 宽侧边栏底部「设置」按钮上方出现「用量 | 余额」标签行（仪表盘图标 + 14px 文案，34px 行高），文案随界面中英文切换
- [ ] 悬停标签行，右侧弹出玻璃拟态详情卡；运行时长在有进行中的对话时**每秒跳动**；Token 三行（输入·未命中 / 缓存命中 / 输出）数值与 DSH 内置统计一致；金额显示 `≈¥` 估算值，hover 显示时段拆分
- [ ] 卡片不会超出视口：右侧放不下时自动翻转到左侧，且保持 12px 边距
- [ ] 卡底椭圆滑杆：点击后门帘拉满＝详情常驻（刷新页面后回到悬停模式）；再点收回
- [ ] 侧边栏折叠为窄栏时显示 36×36 圆形仪表盘按钮，悬停同样展开详情卡
- [ ] 与 Cordis 面板按钮纵向堆叠共存，不互相遮挡
- [ ] 在 `llm-deepseek` 已配置（或 `DEEPSEEK_API_KEY` 可用）且网络可达时，余额行显示 `¥` + 官方 total_balance；拔掉 key 后显示「未配置 key」
- [ ] `curl /v1` 剥除验证：`baseURL` 带 `/v1` 或尾部斜杠时，余额端点拼为 `{base}/user/balance`（用临时服务或抓包验证一次）
- [ ] 结构/集成点：`package.json`（`dsh.bundle.patch`、`dsh.client.platform: "web"`、`exports`）、`cordis.patch.yml` insert 行、Host `inject: ['webServer']` + 惰性 `ctx.get`、Client `window.__ModuleLoader__.load` 与 `slots.inject('sidebar.footer.action', ...)`（`order: -100`）与本描述一致
- [ ] 样式跟随亮/暗主题（全部 `--dsw-*` 变量），无硬编码颜色
- [ ] 余额 60s 缓存生效（两次请求间隔 <60s 时仅一次真实出网，可用抓包或日志验证）
