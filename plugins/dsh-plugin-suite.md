# 【文字开源】dsh-plugin-suite — DeepSeek Harness 定制插件套件（改动切片仓库）

> **这是什么**：本文件是「文字开源」描述，不是代码。把本文件全文粘贴给你 DSH 里的 AI 会话，
> AI 即可为你复刻出功能相同的插件套件；你也可以阅读本文件理解其原理，并自行微调需求。
>
> **对应源码**：https://github.com/zdjmrq/dsh-plugin-suite（描述基于 commit `e195a54961a8e5a2625ce768b94f58edd2fe41a6`）｜许可证：MIT
>
> **最近更新**：2026-08-16

> **阅读提示**：本套件不是单一插件，而是一个「定制插件套件（局部 fork）」。复刻它 = 复刻其中两个功能（命令守卫 + 一键重启/刷新）+ 它们在宿主内部的全部接线改动。请先读第 1、3 节理解形态，再读第 2、5 节理解功能细节。

## 1. 插件概述

`dsh-plugin-suite` 是 DeepSeek Harness 的**定制插件套件（局部 fork / 改动切片仓库）**：它不携带官方 [deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) 的整个代码库，只携带「相对上游 base commit 的**改动切片**」——新增的插件包（源码 + 构建产物）+ **一张完整累计补丁 `install.patch`**。因此它轻量、易浏览、可独立审阅；后续所有「需要改动宿主内部」的插件都按同一方式收纳进来。其存在意义是：某些插件光靠 Cordis 组合层做不到，必须给宿主核心打补丁（新增沙箱模式、改审批链路、接设置页 UI），套件把这类改动集中管理，用一张补丁叠加到官方工作区即可启用。

当前收录两个功能（各有一个对应的单功能独立发行仓库，见 §3.5）：

1. **careful-full-access 命令守卫（防误删）**——新增第三个沙箱权限模式 `careful-full-access`（介于 `workspace-write` 与 `danger-full-access` 之间：全权限但删除审慎），并配套一个只在**该模式**下生效的宿主侧命令守卫：为每条 `pwsh`/`bash` 调用做静态分级，危险删除走 WhatIf 干跑 + 模型三问复核，灾难级删除永远以人工确认收尾，全程双重审计。
2. **一键关闭后台 / 刷新前端**——在 Web 界面「设置 → 通用设置」新增两行操作：「关闭后台服务」（优雅关闭 DSH 后端进程，由你手动重启）与「刷新前端」（强制重载页面且**保留创造模式的热插件**）。

适用场景：日常在 `danger-full-access`（沙箱关、审批关）下干活、又怕一条解析错误的 `Remove-Item -Recurse -Force C:\` 酿成大祸的用户；以及经常需要在浏览器里重启/刷新 DSH 服务、不想手敲命令的人。

## 2. 功能规格

### 2.1 careful-full-access 沙箱模式（新增权限档位）

- **入口**：Web 输入框旁的访问模式选择器、`/permission` 命令、权限预设表，均可选中 `careful-full-access`（沙箱 `careful-full-access` + 审批 `ask`），介于 `workspace-write` 与 `danger-full-access` 之间；选择跨会话保持（沿用既有 settings 持久化）。
- **语义**：与 `danger-full-access` 一样**不受文件沙箱约束**（`isUnconfinedMode` 为真），但每条被守卫标记的删除命令都要过复核流水线。官方描述文案：`Current DSH file policy: careful-full-access. File modifications are unrestricted, and every deletion command runs through the command guard preview and model-check confirmation.`
- **提升阶梯**：`read-only → workspace-write → careful-full-access → danger-full-access`；`careful-full-access` 之上可再提升到 `danger-full-access`。

### 2.2 命令守卫（command-guard 插件，仅 careful 模式生效）

**模式门控**：守卫挂在 `tools/pre-execute` 瀑布上，**只对 `careful-full-access` 模式生效**；其他模式（含 `workspace-write`、`danger-full-access`）直接透传，零守卫工作、零审计。

**静态四档分级**（对每条 `pwsh`/`bash` 调用）：

| 档位 | 处置 | 典型命令 |
| --- | --- | --- |
| **normal** | 直接放行 | 非破坏性命令；`git rm --cached`/`-n`/`--staged`（只动索引） |
| **elevated** | model-check 复核 | 单个显式删除、`git clean`、`git reset --hard`、清空回收站、动态目标的递归删除、批量删除 |
| **disaster** | model-check 复核 + **永不自动放行**（须人工确认） | 盘符根（`C:\`）、根级通配（`C:\*`）、UNC 与 `\\?\` 扩展根、用户主目录、系统目录（SystemRoot/ProgramFiles）、工作区根、格式化族（`Format-*`、`Clear-Disk`、`Initialize-Disk`、`Remove-Partition`）、`diskpart clean`、向受保护根 `robocopy /MIR`、受保护根的递归 .NET 删除 |
| **unparseable** | 按 disaster 对待（fail-closed） | `iex`/`Invoke-Expression` 动态执行、AST 失败/超时、删除命令无法安全解析 |

**复核路由**（被标记命令，即非 normal 档）：

1. **WhatIf 干跑解析真实范围**（仅 pwsh 方言的删除命令）：以 `$WhatIfPreference = $true` 在辅助 `pwsh` 进程中干跑一次，由 PowerShell 自己展开通配符/变量/`$env:`，从 `What if:` 输出行（支持英文与 zh-CN 两种文案）解析出实际目标列表；递归目录目标再补一次只读子树枚举（统计文件/目录数 + 采样 ≤10 个路径）。干跑结果：
   - **零目标**（证明删不到东西）→ 直接放行；
   - **命中受保护根**（解析出的真实范围就是受保护根）→ **升级为 disaster 档**，人工确认带红色标注；
   - **无法干跑**（spawn 失败、超时、无输出且非零退出）→ 记录 `unpreviewable` 细节，照常走复核。
2. **model-check 三问复核**：对**会话当前路由模型**做一次小型旁路调用（温度 0、输出上限默认 300 token、超时默认 20s、不可用即 fail-closed），展示命令全文、静态档位、标记原因、预演范围摘要，要求模型严格输出一个 JSON 对象回答三问（意图？是否安全且在范围内？是否确实危险？）。
3. **结果映射**：
   - 模型答 `intent: "no"`（模型自己否认这是本意命令）→ **直接拒绝**（`deny`），工具结果显示 `Error: command guard: model-check concluded this command was not the intended one: <模型解释>`——这是守卫存在的核心误解析场景，不需要人工确认；
   - 模型答 `intent: "yes" + assessment: "safe"` → elevated 档**放行**；disaster 档仍需**人工确认**（最重一级永远由人类兜底）；
   - 模型答 `assessment: "dangerous"` → 无论哪档一律**人工确认**，请求携带模型自己的风险自述；
   - model-check **不可用**（无模型路由、超时、失败、答案不可解析）→ 按 disaster 兜底：**人工确认**。
4. **人工确认**：走常规审批通道；请求携带命令全文、档位标题（`DISASTER tier` / `unparseable (treated as disaster)` / `elevated tier`）、复核结论与预演注记，并经 `severity: 'danger'` 在审批面板**红色突出**（卡片边框/顶部色条/圆点换用 error 主题 token）；会话审批策略为 `never` 时自动拒绝（被标记命令在该会话中不可执行）。

**双重审计**（放行/拒绝/人工确认全部入账）：

- 完整流水 → 轮转文件日志 `$DSH_HOME/logs/command-guard.log`（默认 5 MB × 3 份轮转，串行异步追加，写失败只记日志、绝不翻转守卫判定）；相同命令在去重 TTL（默认 10 分钟）内合并为紧凑的 `{event:"repeat", count, fingerprint}` 标记行；
- 会话日志 → 有界窗口（默认每会话 20 条）的 `command-guard/decision` 事件（字段：`toolName`、`decision: allow|deny|ask`、`tier`、`reason`、`mode`、`callId`、`modelCheck: not-intended|safe|dangerous|unavailable`）；重复判定跳过追加。

**模型可见的提示段**：插件激活时向系统提示注入 `command-guard:deletion-discipline` 段（order 112，固定文本）——教授删除纪律：优先 `-WhatIf` 干跑或显式列举；绝不递归进入盘符根/用户主目录/系统目录；把未定义的 `$env:` 变量当错误；careful 模式被标记命令由模型自查，disaster 档还需人工确认。

**Windows ACL 加固（接线改动，与守卫互补）**：`sandbox-windows-acl` 对工作区根的授权从「单条 OI|CI 完整 GRANT_MASK ACE」拆为两条：一条 **inherit-only**（`OI|CI|IO`）完整 GRANT_MASK ACE 供子孙对象（子进程删除/改名/git 所需 Write+Delete 不受影响），一条**仅根对象**的 `ROOT_GRANT_MASK`（`0x00100116`，无 DELETE、无 FILE_DELETE_CHILD）ACE。效果：受限子进程无法删除或改名工作区根本身；递归删除可以清空树内内容但**永远拿不走根**。旧形态单 ACE 原地迁移（一次 revoke+pair 合并）。根改名普遍被拒（改名检查对象自身 DELETE）；根删除走父目录 FILE_DELETE_CHILD，父目录未向调用者授权时 ACL 即拦住——带环境同用户/Everyone 类 ACE 的父目录仍是既有 partial 边界，因此守卫的 disaster 档才是根删除的主防线。

### 2.3 一键关闭后台 / 刷新前端

**「关闭后台服务」行**（设置 → 通用设置，order 30，id `restart-backend`）：

- 点击 → 弹出置顶确认对话框（`shell.overlay`，portal 到 body）：「确定要关闭 DSH 后台服务吗？页面会断开，需手动重启后台并重新打开页面；创造模式的热插件会消失」。
- 确认 → Host 侧 Typert Remote `restart` 被调用：先应答浏览器（600ms 延迟后）再调用 launcher 的 `appExit(0)` 优雅退出整个 Cordis 树，进程干净退出；返回消息附**重启提示**（由当前进程 `process.execPath` + argv + cwd 拼出的可重跑命令）。
- 对话框进入「正在关闭…」阶段并展示 Host 状态细节；浏览器尽力 `window.close()`（脚本打开的窗口才会被关闭，关不掉也无妨——后端退出后页面自然断开）。
- 失败路径：`appExit` 不存在 → 提示手动关闭；请求重入 → `restart already in progress`；对话框显示错误阶段。

**「刷新前端」行**（order 40，id `refresh-frontend`）：

- 点击 → 确认对话框：「页面将重新加载，创造模式的热插件会保留，后台服务不会关闭」。
- 确认 → 设置 sessionStorage 标志 `dsh:reattach-cordis-runs`，然后 `window.location.reload()`。
- **F5 / Ctrl+R 快捷键同样生效**（仅纯 F5 与纯 Ctrl+R；Ctrl+Shift+R 强刷保留浏览器原生行为）：`preventDefault` + `location.reload()` 强制重载，兼容桌面壳/嵌入视图里 F5 默认行为被抑制的环境。
- **热插件保留机制**：新页面加载后，`ui-cordis` 的库存订阅发现该标志，一次性把仍在运行的动态插件（有 Client 半、无待审批请求、当前未加载）的 Client 半重新 attach（`runner.startUserRun` 复用宿主已有 run，而非重启它），随后清除标志。Host 半本来就在宿主里，不受页面刷新影响。

**对话框行为细节**：`connection/reset`（后端重启导致的连接重置）自动关闭对话框；确认阶段可点遮罩/取消关闭；确认后按钮防重入（busy 态）。

## 3. 技术路线

### 3.1 插件形态与包构成

| 包 | 形态 | 角色 |
| --- | --- | --- |
| `@deepseek-ai/dsh-careful-full-access`（`packages/guard/careful-full-access/`） | **Host 半** Cordis 插件，插件名 `command-guard`，`inject: ['tools']` | 命令守卫：`tools/pre-execute` 监听器 + 分级/复核/审计引擎 + 系统提示段 |
| `@deepseek-ai/dsh-host-restart`（`packages/host/restart/`） | **Host 半**，Typert Remote 服务（`RestartGateway extends TypertRemoteService`，`@Remote('restart')`） | 一键优雅关闭后端（`appExit(0)`） |
| `@deepseek-ai/dsh-client-ui-settings-restart`（`packages/client/ui-settings-restart/`） | **Client 半**，浏览器插件（`dsh.client.platform: web`） | 设置页两行 + 确认对话框 + F5/Ctrl+R 处理 + 语言包 |

每个包都是标准 deepseek-harness 包结构：`src/`（TypeScript 源码）+ `lib/`（tsdown 构建产物，含 `.js`/`.d.ts`/`.js.map`）+ `package.json`（`exports` 含 `./src/*`）+ `tsconfig.json` + `tsdown.config.ts`；guard 包还带 `tests/`（vitest）与三语 README（`README.md`/`README.zh.md`/`README.i18n.yaml` 配对校验）。版本号均为 `0.1.0-rc.5`，与上游 base 一致。

### 3.2 加载与集成方式（接线改动，全部在 `install.patch` 内）

- **组合注册**：`packages/bundle/base/cordis.patch.yml` 增加插件行 `command-guard`（Host 平面，所有会话的 shell 调用都过它）与权限预设表第三档 `careful-full-access: { sandbox: careful-full-access, approval: ask }`；`packages/bundle/web-app/cordis.patch.yml` 增加 Host 行 `restart` 与 Client 行 `ui-settings-restart`。
- **依赖声明**：`packages/bundle/base/package.json`、`packages/bundle/web-app/package.json`、`packages/api/remotes/package.json` 增加 workspace 依赖。
- **tsconfig 引用**：`tsconfig.host.json` 增加 `guard/careful-full-access` 与 `host/restart` 两个 project 引用；`tsconfig.client.json` 增加 `client/ui-settings-restart`；`packages/api/remotes/tsconfig.client.json` 增加 `host/restart`。
- **Remote API 装配**：`packages/api/remotes/src/client/index.ts` import `restartRemote` 并 `ctx.remote.$mount`，导出 `RestartAck` 类型。
- **宿主核心改动**：见 §5.4 清单（沙箱词汇、ACL、审批链路、UI 组件、会话事件等）。
- **pnpm-lock.yaml**：随之更新。
- **文档**：README 功能表、`docs/config-catalog*`（新沙箱模式）、`docs/event-producer-consumer*`（`command-guard/decision` 事件）、`docs/subsystems/{sandbox,tools,approval}*`、`docs/module-graph*` 同步更新；`.agents/notes/implemented/feature/2026-08-15-command-guard-careful-full-access.*` 与 `2026-08-15-command-guard-model-check-refactor.*` 是两份设计说明（详见 §5）。

### 3.3 依赖的核心 Service / Event / API

**守卫插件**（版本敏感点：均为 dsh 0.1.0-rc.5 时代的接口形状）：

- `ctx.on('tools/pre-execute', handler)`——宿主工具注册表的执行前瀑布；handler 签名 `(exec, next) => Promise<PreToolDecision>`，`exec` 携带 `name`（`pwsh`/`bash`）、`arguments`（取 `command` 字段）、`agent`、`signal`、`callId`。
- `PreToolDecision`——判定联合：`{kind:'allow'} | {kind:'deny'; reason} | {kind:'ask'; reason?; severity?: 'danger'}`（**severity 字段是本套件新增**）。
- `ctx.get('sandboxPolicy')`→`SandboxPolicyService.resolve({session})`——每次调用解析 `mode`（含新值 `careful-full-access`）与 `workspaceRoot`。
- `ctx.get('llm')` 的 `stream(options)`——model-check 的补全通道；入参 `{provider, model, messages, system, temperature: 0, maxTokens, signal}`，流式块 `{type:'text-delta'|'finish', ...}`；消息经 `createUserMessage({content, source:{kind:'plugin', plugin:'command-guard'}})` 构造。会话模型路由取自 `agent.session.requestHeader()?.config.provider/model`（回退 `agent.options`）。
- `ctx.inject(['systemPrompt'], scope => scope.systemPrompt.context({name, order, text}))`——注入删除纪律提示段（order 112）。
- `session.append('command-guard/decision', payload)`——会话侧审计事件；类型经 `declare module '@deepseek-ai/dsh-session/types'` 扩展示例，并登记进 `KNOWN_SESSION_EVENT_TYPES`。
- `resolveDshHome()`（`@deepseek-ai/dsh-home-paths`）——定位 `$DSH_HOME`，审计日志默认路径 `$DSH_HOME/logs/command-guard.log`。
- 审批链：`PreToolDecision.ask` → `ctx.approval.request`（`ApprovalRequest` 新增 `severity?: 'danger'`）→ `approval/asked` 事件 → `approval/requested` mux 帧（`events.schema.ts` zod 校验）→ 审批面板。

**重启插件**：

- Host 侧：`ctx.get('appExit')`——launcher 暴露的优雅退出函数 `(code: number) => void`（不存在则提示手动关闭）；`TypertRemoteService`/`@Remote`（`@deepseek-ai/dsh-typert-protocol`）声明 RPC 方法。
- Client 侧：`ctx.remote.restart.restart()`（由 `@deepseek-ai/dsh-api-remotes` 装配的生成面）；`ctx.slots.inject('settings.general.item', ...)` 与 `ctx.slots.register({name, id, order, locale, inject}, Component)` 注册设置行；`ctx.slots.inject('shell.overlay', ...)` 注册对话框；`ctx.locale.register('settings.restart', {zh, en})`；`ctx.on('connection/reset', ...)` 重置对话框。
- `ui-cordis`（宿主改动）：库存订阅里消费 sessionStorage 标志 `dsh:reattach-cordis-runs`，对 `snapshot.rows` 中 `activeRun` 存在、无 `approvalRequestId`、未加载、包有 Client 半的插件行调用 `runner.startUserRun({agentId, pluginId, packageId, mode:'run', hasClientHalf:true})` 重新 attach。注意：标志字符串是跨插件共享的字面量（client bundle 纯度门禁禁止跨包值导入）。

### 3.4 关键外部依赖与系统能力

- **PowerShell**（`pwsh`）：AST 精析辅助进程（`-NoLogo -NoProfile -NonInteractive -EncodedCommand`）、WhatIf 干跑、子树枚举都用它；`pwshPath` 可配置，默认 `'pwsh'`。spawn 层（`node:child_process` spawn + 超时 kill + 有界流收集）可注入，全部进程路径可测试。
- **Windows Win32 API**（经 `koffi` 绑定，`sandbox-windows-acl`）：`SetEntriesInAclW`/`SetNamedSecurityInfoW`/`GetNamedSecurityInfoW`/`LocalFree` 等；ACL 结构与 SID 均为指针偏移读取（koffi.decode），ACE 内联 SID 逐字段比对。
- **无其他系统能力**：守卫本身不依赖桌面通知/Electron 等；所有路径皆进程内 + 辅助子进程。

### 3.5 与其他 dsh-* 仓库的配套关系（双向互链）

- [deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)——官方上游；本套件的安装目标。
- [dsh-careful-full-access](https://github.com/zdjmrq/dsh-careful-full-access)——「命令守卫 + careful-full-access 沙箱模式」的**单功能独立发行**（自带 `patches/careful-full-access.patch` 核心补丁、支持 npm 安装），适合只要这一项的人。
- [dsh-restart-plugin](https://github.com/zdjmrq/dsh-restart-plugin)——「关闭后台 / 刷新前端」的**单功能独立发行**（`install.patch` 仅含该功能）。
- 三者同源：本套件是合集（含完整累计补丁）；两个独立仓库是拆出的单功能版。需要哪个形态就取哪个。

## 4. 结构设计

### 4.1 仓库布局

| 路径 | 职责 |
| --- | --- |
| `README.md` | 套件说明：形态、功能表、安装步骤、扩展指引、相关项目互链 |
| `LICENSE` | MIT |
| `.gitignore` | `node_modules/`、`*.tsbuildinfo`（构建产物 `lib/` 被收纳，但忽略 node_modules 与 tsbuildinfo） |
| `install.patch` | **完整累计补丁**（~481 KB，136 个文件）：全部接线改动 + 三个新包全部文件；由 `git diff <base> HEAD` 生成 |
| `packages/guard/careful-full-access/` | 命令守卫包（Host 半） |
| `packages/host/restart/` | 一键关闭后端包（Host Remote） |
| `packages/client/ui-settings-restart/` | 设置页两行 + 对话框包（Client 半） |
| `.agents/notes/implemented/feature/` | 设计说明归档（整个官方仓库都有的惯例目录；本套件新增 2026-08-15 两篇） |

### 4.2 守卫包内部模块（`packages/guard/careful-full-access/src/`）

| 文件 | 职责 |
| --- | --- |
| `index.ts` | 插件入口：`apply(ctx, config)`；注册 `tools/pre-execute` 监听、审计 sink、提示段；导出全部公共 API 与默认常量 |
| `engine.ts` | `GuardEngine`：每次调用一次判定（模式门控 → 词法快筛 → AST 精析 → 分级 → 复核路由）；持有 analyzer/preview/modelCheck/protectedRoots 四个注入件，本身无状态，一个实例服务所有会话 |
| `lexer.ts` | 进程内零成本词法预筛：tokenize + 危险动词/别名表、cmd 开关、.NET 删除调用、动态标记、顶层 `git` 分派；`hasDestructiveSignal` 是快速放行闸门；宁可误报不可漏报 |
| `analyzer.ts` | PowerShell AST 精析：内嵌 PowerShell 脚本（UTF-16LE base64 `-EncodedCommand`），命令经 `$env:DGUARD_CMD` 传入，`Parser::ParseInput` **只解析不执行**，输出单行 JSON（verbs/strings/expandables/variables/parameters/.NET member calls）；`nodeSpawner` 默认 spawn 实现 |
| `tiers.ts` | 纯函数四档分类器 `classifyPwsh`/`classifyBash`：档位与模式无关（哪个档=放行/复核/复核+人工是 engine 的决定）；受保护根命中、递归/强制/动态/批量等细粒度规则 |
| `model-check.ts` | `ModelCheckRunner`：对会话路由模型的单次旁路补全；严格 JSON 三问提示词；`parseModelAnswer` 宽容解析（裸 JSON / 围栏 / 花括号抽取）；超时/中止/失败全部 fail-closed 为 `unavailable` |
| `preview.ts` | `PreviewRunner`：WhatIf 干跑（`$WhatIfPreference=$true`）+ 递归目录只读子树枚举；中英双语文案解析；`renderPreviewSummary` 生成有界摘要（≤10 个采样路径） |
| `protected.ts` | 受保护根注册表与路径判定：盘符根/根通配/UNC/扩展根/裸盘符/POSIX 根的结构化匹配 + 环境派生（USERPROFILE/HOME、SystemRoot、ProgramFiles×3）+ 用户额外配置；大小写折叠（仅 Windows 形态） |
| `git.ts` | 顶层 `git` 子命令分派：`rm --cached`/`-n` 正常，`rm`（无 cached）、`clean`、`reset --hard` 破坏性；`-C`/`-c` 等取值选项正确跳过 |
| `verbs.ts` | 危险动词表：pwsh 删除/格式化/回收站族 + cmd 别名、bash 删除/格式化族、cmd 开关、.NET 删除调用正则、`iex` 动态动词、`diskpart clean`、`robocopy /MIR`、`find -delete` |
| `audit.ts` | 双重审计：`AuditLogger`（JSONL 轮转文件日志，串行 promise 链，失败入 onError）+ `SessionAuditGate`（会话事件上限 + 去重合并） |
| `fingerprint.ts` | 命令指纹（空白折叠）+ 滑动 TTL 去重窗口 |
| `types.ts` / `invariant.ts` | 共享类型 / 断言工具 |

调用关系（单次判定）：

```
tools/pre-execute 事件
  → index.ts（模式门控：非 careful-full-access 直接 next()）
  → GuardEngine.judge
      → lexer（hasDestructiveSignal？无 → allow）
      → classifyPwsh/classifyBash（词法快档；git/灾难档就地判定）
      → 需要时 PwshAnalyzer（AST 辅助进程）→ 再 classify
      → route()：normal → allow；否则 review()
          → PreviewRunner（仅 pwsh 删除：WhatIf 干跑 + 子树枚举）
              → zero-targets → allow；protected-hit → 升 disaster
          → ModelCheckRunner（三问 JSON）
              → not-intended → deny；safe+elevated → allow
              → safe+disaster / dangerous / unavailable → ask(severity:'danger')
  → 审计（AuditLogger 文件 + session.append 事件，去重计数）
  → 返回 PreToolDecision → ToolRuntime 按 allow/deny/ask 继续
```

### 4.3 重启/刷新功能结构

- `packages/host/restart/src/index.ts`：`RestartGateway extends TypertRemoteService`，`@Remote('restart') restart()`——armed 防重入 → 600ms 后 `ctx.get('appExit')?.(0)` → 返回 `{ok, message: 重启提示}`。
- `packages/client/ui-settings-restart/src/client/index.ts`：注册两行（`settings.general.item`，order 30/40）+ 对话框（`shell.overlay`，order 0）；F5/Ctrl+R 键处理；`connection/reset` 关闭对话框；`REATTACH_FLAG = 'dsh:reattach-cordis-runs'` 常量。
- `restart-store.ts`：极简可观察 store（`openConfirm`/`beginRestart`/`fail`/`restarting`/`close`，阶段 `confirm|restarting|error`）。
- `RestartRow.tsx` / `RestartDialog.tsx`：React 组件（CSS Modules；对话框 portal 到 body）；`locales.ts`：`zh`（key 源）+ `en` 双语字典。

## 5. 关键实现细节

### 5.1 核心逻辑步骤化（守卫判定流水线）

1. **模式门控**：`exec.name` 不是 `pwsh`/`bash`，或 `arguments.command` 缺失，或解析出的 `sandboxPolicy.mode !== 'careful-full-access'` → `next()` 透传。守卫**绝不**在其余模式做任何工作或审计。
2. **词法快筛**（`lexPwsh`/`lexBash`，无进程、无状态）：tokenize（引号保留）→ 若首动词是 `git`/`git.exe`，进入顶层 git 分派（`analyzeGit`）并返回；否则扫描：危险动词族（别名解析）、cmd 风格开关（`/s`、`/q`、`/f`、`/mir`）、`-Recurse`/`-Force`、路径 token（盘符/UNC/绝对路径）、通配符、`$`/反引号动态标记、`iex` 动态动词、`.NET` 删除调用正则、`diskpart clean`、`find -delete`。`hasDestructiveSignal` 为假 → 放行（绝大多数命令走这条快路，零 spawn）。
3. **AST 精析**（仅 pwsh 且有破坏信号、且不是词法已证实的 disaster/git 场景）：spawn `pwsh -EncodedCommand <ANALYZER_SCRIPT base64>`，脚本从 `$env:DGUARD_CMD` 读命令，`Parser::ParseInput` 解析（不执行），遍历全部 `CommandAst` 输出 verbs/strings/expandables/variables/parameters 与 .NET member calls 的单行 JSON；超时（15s 默认）/spawn 失败/非零退出/不可解析 → `ok:false` 报告，fail-closed。
4. **四档分级**（`classifyPwsh`，先 git 分派后家族判断）：
   - `dynamicVerb`（iex）→ `unparseable`；
   - `format` 族或 `diskpartClean` → `disaster`；
   - `robocopy /MIR` 命中受保护根 → `disaster`；`.NET` 递归删除受保护根 → `disaster`；`.NET` 删除受保护根 → `elevated`；`.NET` 删除受保护根内部 → `elevated`；`delete` 族 + 递归 + 受保护根 → `disaster`；
   - `recycle`（清空回收站）→ `elevated`；
   - 无可用 AST 报告 → 非灾难破坏命令一律 `unparseable`；
   - `delete` 族细分：递归+动态目标 → elevated；递归+强制+（通配/工作区外）→ elevated；递归+无静态目标 → elevated；多目标（>1）→ elevated（批量）；其余 → elevated。
   - bash 侧（`classifyBash`，无 AST）：`format` 族 → disaster；递归+受保护根 → disaster；递归+动态 → elevated；递归+强制+（通配/工作区外）→ elevated；`find -delete` → elevated；delete 族 → elevated。
5. **复核路由**（见 §2.2 的第 1~4 步），关键映射再强调：`preview` 的 `zero-targets` 直接 allow（干跑已证明删不到东西，无需复核）；`protected-hit` 把档位升为 disaster 再送模型；`unpreviewable` 只影响请求文案（追加「无法干跑预演」注记），不改变档位。
6. **结果落账**：文件日志每判定一行 JSON（`ts`/`sessionId`/`toolName`/`decision`/`tier`/`mode`/`modelCheck`/`fingerprint`/`count`）；会话事件受窗口上限与去重控制；重复只向文件追加 `repeat` 标记行。

### 5.2 重要边界处理与坑（实战踩过的雷）

- **WhyIf 干跑是「问引擎而不是问守卫」**：通配、变量、`$env:` 的展开由 PowerShell 自己完成，守卫的解析永远不可能成为误读的那一环——这是选择 WhatIf 而非自写路径展开的原因。
- **干跑的副作用权衡**：命令带着 `$WhatIfPreference=$true` 真实执行一遍，删除 cmdlet 之前可能存在的非删除副作用**会真实发生**——文档明确记录的取舍。
- **AST 脚本传递方式**：命令文本经 `-EncodedCommand` + 环境变量传递，绝不进命令行参数，任何引号层都无法破坏它；脚本内 `$ErrorActionPreference='Stop'`，解析错误照样输出 `{ok:false,...}` 单行 JSON 后退出 0，由宿主解析层判定。
- **词法宁可误报**：AST 会精化；词法漏报不可接受，故 tokenizer 对每个可能是动词位置的 token 都过表（别名与 cmd 风格二进制都能浮出）。
- **`git` 只能做顶层分派**：管道或嵌套的 `git` 退回通用扫描，可能误读子命令语义（如管道里的 `git rm` 读成删除动词）——记录为已知局限，宁可多复核不可漏。
- **.NET 删除是成员表达式而非 CommandAst**：词法与 AST 两层都按文本正则匹配 `[IO.Directory]::Delete(path, $true)` 签名，递归标志从第二参数字面 `true` 判定。
- **ACL 拆分细节**：仅根 ACE 无继承 + 无 DELETE；子对象靠 inherit-only 完整 ACE（删除/改名子对象需要的是**子对象自身**的 DELETE，父级的 inherit-only 恰好供给）；旧单 ACE 形态一次性迁移；幂等跳过（已存在 exact pair 时**跳过** `SetNamedSecurityInfoW`，否则会触发整树 eager 继承重传播，大工作区耗时数分钟）。
- **审批 severity 透传全链路**：`PreToolDecision.ask.severity` → `ApprovalRequest.severity` → `approval/asked` 事件 → apiproxy `approval/requested` mux 帧（zod 校验）→ 前端 `PendingApproval.severity` → `ApprovalPanel` 红色样式（error 主题 token，无硬编码色值）。
- **审计失败不得翻转判定**：文件日志写失败仅 `ctx.logger.warn`；会话事件 append 失败同样只告警——判定已成立且由流水线强制，审计只丢痕迹不丢结论。
- **模型答案宽容解析**：裸 JSON、``` ```json ``` 围栏、花括号片段三种候选都试；严格校验 `intent`/`assessment`/`explanation` 字段；解析不出 → `unavailable` → disaster 兜底。
- **超时/中止区分**：model-check 超时（内部 AbortController）与外部工具调用中止分别判定，都归 `unavailable` 但 detail 文案不同。
- **`never` 审批策略下的行为**：disaster/危险命令一律自动拒绝——被标记命令在该会话不可执行，这是有意为之的 fail-closed。
- **F5 兼容**：`preventDefault` + `location.reload()` 使刷新在桌面壳/嵌入视图（浏览器 F5 默认行为被抑制）也生效；Ctrl+Shift+R 故意保留原生强刷（不设 reattach 标志，热插件不保留——符合用户「绕过缓存强刷」的预期）。

### 5.3 配置项 / 常量（含默认值）

**守卫插件 Config**（schemastery 校验，全部可选、schema 给默认）：

| 键 | 默认 | 含义 |
| --- | --- | --- |
| `extraProtectedPaths` | `[]` | 额外受保护根（绝对路径） |
| `dedupeTtlMs` | `600000`（10 分钟） | 相同命令合并审计的窗口 |
| `analyzeTimeoutMs` | `15000` | AST 分析 spawn 截止 |
| `previewTimeoutMs` | `15000` | 干跑/枚举 spawn 截止 |
| `previewSampleLimit` | `10` | 预演摘要采样路径上限 |
| `modelCheckTimeoutMs` | `20000` | model-check 整调用截止 |
| `modelCheckMaxTokens` | `300` | model-check 输出预算 |
| `auditLogPath` | `''`→`$DSH_HOME/logs/command-guard.log` | 审计文件路径 |
| `auditLogMaxBytes` | `5*1024*1024` | 轮转阈值 |
| `auditLogRotations` | `3` | 轮转份数 |
| `sessionDecisionCap` | `20` | 每会话事件上限 |
| `pwshPath` | `'pwsh'` | 辅助可执行 |
| `enablePrompt` | `true` | 是否注册删除纪律提示段 |

**其他关键常量**：`EXIT_DELAY_MS = 600`（restart Remote 应答后延迟退出）；`REATTACH_FLAG = 'dsh:reattach-cordis-runs'`（ui-settings-restart 写、ui-cordis 读）；`ROOT_GRANT_MASK = 0x00100116`、`GRANT_MASK = 0x00110156`、`INHERIT_ONLY_ACE = 0x8`、`SUB_CONTAINERS_AND_OBJECTS_INHERIT = 0x3`；提示段 order `112`；守卫包内各 spawn 流收集上限 `262144` 字符、报告字段裁剪上限（strings/expandables/variables/parameters 各 64、memberCalls 16）。

### 5.4 与 DSH 版本相关的耦合点（哪些改了宿主内部、哪些随版本漂移）

| 改动点 | 所在宿主包 | 性质 |
| --- | --- | --- |
| `SandboxMode` 联合类型新增 `'careful-full-access'` + `ConfinedSandboxMode` 排除 + `isUnconfinedMode` 谓词 | `packages/sandbox/sandbox` | **宿主内部**，随版本漂移 |
| `WIDER_MODES`/`ESCALATION_TARGETS` 提升阶梯 | `packages/sandbox/sandbox` | **宿主内部** |
| `sandboxPolicy` 文案/`SANDBOX_MODES`/config 默认 | `packages/sandbox/sandbox-policy` | **宿主内部** |
| `grantWrite`/`revokeWrite` ACE 拆分 + `ROOT_GRANT_MASK` + `buildExplicitAccess` 继承参数 | `packages/sandbox/sandbox-windows-acl` | **宿主内部**（koffi/Win32 绑定，仅 Windows） |
| pwsh/bash/terminal/fs 四处的 unconfined 分支 | `pwsh-sandbox`、`bash-sandbox`、`terminal-bash`、`fs-sandbox` | **宿主内部** |
| 权限预设第三档 | `packages/interaction/permission-presets` | **宿主内部** |
| `PreToolDecision.severity`、`ApprovalRequest.severity`、`approval/asked` 事件 | `packages/core/tools`、`packages/interaction/user-approval` | **宿主内部**，工具/审批契约 |
| apiproxy `approval/requested` mux 帧 severity（含 zod schema） | `packages/host/apiproxy` | **宿主内部**，wire 契约 |
| ApprovalPanel 红色 danger 样式 + `PendingApproval.severity` + PermissionSelect 眼睛图标 | `packages/client/ui-conversation` | **宿主内部**，Web UI |
| `command-guard/decision` 事件注册 | `packages/core/session`（`known-event-types.ts`） | **宿主内部** |
| ui-cordis 刷新后重新 attach 热插件（REATTACH_FLAG 消费） | `packages/extensions/ui-cordis` | **宿主内部** |
| Remote 装配与 slot-catalog/api-catalog 登记 | `packages/api/remotes`、`cordis-client-runner`、`tool-cordis` | **宿主内部**（多为生成物/目录） |
| 组合行、依赖、tsconfig 引用、lockfile | bundle 包 + 根配置 | 接线，随版本漂移 |
| `tools/pre-execute`、`ctx.llm.stream`、`ctx.appExit`、`settings.general.item`/`shell.overlay` slot | 既有稳定接口 | **上游 API**，相对稳定 |

**版本基线与漂移对策**：上游 base commit `47f943859b`（dsh `0.1.0-rc.5` 时代）。`install.patch` 在 base 上干净应用；上游前进后可能不干净——此时对照补丁手工合并（改动点均为组合注册、依赖声明与 tsconfig 引用、以及上表宿主内部项，结构清晰）。新增的语义词汇（`careful-full-access` 模式、`severity:'danger'`、`command-guard/decision` 事件）若上游未来自己引入同名概念，需重新对齐。

### 5.5 两份设计说明（归档于 `.agents/notes/implemented/feature/`，值得先读）

- `2026-08-15-command-guard-careful-full-access.*`：v1 决策——为什么用「命令语义分析 + 两步模型复核确认的审慎全权限模式 + 让工作区根不可删除的 ACE 拆分」三层；否决过的方案（路径级授权、回收站软删除的虚假安全感——受限令牌下 `$Recycle.Bin` 不可写会静默降级为永久删除、minifilter 驱动级拦截成本过高）。
- `2026-08-15-command-guard-model-check-refactor.*`：v2 重构——守卫只守 careful 模式（不再干预 danger-full-access）；复核从「首次提交拒绝 + 原样重发指纹协议」改为「三问 JSON + disaster 永不自动放行 + 人工兜底」；会话无限增长审计改为轮转文件 + 有界会话窗口；`git` 词法修正（`rm --cached`/`-n`/`--staged` 只动索引）。

## 6. 集成与安装

**场景 A：你有官方 deepseek-harness 工作区（推荐，等于复刻安装）**

1. 把本文件全文粘贴给 DSH 里的 AI 会话，要求它「在 deepseek-harness 工作区（checkout 到 `47f943859b`）上按描述复刻 dsh-plugin-suite 套件」。
2. AI 产出：三个插件包目录（`packages/guard/careful-full-access/`、`packages/host/restart/`、`packages/client/ui-settings-restart/`，含源码 + `tsdown` 构建产物）+ §5.4 表中全部宿主接线改动 + 更新 `pnpm-lock.yaml`。
3. 落到工作区后验证接线齐全：`packages/bundle/base/cordis.patch.yml` 有 `command-guard` 行与 `careful-full-access` 预设；`packages/bundle/web-app/cordis.patch.yml` 有 `restart` 与 `ui-settings-restart` 行；`tsconfig.host.json`/`tsconfig.client.json`/`api/remotes/tsconfig.client.json` 有引用；`api/remotes/src/client/index.ts` 挂载了 `restartRemote`。
4. `pnpm install && pnpm run build && pnpm dsh web` 启动。
5. 验证（见 §8 检查单）。

**场景 B：已有本套件的 `install.patch`（或复刻 AI 额外生成一张）**

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
git checkout 47f943859b
git apply /path/to/dsh-plugin-suite/install.patch   # 完整累计补丁（组合接线 + 全部新包文件）
pnpm install
pnpm run build
pnpm dsh web
```

若上游已前进导致 patch 不干净，对照补丁手工合并（见 §5.4 漂移对策）。

**配置生效途径**：守卫插件的 Config（§5.3）通过宿主组合层（`cordis.patch.yml` 插件行下挂配置）或部署自己的 composition 设置；沙箱模式通过 Web 权限选择器 / `/permission careful-full-access` / settings 持久化选择。审计日志默认出现在 `$DSH_HOME/logs/command-guard.log`。

## 7. 已知边界与注意事项

- **模式门控是设计而非缺陷**：`workspace-write` 与 `danger-full-access` 下守卫完全不介入（零审计）——前者沙箱已约束，后者是用户明确放手；想被守卫保护就必须切到 `careful-full-access`。
- **`iex`/动态构造无法静态分析**：fail-closed 按 disaster（人工确认；`never` 策略下自动拒绝），`iex` 绝不漏过闸门。
- **bash 无 WhatIf 等价物**：POSIX 上复核不带解析出的范围摘要，全靠模型自查 + 人工兜底。
- **只有顶层 `git` 获得子命令分派**：管道/嵌套 `git` 退回通用扫描，可能误读其子命令语义（宁可多复核）。
- **model-check 成本**：每条被标记命令消耗一次模型调用（延迟 + 约 300 token 输出）；其判断质量继承复核模型的水平——这正是 disaster 档与模型自称危险的命令永远以人工收尾的原因。
- **WhatIf 干跑的副作用**：删除 cmdlet 之前的非删除副作用会真实执行（见 §5.2）。
- **ACL 的 partial 边界**：父目录带环境同用户/Everyone 类 ACE 时根删除仍可能穿过 ACL；守卫的 disaster 档才是根删除主防线。ACL 改动仅 Windows。
- **回收站软删除不可用**：受限令牌下 `$Recycle.Bin` 不可写、会静默降级为永久删除——v1 已否决该方案，不要引入虚假安全感。
- **重启功能的取舍**：关闭后端是「手动重启」而非自拉进程（无 relaunch 机制，简单可靠）；创造模式热插件在关闭后端时消失（对话框已明示）；刷新前端保留热插件依赖 `sessionStorage` 标志，被浏览器禁用存储时刷新仍发生但不保留热插件。
- **未完成项（作者自述）**：manual/auto 确认策略、持久化规则表（「始终允许此模式」）、L4 可恢复层（软删除/git 撤销）均为后续项。
- **待确认**：本套件仓库的公开 URL 未在仓库内给出（README 只给了两个单功能独立仓库的 URL）；`dsh-plugin-suite` 自身的远端地址需向作者确认。

## 8. 复刻验收标准（给复刻 AI 的检查单）

- [ ] **沙箱模式**：`SandboxMode` 含 `'careful-full-access'`；`/permission` 命令与 Web 权限选择器可选该档（选择器显示「眼睛」图标）；`WIDER_MODES` 阶梯为 `read-only → workspace-write → careful-full-access → danger-full-access`；四个非受限分支（pwsh-sandbox/bash-sandbox/fs-sandbox/terminal-bash）改用 `isUnconfinedMode`。
- [ ] **模式门控**：在 `workspace-write` 与 `danger-full-access` 下执行 `Remove-Item -Recurse -Force C:\` 直接放行、守卫零介入（无审计）；切到 `careful-full-access` 后同一命令被拦（disaster → 人工确认，红色 `DISASTER tier` 标注；审批策略 `never` 时自动拒绝）。
- [ ] **四档分类**：`Get-ChildItem C:\ws` 放行；`Remove-Item C:\ws\a.txt`（elevated，模型答 safe）放行；`Remove-Item -Recurse -Force C:\` 为 disaster；`iex (Get-Content x)` 为 unparseable（fail-closed）；`git rm --cached f` 放行、`git clean -fd` 被标记、`git -C repo status` 放行。
- [ ] **WhatIf 预演**：pwsh 删除命令（如带通配 `Remove-Item C:\ws\*.tmp`）复核请求/模型提示中包含解析出的真实范围摘要（「此删除将解析为 N 个对象…」）；干跑解析到受保护根时升级为 disaster 档红色人工确认；干跑证明零目标时直接放行、不消耗模型调用。
- [ ] **model-check 三问**：模型答 `intent:"no"` → 工具结果 `Error: command guard: model-check concluded this command was not the intended one: …`（直接拒绝、无人工确认）；答 `dangerous` → 无论档位一律人工确认并附风险自述；model-check 不可用（断路由/超时）→ 按 disaster 人工确认。
- [ ] **双重审计**：`$DSH_HOME/logs/command-guard.log` 存在且每判定一行 JSON（含 `ts`/`decision`/`tier`/`mode`/`modelCheck`/`fingerprint`）；文件超 5 MB 轮转为 `.1`…`.3`；相同命令 10 分钟内重复只追加 `{event:"repeat",count}`；会话日志 `command-guard/decision` 事件每会话不超过 20 条。
- [ ] **审批红色显示**：disaster 人工确认在 Web 审批面板呈现红色卡片边框/色条/圆点（error 主题 token，非硬编码色值）。
- [ ] **提示段**：激活插件后模型系统提示出现 `command-guard:deletion-discipline`（order 112）删除纪律段。
- [ ] **ACL 拆分**（Windows）：工作区根的 DACL 呈现「inherit-only 完整 GRANT_MASK ACE + 根 ROOT_GRANT_MASK ACE」两条；受限子进程无法改名/删除工作区根，但工作区内子对象删除/改名/git 正常；旧单 ACE 工作区升级后自动迁移。
- [ ] **重启功能**：Web 设置 → 通用设置出现「关闭后台服务」（order 30）与「刷新前端」（order 40）两行；「关闭后台服务」确认后后端进程优雅退出（返回含可重跑命令的提示、对话框进入「正在关闭…」、页面随后断开）；「刷新前端」确认后页面重载、后端不退出、创造模式热插件保留（`sessionStorage['dsh:reattach-cordis-runs']` 标志被设置并被 ui-cordis 消费后清除）；纯 F5/Ctrl+R 也触发刷新且保留热插件，Ctrl+Shift+R 不设标志。
- [ ] **接线完整性**：`cordis.patch.yml`（base/web-app）插件行、bundle `package.json` 依赖、`tsconfig.host.json`/`tsconfig.client.json` 引用、`api/remotes` 挂载齐全；`pnpm install && pnpm run build` 通过；`command-guard/decision` 登记在 `KNOWN_SESSION_EVENT_TYPES`；`approval/requested` mux 帧 schema 含可选 `severity: 'danger'`。
- [ ] **包测试**：guard 包 `tests/`（12 个 spec：analyzer/audit/engine/fingerprint/git/index/lexer/model-check/preview/protected/tiers/verbs）通过（复刻时至少覆盖四档分类、模式门控、三问映射、审计轮转、git 分派）。
