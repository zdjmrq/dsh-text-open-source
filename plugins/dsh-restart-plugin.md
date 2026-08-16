# 【文字开源】dsh-restart-plugin — DSH 设置页一键「关闭后台服务 / 刷新前端」插件

> **这是什么**：本文件是「文字开源」描述，不是代码。把本文件全文粘贴给你 DSH 里的 AI 会话，
> AI 即可为你复刻出功能相同的插件；你也可以阅读本文件理解其原理，并自行微调需求。
>
> **对应源码**：https://github.com/zdjmrq/dsh-restart-plugin.git （描述基于 commit `bce96b5779943ca9e241a44aa03d7715613e2fbd`）｜许可证：MIT
>
> **最近更新**：2026-08-16（描述撰写基于仓库最近一次提交日期）

## 1. 插件概述

这是一个 DeepSeek Harness（DSH）网页插件：在 **设置 → 通用设置** 中新增两行一键操作——「关闭后台服务」与「刷新前端」。前者让后台进程**优雅退出**（不建进程、不解析脚本，路径上无可失败环节），并在对话框里展示手动重启命令；后者重新加载页面且**保留创造模式的热插件**（动态插件的客户端半边重新挂载，Host 半边原样不动），后台不重启。核心价值是解决网页端"想关后台 / 想刷新又怕丢热插件"的操作痛点，尤其适合页面卡死时无需打开设置、直接按 F5/Ctrl+R 即可刷新并保留热插件的场景。本插件按**源码方式**集成进 DeepSeek Harness 工作区（基于 `0.1.0-rc.5` 结构），与 [dsh-plugin-suite](https://github.com/zdjmrq/dsh-plugin-suite)（局部 fork 套件，内置本插件与 `dsh-careful-full-access` 命令守卫的累计补丁）及官方上游 [deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) 配套。

## 2. 功能规格

用户可感知的行为清单（可验收）：

- **功能 A：关闭后台服务**（设置 → 通用设置 →「关闭后台服务」行 →「关闭」按钮）
  - 弹出**置顶**确认框（`createPortal` 到 `document.body`，z-index 2000）：确认按钮黑底白字、取消按钮白底黑字；点击背景遮罩可关闭（仅限确认阶段、非忙碌时）。
  - 确认后：调用 Host Remote `restart()` → 应答浏览器后延时 **600ms** 调用启动器 `appExit(0)` 优雅退出 → 对话框进入「正在关闭…」阶段并展示**手动重启命令**提示（`"<node路径>" <启动参数...> (cwd: <工作目录>)`）。
  - 后台退出后页面自动断开（前端额外 best-effort 执行 `window.close()`，失败无碍——页面会自行断开）。
  - 之后由用户自己在终端重新运行该命令、重新打开页面。**热插件会随后台关闭而消失**（确认框文案已提示）。
  - 失败路径：a) 本进程生命周期内已发起过一次关闭请求 → 返回 `restart already in progress`；b) 启动器未暴露 `appExit` 服务 → 返回 `the launcher exposes no graceful exit; stop the backend manually`；c) Remote 传输层失败 → 对话框进入「关闭失败」阶段展示错误信息。
- **功能 B：刷新前端**（设置行「刷新前端」→ 确认框 → 确认）
  - 确认时写入一次性 `sessionStorage` 标记（`dsh:reattach-cordis-runs = '1'`），随后 `window.location.reload()`。
  - 页面重载后由 **ui-cordis**（`@deepseek-ai/dsh-extensions-ui-cordis`）消费该标记：对每个「有 active run 且该包有客户端半边、且无待审批、且尚未加载」的动态插件调用已有的"附加"运行路径 `runner.startUserRun({ mode: 'run', ... })`，**仅恢复客户端界面，不重启 Host 半边**。标记一次性消费（读取后即 `removeItem`）。
  - **F5 与 Ctrl+R 行为等同**：按下无修饰键的 F5，或 Ctrl+R（不含 Alt/Meta/Shift）时，同样先写标记、`preventDefault()` 后由页面代码强制 `location.reload()`——因此即使浏览器默认 F5 行为被桌面壳/内嵌视图抑制，刷新仍生效；页面卡死时无需打开设置即可刷新并保留热插件。**Ctrl+Shift+R（绕过缓存的强制刷新）刻意保留原生行为**，不写标记。
  - 后台不重启。
- **功能 C：连接重置处理**：监听 `connection/reset` 事件（后端已重启/重连），自动关闭/重置对话框状态。
- **功能 D：双语本地化**：zh/en 两套文案，注册到 locale 命名空间 `settings.restart`（通过 `LocaleNamespaceMap` 声明合并）。
- **功能 E：防重复提交**：对话框 `busy` 状态下确认按钮不再响应；Host 端 `armed` 标志保证一次进程生命周期内只执行一次退出序列。

## 3. 技术路线

- **插件形态**：双半（Host 半 + Client 半），两个独立 npm 包，均为 Cordis 插件、按 DSH 标准包结构组织：
  - `@deepseek-ai/dsh-host-restart`（Host 半，`packages/host/restart/`）：提供 `restart` Remote 服务；
  - `@deepseek-ai/dsh-client-ui-settings-restart`（Client 半，`packages/client/ui-settings-restart/`）：设置行 + 置顶对话框 + 快捷键处理。
- **加载与集成方式**：**源码方式安装**（非已发布 npm 包），通过 `install.patch` 一次性修改 DSH 工作区 6 处：
  1. `packages/bundle/web-app/cordis.patch.yml`：注册两行插件——Host 行 `id: restart / name: '@deepseek-ai/dsh-host-restart'`，Client 行 `id: ui-settings-restart / name: '@deepseek-ai/dsh-client-ui-settings-restart'`；
  2. `packages/api/remotes/`：`package.json` 加依赖、`tsconfig.client.json` 加项目引用、`src/client/index.ts` 导入 `restartRemote from '@deepseek-ai/dsh-host-restart/remote'` 并加入 `ctx.remote.$mount(...)` 贡献列表、导出 `RestartAck` 类型——这是把 Host Remote 暴露给浏览器的"装配层"；
  3. `packages/extensions/ui-cordis/src/client/index.ts`：打补丁加入 `REATTACH_FLAG` 常量与 `reattachArmed()`，并在 inventory 订阅回调里加入一次性重附加逻辑（见第 5 节）；
  4. `tsconfig.host.json` 加 `./packages/host/restart` 引用；
  5. `tsconfig.client.json` 加 `./packages/client/ui-settings-restart` 引用；
  6. `packages/bundle/web-app/package.json`：两个包加入 workspace 依赖。
- **依赖的核心 Service / Event / API**（名称以仓库代码为准）：
  - Host：`ctx.get('appExit')`（可选服务，来自宿主 launcher，签名 `(code: number) => void`）；`@deepseek-ai/dsh-typert-protocol` 的 `TypertRemoteService` 基类 + `@Remote('restart')` 装饰器；`@deepseek-ai/dsh-invariants` 的 `ctx.invariants.register`（空 invariant 伴随插件，仅为占位注册包所有权）。
  - Client：`ctx.slots.inject / ctx.slots.register`（slot `settings.general.item`、`shell.overlay`）；`ctx.locale.register`（命名空间 `settings.restart`）；生成式 Remote 面 `ctx.remote.restart.restart()`（返回 `Promise<RemoteResult<RestartAck>>`）；事件 `ctx.on('connection/reset')`；浏览器 `sessionStorage` / `window.location.reload` / `window.addEventListener('keydown')`。
  - ui-cordis（被补丁扩展）：`inventory.subscribe`（共享 Host 库存）、`runner.isLoaded(pluginId)`、`runner.startUserRun({ agentId, pluginId, packageId, mode: 'run', hasClientHalf: true })`。
- **关键外部依赖**：Node.js 22+、pnpm（monorepo）；构建工具 `tsdown`（Host 侧两个 entry 打 ESM/node/es2024 包；Client 侧用 harness 提供的 `clientBundle()` 辅助）；Client 运行时 React 18.2 + react-dom 18.2；Host 包声明了 `zod ^4.4.3` 依赖（typert 协议栈的常规依赖）；typert 契约由 `@deepseek-ai/dsh-typert-generator` 从 FaceModel 生成，生成物 `lib/typert.*` 已随仓库附带，通常无需重新生成。
- **与其他 dsh-\* 仓库的配套关系**：`dsh-plugin-suite`（套件 fork，内置本插件与 `dsh-careful-full-access` 的完整累计补丁，clone 上游 → `git checkout 47f943859b` → `git apply install.patch` 一键获得全部功能）；本仓库的 `install.patch` 仅用于把「关闭/刷新前端」单独装进其它 harness 工作区。官方上游 `deepseek-harness` 为集成目标。

## 4. 结构设计

仓库文件布局（`lib/` 为已编译产物，随仓库附带；`src/` 为源码）：

| 路径 | 职责 |
| --- | --- |
| `install.patch` | 对 DSH 工作区的集成补丁（web-app 组合注册、api-remotes 挂载、ui-cordis 重附加、tsconfig 引用） |
| `packages/host/restart/src/index.ts` | Host 半核心：`RestartGateway`（extends `TypertRemoteService`），`@Remote('restart') restart()` 方法、`launchHint()`、`armed` 标志、`EXIT_DELAY_MS` |
| `packages/host/restart/src/types.ts` | `RestartAck` 契约（`{ ok: boolean; message: string }`） |
| `packages/host/restart/src/invariant.ts` | Host 包 invariant 伴随插件（空实现，注册包所有权） |
| `packages/host/restart/package.json` | 包元数据、exports 映射（`.` `/invariant` `/types` `/typert` `/remote` `/src/*`）、构建脚本 |
| `packages/host/restart/tsdown.config.ts` | 把 `lib/types/index.js`、`lib/types/invariant.js` 打成独立 ESM bundle |
| `packages/host/restart/lib/` | 编译产物：`index.js`、`invariant.js`、`types/`、`typert.host.*`、`typert.remote-client.*`（生成契约） |
| `packages/client/ui-settings-restart/src/index.ts` | Client 包的 Host loader 入口（空 `apply`；Remote 本体在 host 包） |
| `packages/client/ui-settings-restart/src/client/index.ts` | Client 半主入口：locale 注册、`REATTACH_FLAG` 导出、F5/Ctrl+R keydown、`runRestart`/`runRefresh`、两个设置行 + 对话框的 slot 注册 |
| `packages/client/ui-settings-restart/src/client/RestartRow.tsx` | 设置行组件（标题/描述/操作按钮） |
| `packages/client/ui-settings-restart/src/client/RestartDialog.tsx` | 置顶确认/进行中/错误三态对话框（`createPortal` → `document.body`） |
| `packages/client/ui-settings-restart/src/client/restart-store.ts` | 微型 observable 共享状态（`createRestartUiStore`：open/kind/phase/error/detail） |
| `packages/client/ui-settings-restart/src/client/locales.ts` | zh/en 字典，`RestartKey = keyof typeof zh` |
| `packages/client/ui-settings-restart/src/client/*.module.css` | 行与对话框样式（CSS Modules + `--dsw-alias-*` 主题变量） |
| `packages/client/ui-settings-restart/src/invariant.ts` | Client 包 invariant 伴随插件（空实现） |
| `packages/client/ui-settings-restart/src/css-modules.d.ts` | `*.module.css` 类型声明 |
| `packages/client/ui-settings-restart/package.json` | 包元数据、`dsh.client.inject` 平台声明（platform: web）、peerDependencies |
| `packages/client/ui-settings-restart/tsdown.config.ts` | 调用 harness 的 `clientBundle()` 打包 |

模块调用关系：

```
[用户点击设置行按钮]
      │ openConfirm(kind)  （store.openConfirm）
      ▼
[RestartDialog (shell.overlay slot, portal→body, z-index 2000)]
      │ 确认
      ├─ kind=restart ─► runRestart(): ctx.remote.restart.restart() ──RPC──► [Host RestartGateway.restart()]
      │                                                                         ├─ armed 检查 / appExit 检查
      │                                                                         ├─ 600ms 后 appExit(0)（优雅退出）
      │                                                                         └─ 返回 RestartAck{ok,message}（含重启命令 hint）
      │                    └─ 对话框 → 'restarting' 阶段展示 message；页面随后断开
      └─ kind=refresh ─► runRefresh(): sessionStorage 写 REATTACH_FLAG + location.reload()
                              ▼ 页面重载
                        [ui-cordis inventory.subscribe] 消费标记（removeItem 一次性）
                              ▼ 对满足条件的活跃动态插件
                        runner.startUserRun({mode:'run', hasClientHalf:true}) → 仅附加客户端半边
```

## 5. 关键实现细节

- **Host 关闭核心逻辑**（`RestartGateway.restart()`，步骤化）：
  1. `if (this.armed) return { ok: false, message: 'restart already in progress' }`——进程生命周期内只放行一次；
  2. `const appExit = this.ctx.get('appExit')`（可选服务；`undefined` 时返回失败文案并**不**置 armed）；
  3. `this.armed = true`；
  4. `setTimeout(() => { appExit(0) }, 600)`——**先应答浏览器、再退出**，保证 RPC 响应能回到前端展示；`appExit` 调用包在 try/catch（失败语义归启动器所有）；
  5. 返回 `{ ok: true, message: 'the dsh backend is shutting down; restart it manually with: <launchHint()>' }`。
- **重启命令提示 `launchHint()`**：用当前进程自身启动事实重建可重跑命令——`process.argv.slice(1)` 的每个参数若含空格则加双引号，拼成 `"${process.execPath}" ${args.join(' ')} (cwd: ${process.cwd()})`。刻意不做进程重启机制（无 spawn/脚本解析），"点一下到退出之间没有任何可失败环节"是设计核心。
- **Client `runRestart()` 的 RemoteResult 解析**：`ctx.remote.restart.restart()` 返回 `RemoteResult<RestartAck>`，结构为 `{ ok: false, error: { message } } | { ok: true, value: RestartAck }`；代码先判 `result.ok`（传输层失败）再判 `result.value.ok`（业务失败），两者都失败时对话框进入 error 阶段展示 `message`；成功后 best-effort `window.close()`（浏览器只允许关脚本打开的窗口，被拦截无妨——页面随后自行断开）。
- **一次性重附加标记的"跨插件字符串字面量"坑**：`REATTACH_FLAG = 'dsh:reattach-cordis-runs'` 在 `ui-settings-restart/src/client/index.ts` 中定义/导出，同时在 `ui-cordis/src/client/index.ts` 补丁里**硬编码同一字符串**——因为**跨插件值导入被 client bundle 的 purity gate 禁止**，两处只能各自保留字面量并靠注释约定同步（升级时若只改一处会静默失配，属已知风险点）。
- **ui-cordis 消费逻辑**（在 `inventory.subscribe` 回调内，`snapshot.read` 为真且 `reattachArmed()` 时执行）：
  1. 先 `sessionStorage.removeItem(REATTACH_FLAG)`（try/catch，存储被拒也不阻塞重附加）——**一次性**，每次页面加载最多消费一次；
  2. 遍历 `snapshot.rows`，跳过：`row.activeRun === undefined`、`row.latestRun?.approvalRequestId !== undefined`（有待审批）、`runner.isLoaded(row.pluginId)`（已加载）、`row.packages.find(p => p.packageId === run.packageId)?.hasClientHalf !== true`（无客户端半边）；
  3. 对剩余行调用 `runner.startUserRun({ agentId, pluginId, packageId: run.packageId, mode: 'run', hasClientHalf: true })`——**mode 'run' 的 attach 路径复用 live run**，因此 Host 半边不被重启。
- **F5/Ctrl+R 快捷键**：window keydown 监听；`isPlainF5 = key==='F5' && 无 ctrl/alt/meta/shift`；`isPlainCtrlR = key.toLowerCase()==='r' && ctrlKey && 无 alt/meta/shift`；命中则 `armReattachFlag()` → `event.preventDefault()` → `window.location.reload()`。**Ctrl+Shift+R 刻意不拦**（保留浏览器缓存绕过式强制刷新，自然也不保留热插件）。effect 返回清理函数移除监听。
- **对话框状态机**：`restart-store.ts` 用 `Set<listener>` 实现微型 observable；phase 三态 `'confirm' | 'restarting' | 'error'`；`busy` 标志防连点；`connection/reset` 事件把对话框 `close()` 复位（后端重启后旧对话框状态无意义）。
- **置顶实现**：`RestartDialog` 用 `createPortal(..., document.body)` 渲染，遮罩 `z-index: 2000`（注释说明高于设置面板 1000、toast 与引导层 1100）；背景点击关闭仅当 `phase==='confirm' && !busy`；卡片 `stopPropagation` 防误关。
- **边界与坑**：
  - `sessionStorage` 写入/读取均包 try/catch——隐私模式/存储被拒时：刷新本身照常进行，只是热插件不保留；
  - 插件行 slot `settings.general.item` 注册两个 id（`restart-backend` order **30**、`refresh-frontend` order **40**，排在其它设置项之后）、对话框注册 `shell.overlay`（id `restart-backend-dialog`，order **0**）；
  - 两个包的 `src/index.ts` 是空 `apply`——Client 包经 harness 客户端装配时只加载 `./client` 半边，Host loader 入口仅是占位（Host Remote 本体在 `@deepseek-ai/dsh-host-restart`）；
  - invariant 伴随插件均为空实现（`install = () => {}`），只因 DSH 包规范要求 `ctx.invariants.register(PACKAGE_NAME, install)` 声明包所有权。
- **配置项/常量一览**：`EXIT_DELAY_MS = 600`（应答到退出的延时）；`REATTACH_FLAG = 'dsh:reattach-cordis-runs'`（标记键，值 `'1'`）；slot order 30/40/0；z-index 2000；Remote 名与方法名均为 `'restart'`（typert 全路径 `restart/restart`）。
- **与 DSH 版本相关的耦合点**：`appExit` 服务来自宿主 launcher（宿主内部契约，缺失时降级为报错）；`TypertRemoteService`/`@Remote` 与 `RemoteResult` 结构来自 `@deepseek-ai/dsh-typert-protocol`（版本敏感）；typert 生成物 `lib/typert.*` 随仓库附带、`packages/client/tsdown.client.ts` 的 `clientBundle()` 辅助来自 harness；**补丁直接改了 harness 内部文件**（`ui-cordis` 的 client 入口、`api/remotes` 的 client 入口、web-app 的 `cordis.patch.yml`），这些位置会随上游版本漂移——README 明确"如版本有出入请对照补丁手工合并"，本插件按 `0.1.0-rc.5` 结构集成。

## 6. 集成与安装

从拿到本描述到插件生效的完整步骤：

1. **复刻**：把本文件全文粘贴给 DSH 里的 AI 会话，要求其产出两个包源码（`packages/host/restart/`、`packages/client/ui-settings-restart/`）与 `install.patch`，再按第 8 节检查单自验。
2. **放置**：把两个包目录复制到 DeepSeek Harness 源码工作区的 `packages/host/` 与 `packages/client/` 下；`install.patch` 放到工作区根目录。
3. **打补丁**：`git apply install.patch`（含 web-app 组合注册、api-remotes 挂载、ui-cordis 热插件重挂载与 tsconfig 引用；版本有出入时对照补丁手工合并）。
4. **安装与构建**（工作区根目录）：
   ```powershell
   pnpm install
   pnpm exec tsc -b tsconfig.host.json
   pnpm exec tsc -b tsconfig.client.json
   pnpm --filter @deepseek-ai/dsh-host-restart bundle
   pnpm --filter @deepseek-ai/dsh-client-ui-settings-restart bundle
   pnpm --filter @deepseek-ai/dsh-api-remotes bundle
   pnpm --filter @deepseek-ai/dsh-client-ui-cordis bundle
   ```
   （两个包的 `lib/` 产物含 typert 契约已随仓库附带，通常无需重新生成。）
5. **生效与验证**：手动重启后台（如 `pnpm dsh web`），刷新页面 → 打开 **设置 → 通用设置**，最下方应出现「关闭后台服务」「刷新前端」两行；按第 8 节逐项验收。

## 7. 已知边界与注意事项

- **后台不会自动重启**：关闭后必须由用户手动在终端重跑提示的命令；无进程守护/自拉起能力（这是刻意的设计取舍）。
- **刷新只恢复客户端半边**：动态插件的 Host 半边在刷新前后原样不动；若想重启插件 Host 逻辑需另行操作。
- **有 pending approval 的插件不会被重附加**（`approvalRequestId` 判定），需用户另行处理审批。
- **Ctrl+Shift+R 是原生强制刷新**：不写标记、不保留热插件（预期行为）。
- **`sessionStorage` 被拒绝时**（隐私模式等）：刷新本身仍执行，但热插件不保留（try/catch 降级）。
- **`window.close()` 仅对脚本打开的窗口生效**：被浏览器拦截无碍，页面随后自行断开。
- **版本漂移风险**：按 `0.1.0-rc.5` 结构集成；`install.patch` 直接改 harness 内部文件（ui-cordis、api/remotes、web-app 组合、tsconfig），上游升级后需手工合并。
- **两处硬编码的 `REATTACH_FLAG` 字面量**必须保持一致，否则刷新后热插件静默不保留。
- 未以 npm 发布形式安装，也没有独立于 harness 的运行时；`lib/` 为已提交产物（含 typert 契约）。

## 8. 复刻验收标准（给复刻 AI 的检查单）

- [ ] **设置行出现**：启动后 设置 → 通用设置 最下方有两行「关闭后台服务」「刷新前端」（slot `settings.general.item`，order 30/40）。
- [ ] **关闭流程**：点「关闭」→ 置顶对话框（黑确认/白取消）→ 确认 → 「正在关闭…」阶段展示含 `(cwd: ...)` 的手动重启命令 → 后台进程优雅退出、页面断开；重启后手动命令可恢复服务。
- [ ] **防重复**：进程生命周期内第二次调用返回 `restart already in progress`；启动器无 `appExit` 时返回明确失败文案并进入「关闭失败」态。
- [ ] **刷新流程**：点「刷新」→ 确认 → 页面重载 → 活跃且有客户端半边的动态插件界面恢复，Host 半边未重启（可用插件运行状态佐证）。
- [ ] **快捷键**：无修饰键 F5 与 Ctrl+R 直接刷新并保留热插件；Ctrl+Shift+R 为原生强制刷新（不保留）。
- [ ] **一次性标记**：`sessionStorage['dsh:reattach-cordis-runs']` 每次页面加载至多消费一次（读取后即移除）。
- [ ] **连接重置**：`connection/reset` 事件触发后对话框复位关闭。
- [ ] **置顶与样式**：对话框 `createPortal` 到 `document.body`、z-index 2000；确认黑底白字、取消白底黑字；busy 时不可重复提交、背景点击仅在确认阶段可关闭。
- [ ] **本地化**：`settings.restart` 命名空间 zh/en 字典齐全（行、对话框、错误、进行中文案均覆盖）。
- [ ] **集成点一致**：`cordis.patch.yml` 两行插件、`api/remotes` 挂载 `restartRemote` 并导出 `RestartAck`、`ui-cordis` 消费标记、`tsconfig.host.json`/`tsconfig.client.json` 引用、web-app 依赖齐全；构建命令按第 6 节可复现。
