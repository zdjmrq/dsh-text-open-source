# 【文字开源】dsh-intercom — DSH 顶层对话之间的通信中心(私聊、协作群、休眠唤醒、模型工具)

> **这是什么**：本文件是「文字开源」描述，不是代码。把本文件全文粘贴给你 DSH 里的 AI 会话，
> AI 即可为你复刻出功能相同的插件；你也可以阅读本文件理解其原理，并自行微调需求。
>
> **对应源码**：https://github.com/zdjmrq/dsh-intercom （描述基于 commit `8c3b0ef3982e0f1040bc30783a14769870d65127`）｜许可证：MIT
>
> **最近更新**：2026-08-16

## 1. 插件概述

dsh-intercom 解决 DeepSeek Harness 里**多个顶层对话（父代理）之间无法互相发现与协作**的问题：每个对话是一个独立 Agent，默认彼此看不见。本插件在宿主侧注册一个 Typert Remote 服务 `ctx.remote.intercom` 和 13 个 `intercom_*` 模型工具，在网页侧提供一个聊天式「通信中心」面板，让对话可以：发现其他活跃对话（含工作区与忙碌状态）、私聊投递（唤醒/排队/介入三种模式）、组建协作群（对话群）并广播、发现并**唤醒休眠的历史对话继续工作（延续工作）**。核心价值是让对话在需要时**自动协调其他对话**——请求帮助、并行合作、延续旧工作，而无需人类手工传话。

适用场景：多对话并行推进同一项目时的分工与汇总；向另一个正在跑的对话发协作请求并等待回执；把一个历史会话拉起来继续它未完成的任务。它与用户的 `dsh-plugin-suite`（局部 fork 套件，含 `dsh-restart-plugin`、`dsh-careful-full-access`）以及本「文字开源」枢纽中的其他 `dsh-*` 描述文件配套：intercom 是这些对话之间的“通信总线”。

**边界约定（不可破坏）**：intercom 只服务**顶层会话之间**；父↔子代理通信完全交给内置 `send_message`/`list_agents`，本插件不拦截、不越权；跨工作区投递默认拒绝。

## 2. 功能规格

### 面板（Client 半）

- **通信中心入口**：侧栏底部「通信中心」按钮（链接图标 + 文字，`sidebar.footer.action` slot）与每个会话头部图标按钮（`conversation.session.header.actions` slot）。点击开关面板。
- **双栏聊天面板**：注册于 `shell.overlay` slot 的悬浮面板（宽 680px、高 min(76vh,560px)，DSH 主题色 `--dsw-alias-*` 深浅色自适应）。左侧两个 Tab：**会话**（活跃列表 + 休眠列表）与**群聊**；右侧为消息气泡流 + 底部输入框 + 投递模式下拉（唤醒/介入）。
- **活跃会话列表**：触发条件=打开面板或每 3 秒轮询 → 调 `remote.intercom.list()` → 列出全部活跃顶层对话，每项显示标题、忙碌/空闲徽标、📁 工作区路径；点击选中进入私聊视图，可查看该会话最近 80 条表面消息（每 2 秒轮询 `readConversation`）。
- **休眠会话列表**：触发条件=同轮询 → 调 `remote.intercom.dormant()` → 灰显列出所有持久化但未运行的顶层历史会话（💤 休眠徽标、标题、工作区、按创建时间倒序）。选中后输入框提示「该会话休眠中，发送将唤醒它…」，按钮变为「**唤醒并发送**」。
- **唤醒并发送**：对休眠目标发送 → `remote.intercom.wakeSend()` → 宿主恢复该会话 Agent 并投递 → 面板刷新，该会话移入活跃列表。
- **群聊视图**：列出全部群（含自动群，显示人数徽标）；可新建群（输入名称）、查看成员、从活跃会话下拉中添加成员、移除成员、广播消息；群记录合并阅读（每条标出来源成员标题）。
- **快捷按钮**：面板头部「＋ 新会话」（`workspaces.startSession()`）与「Fork」（`sessions.fork({sessionId})` 后打开新会话）——用于快速制造另一个可协作的顶层对话。

### 模型工具（Host 半，13 个 `intercom_*`）

1. `intercom_list_conversations`：列出全部**活跃**顶层对话（id/标题/状态/cwd）。
2. `intercom_list_dormant_conversations`：列出全部**休眠**顶层对话（id/标题/cwd/createdAt，倒序）。
3. `intercom_send`：向指定活跃顶层对话发消息；`delivery=wake`（空闲即唤醒开工，忙碌排队）或 `steer`（显式介入当前回合，慎用）。
4. `intercom_wake_send`：向休眠对话发消息——先 `agents.resume` 恢复该会话，再按 wake/steer 投递（延续工作）。
5. `intercom_ask`：发送 + 有界等待回复（wait_ms 上限 30000），返回目标的新回复文本。
6. `intercom_check_replies`：收集目标会话自上次发信以来的新内容（since_time=0 时自动取上次 intercom 消息时间戳）。
7. `intercom_read_conversation`：只读地读另一个会话的最近表面记录（user 任务 + assistant 结论），**冷会话也可读**，用于延续工作前的状态提取。
8. `intercom_collect`：并行收集一批会话的当前状态与最后结论（合作汇总）。
9. `intercom_spawn_conversation`：创建**持久化的子代理**新对话并立即开工（继承发起者的 workspace/provider/model）；返回的是子代理，后续用内置 `send_message`/`list_agents` 管理。
10. `intercom_create_group`：以逗号分隔的成员 id 建显式协作群。
11. `intercom_broadcast`：向群内除发起者外的全部成员投递。
12. `intercom_list_groups`：列出全部群（含自动群与成员数）。
13. `intercom_read_group`：合并阅读群内各成员自 since_time 以来的对话内容（每段标成员标题）。

### 消息投递语义（面板与工具共用）

- 投递模式：`wake`（默认）→ 目标**空闲**且有唤醒预算时 `agent.followup` 立即开工；否则 `agent.inject` 排队到下一步；`steer` → `agent.steer` 显式介入当前回合。
- 防护：限频 10 条/分钟/目标；唤醒预算 3 次/目标（目标收到**人工输入**时重置）；禁止发给子代理、禁止发给同一会话；来源包装定型防提示注入（见 §5）。
- 群持久化：群数据（含自动群成员）经 storage-domain 全局槽落盘，**重启不丢**；自动群「协作中的对话(自动)」在每次通信时把双方加入。

## 3. 技术路线

- **插件形态**：**双半**（Host + Client 两个独立 workspace 包），Cordis 插件体系。
  - Host 半：`@deepseek-ai/dsh-host-intercom`（`packages/host/intercom`）——一个 `Service` 类（默认导出），`export class IntercomGateway extends TypertRemoteService`，构造器 `super(ctx, 'intercom')` 即注册 Remote 服务。
  - Client 半：`@deepseek-ai/dsh-client-ui-intercom`（`packages/client/ui-intercom`）——浏览器 UI 插件（React.createElement，无 JSX），`inject = ['slots','locale','remote','remote.intercom','sessions','workspaces']`。
- **加载与集成方式**：两包都是 npm workspace 成员（glob `packages/*/*` 自动收录），通过 `packages/bundle/web-app/cordis.patch.yml` 补丁层注册组合行：宿主行 `{id: intercom, name: '@deepseek-ai/dsh-host-intercom'}`、客户端行 `{id: ui-intercom, name: '@deepseek-ai/dsh-client-ui-intercom'}`。客户端包同时声明 `dsh.client` 元数据（`platform: "web"` + `inject` 依赖面），由 `client-modules` 服务扫描后提供 `/plugins/<包名>/client.js` bundle 路由。Remote 客户端经 `api-remotes` 挂载：`packages/api/remotes/src/client/index.ts` 中 `import intercomRemote from '@deepseek-ai/dsh-host-intercom/remote'` 并加入 `ctx.remote.$mount(...)` 列表。
- **依赖的核心 Service / Event / Tool**（全部经 `ctx.get` 或注入，名称需与 DSH 0.1.0-rc.5 一致）：
  - `static inject = ['tools', 'storageDomain']`（宿主服务声明，见 §5 时序坑）；
  - `tools`：`ctx.tools.register(defineTool({...}))` 注册 13 个模型工具（`defineTool` 来自 `@deepseek-ai/dsh-tools`）；
  - `storageDomain`：`defineDomain` + `facility.open(intercomDomain)` 做群持久化；
  - `agents`：`get/list/roots/create/resume` 管理活跃代理与恢复休眠会话（`resume({resumeSessionId})` 返回 `{agent}`）；
  - `sessionQuery`：`readSurface(sessionId)`（读表面事件做历史/回复轮询）与 `readTitle(sessionId)`（休眠会话标题）；
  - `sessionPersistence`：`list()` 枚举全部已物化会话头（id/parentSession/origin/cwd/createdAt）用于休眠列表与唤醒前校验；
  - `sessionTitle`：`get(session)` 取活跃会话折叠标题；
  - `subagents`：`list/getProvider/startContinuable` 用于 `intercom_spawn_conversation`（选带 `prepareContinuable` 的 provider）；
  - Event：`agent/inbox/claimed`（目标收到 `kind:'user'` 的人工消息 → 重置其唤醒预算）、`agent/disposed`（清理该代理的限频桶与发信记录）；
  - Remote 契约经 **Typert**：`@Remote('list' | 'dormant' | 'groups' | 'send' | 'wakeSend' | 'broadcast' | 'readConversation' | 'readGroup' | 'createGroup' | 'addMember' | 'removeMember')` 装饰器声明 11 个方法；`lib/typert.host.{js,d.ts}` 与 `lib/typert.remote-client.{js,d.ts}` 由上游 `@deepseek-ai/dsh-typert-generator` 的 tsdown 插件在工作区 host pass 生成（见 §5）。
- **关键外部依赖**：zod `^4.4.3`（domain schema）；peer：`@deepseek-ai/cordis`、`@deepseek-ai/dsh-storage-domain`、`@deepseek-ai/dsh-tools`、`@deepseek-ai/dsh-typert-protocol`（host）；client 侧 peer 再加 `@deepseek-ai/dsh-api-remotes`、`dsh-client-{locale,runtime,ui-conversation,ui-layout,ui-slots}` 与 `react`/`react-dom` `^18.2.0`。
- **与其他 dsh-* 仓库的配套关系**：依赖上游 DeepSeek Harness 源码工作区（按 `0.1.0-rc.5` 结构集成）；与 `dsh-plugin-suite`（套件补丁）、`dsh-restart-plugin`（重启后验证恢复）、`dsh-careful-full-access`（文件权限）互不依赖但同属用户的定制插件集。

## 4. 结构设计

| 路径 | 职责 |
| --- | --- |
| `packages/host/intercom/src/index.ts` | IntercomGateway 服务类：11 个 Remote 方法、投递核心 `deliverTo`、休眠唤醒、13 个模型工具注册、事件监听、群/域持久化 |
| `packages/host/intercom/src/types.ts` | 全部线上契约类型（`type` 别名：ConversationInfo/GroupInfo/MessageEntry/SendRequest/SendResult/WakeSendRequest/WakeSendResult/DormantConversation/…），被 Remote 参数/返回与工具 payload 共用 |
| `packages/host/intercom/src/spec.ts` | storage-domain 声明：`intercomDomain = defineDomain({name:'intercom', version:0, global:{schema: intercomGlobalSchema, initial:{groups:{}}}, tables:{}})`，全局槽只存 `{groups: Record<groupId, {name, members[]}>}` |
| `packages/host/intercom/lib/` | 构建产物：`index.js`（服务主包）、`spec.js`/`types.js`、`typert.host.{js,d.ts}`（TYPERT 注册）、`typert.remote-client.{js,d.ts}`（客户端远程代码 + 命名空间类型）、`types/*.d.ts` |
| `packages/host/intercom/tsconfig.json` + `tsdown.config.ts` | tsc 编译到 `lib/types`；tsdown 以 `lib/types/{index,types,spec}.js` 为入口打包 esm |
| `packages/client/ui-intercom/src/index.ts` | 空 Host 半 apply（Node 半仅存在以符合双面约定） |
| `packages/client/ui-intercom/src/client/index.ts` | 浏览器半：图标/样式表（`ctx.effect` 注入 `<style data-plugin>`）、三个 slot 注册、IntercomPanel 组件（状态、轮询、发送/建群/加成员等交互） |
| `packages/client/ui-intercom/tsdown.config.ts` | 一行委托：`clientBundle('@deepseek-ai/dsh-client-ui-intercom', ['lib/types/index.js'])`（复用上游共享预设） |
| `packages/client/ui-intercom/lib/` | `index.js`（Node 半）与 `client.js`（`window.__ModuleLoader__.load({id, factory})` 包装的 CJS 浏览器包 + sourcemap） |
| `install.patch` | 8 文件接线 diff（见 §6） |
| `README.md` / `LICENSE` / `.gitattributes` | 文档 / MIT / 强制 `install.patch` 保持 LF |

```
┌─ Host（Node 进程）────────────────────────────────────────────┐
│ IntercomGateway (TypertRemoteService, inject:[tools,storageDomain])
│   ├─ @Remote ×11 ──► typert 注册表 ──► api-gateway ──► 浏览器 WS
│   ├─ deliverTo: 工作区门禁 → 限频 → followup/inject/steer
│   ├─ wakeSendInternal: persistence.list 校验 → agents.resume → deliverTo
│   ├─ groupStore(Map) ←─ loadDomain/persist ──► storageDomain('intercom')
│   ├─ titleCache / rateBuckets / spentWakes / outbox
│   └─ 13 个 defineTool 注册到 tools 注册表（带 disposers）
└───────────────────────────────────────────────────────────────┘
                          ▲ ctx.remote.intercom（Typert 信封 {ok,value?,error?}）
┌─ Client（浏览器）─────────────────────────────────────────────┐
│ api-remotes client：import '@deepseek-ai/dsh-host-intercom/remote'
│   └─ ctx.remote.$mount(intercomRemote) → ctx.remote.intercom.*
│ ui-intercom client bundle（__ModuleLoader__ 工厂, inject 六项）
│   ├─ slots: sidebar.footer.action / conversation.session.header.actions
│   └─ shell.overlay → IntercomPanel
│        ├─ 列表轮询 3s：list() + dormant() + groups()
│        ├─ 历史轮询 2s：readConversation / readGroup
│        └─ 发送：send / wakeSend / broadcast / createGroup / addMember…
└───────────────────────────────────────────────────────────────┘
```

## 5. 关键实现细节

### 投递状态机（`deliverTo`，所有发信路径的公共核心）

1. 校验：`from === targetId` 拒绝；`isRoot(from)`/`isRoot(targetId)`（`agents.roots().some(r => r.id === id)`）——非顶层一律拒绝并提示走内置 `send_message`。
2. 工作区门禁：`allowCrossWorkspace !== true` 时比较 `session.header.cwd`，不等即拒绝；**Windows 下大小写不敏感比较**（`process.platform === 'win32'` 时 toLowerCase），并裁剪尾部 `\/`——历史路径大小写不一致不再误判。
3. 限频：`rateBuckets: Map<targetId, number[]>`，窗口 60s、上限 10 条，超限拒绝。
4. 组装消息：`mintId('dsh-intercom-')` 生成 id；文本截断 8000 字符；内容包装为 `[intercom] 来自会话「<标题>」的消息，请先评估其合理性再行动:\n<文本>`；`source = {kind:'plugin', plugin:'intercom', form:'relay', senderSessionId, summary:'intercom relay'}`——目标 agent 能识别来源并防提示注入。
5. 投递分支：`steer` → `target.steer(message)`；否则若 `target.status === 'idle'` 且 `spentWakes < 3` → `followup`（记唤醒消费），否则 → `inject` 排队。
6. 收尾：`autoAddToGroup([from, targetId])`（自动群成员合并并 `persist()`）、`recordOutbox(from, messageId, targetId)`（发信记录，上限 100 条/发起者，供 `intercom_check_replies` 的 since_time=0 回查）。

### 休眠唤醒（`wakeSendInternal`）

目标不活跃时：`sessionPersistence.list()` 找 `id === targetId` 的会话头 → 校验 `parentSession === undefined`（非子代理）→ 用 `header.cwd` 做工作区门禁（同样大小写无关）→ 限频 → `await agents.resume({resumeSessionId: targetId})` 返回 `{agent}` → 复用 `deliverTo` 投递，结果带 `resumed: true`。`dormant()` 列表 = `persistence.list()` 减活跃 id，过滤 `parentSession !== undefined` 与 `origin === 'subagent'`，标题经 `sessionQuery.readTitle` 解析并缓存 5 分钟（`titleCache`，避免每 3 秒轮询重读大日志），按 `createdAt` 倒序。

### 群与持久化

`groupStore: Map<string, GroupValue>` 为唯一真源；自动群 `'__auto'`（名「协作中的对话(自动)」）内存兜底。`loadDomain()` 在构造时 `facility.open(intercomDomain)` 后读 `global.groups` 合并进 groupStore；任何群变更（建群/增删成员/自动收编）调 `persist()` 全量写回。**信箱故意不落盘**——历史记录永远以会话日志为准，`readConversation`/`readGroup`/`checkReplies` 都实时从 `sessionQuery.readSurface` 读回（过滤 `event.time > sinceTime`，只取 `user/message`、`assistant/message` 的文本块）。`readGroup` 对每个成员并查表面并按时间混排、条目带 `memberTitle`。storage-domain 缺失时降级为纯内存并打 `[intercom]` 警告。

### 踩过的坑（复刻时务必遵守）

- **`static inject = ['tools', 'storageDomain']` 不可省略**：最初没有依赖声明，服务 apply 可能先于两个提供方执行，守卫静默跳过工具注册与群加载（启动日志 `[intercom] tools/storageDomain service unavailable`），表现为“重启后工具消失”。声明 inject 后 Cordis 会等待就位再 apply。
- **工具注册硬约束**（`ctx.tools.register(defineTool({...}))`）：所有 `parameters` 字段必须 `required: true`；`execute` 必须是 `async` 返回 Promise；`output.schema` 必须 `additionalProperties: false` 且**声明每个返回字段**；线上类型必须是 `type` 别名而非 `interface`（否则过不了 JsonValue 检查）。13 个工具的 disposers 收进数组并在 `ctx.effect(() => () => dispose(), ...)` 里统一回收。
- **typert 工件必须由上游生成器产出**：在 harness 工作区内跑根级 `pnpm exec tsdown --env.DSH_BUILD_FACE host`（根 tsdown 配置注入 `typertPlugin({mode:'workspace', faces:['host']})`），它会为所有含 Remote 的包生成 `lib/typert.host.*` 与 `lib/typert.remote-client.*`；手写这些文件极易与生成器格式漂移。`lib/typert.host.d.ts` 只需 `export declare const TYPERT: unknown`（生成器风格），方法级类型在 `typert.remote-client.d.ts`。
- **客户端 bundle 包装**：客户端包 tsdown 配置必须委托上游 `clientBundle` 预设（`packages/client/tsdown.client.ts`），它产出 `window.__ModuleLoader__.load({id, factory})` 的 CJS 包并跑“bundle 纯净性门禁”（跨插件值导入直接构建失败；类型导入被擦除不受限）。用裸 `defineConfig` 重建会丢掉包装，浏览器加载即失效。
- **api-remotes 客户端 bundle 必须重建**：新增 Remote 方法后，`packages/api/remotes/lib/client.js` 里内联的是旧版 remote 契约，不重建则浏览器侧根本没有新方法。
- **会话工具表在会话创建时冻结**：装好插件后，**新开的对话**才有 `intercom_*` 工具；旧对话（含重启恢复的）看不到，属预期而非故障。
- **常量**：`MAX_TEXT=8000`、`MAX_WAKES=3`、`RATE_LIMIT=10`（/60s）、面板轮询 3s/历史 2s、readConversation 默认 `maxEvents=80`、工具文本输出截断 16000。

## 6. 集成与安装

把本描述粘贴给 DSH 会话后，让它产出（或你手工产出）以下内容：

1. **两个包目录**（按 §4 布局）：`packages/host/intercom` 与 `packages/client/ui-intercom`，复制进你的 DeepSeek Harness 源码工作区对应目录（`packages/host/`、`packages/client/`）。两包的 `lib/` 产物随仓库附带，可先直接使用。
2. **接线补丁 `install.patch`**（对工作区根执行 `git apply`，共 8 文件）：
   - `packages/api/remotes/package.json`：dependencies/peerDependencies 加 `@deepseek-ai/dsh-host-intercom: workspace:^`；
   - `packages/api/remotes/src/client/index.ts`：`import intercomRemote from '@deepseek-ai/dsh-host-intercom/remote'`、`export type {} from '...'`、并把 `intercomRemote` 加入 `ctx.remote.$mount([...])` 数组；
   - `packages/api/remotes/tsconfig.client.json`：references 加 `{path: "../../host/intercom"}`；
   - `packages/bundle/web-app/cordis.patch.yml`：宿主段加 `- id: intercom / name: '@deepseek-ai/dsh-host-intercom'`，客户端段加 `- id: ui-intercom / name: '@deepseek-ai/dsh-client-ui-intercom'`；
   - `packages/bundle/web-app/package.json`：dependencies 加两个包 `workspace:^`；
   - `tsconfig.host.json` references 加 `./packages/host/intercom`；`tsconfig.client.json` references 加 `./packages/client/ui-intercom`；
   - `tsconfig.base.json` paths 加 `"@deepseek-ai/dsh-host-intercom": ["./packages/host/intercom/src"]` 与 `"@deepseek-ai/dsh-client-ui-intercom": ["./packages/client/ui-intercom/src"]`（源码启动模式必需；`install.patch` 必须保持 LF，仓库用 `.gitattributes` 锁死）。
3. **安装与构建**（工作区根，PowerShell）：
   ```powershell
   pnpm install
   pnpm exec tsc -b tsconfig.host.json
   pnpm exec tsc -b tsconfig.client.json
   pnpm --filter @deepseek-ai/dsh-host-intercom bundle
   pnpm --filter @deepseek-ai/dsh-client-ui-intercom bundle
   pnpm --filter @deepseek-ai/dsh-api-remotes bundle
   ```
   （修改过 `src/` 时，typert 契约改跑根级 `pnpm exec tsdown --env.DSH_BUILD_FACE host` + `--env.DSH_BUILD_FACE client` 全量生成。）
4. **重启后台**（`pnpm dsh web`）并刷新页面。
5. **验证**：侧栏出现「通信中心」入口，面板列出活跃会话与 💤 休眠会话；`pnpm exec tsx scripts/verify-cordis-config.ts` 全绿（无 resolution 报错）；新开一个对话，工具列表包含 13 个 `intercom_*`。

## 7. 已知边界与注意事项

- **只服务顶层会话**：子代理完全不进入列表/投递/休眠视图；父↔子走内置 `send_message`/`list_agents`，插件不碰父子通道。
- **跨工作区默认拒绝**：Remote 层有 `allowCrossWorkspace` 显式放行参数，模型工具与面板未暴露；同工作区判定在 Windows 上大小写无关。
- **投递有预算**：限频 10 条/分钟/目标、唤醒预算 3 次/目标（人工输入重置）——批量协作请用广播或分组，不要风暴式连发。
- **休眠列表只覆盖已物化的会话**（`sessionPersistence.list()` 语义：创建后从未写入事件的会话不出现）；标题读取有 5 分钟缓存。
- **唤醒有副作用**：`agents.resume` 恢复的会话若上次有未完成回合会先继续执行，消息随后排队；唤醒后该会话变为活跃，需注意资源占用。
- **工具可见性**：新工具只出现在安装后新建的对话里（工具表在会话创建时冻结）。
- **信箱不落盘**：面板历史永远从会话日志实时读回，不额外持久化消息本身。
- **未完成项/取舍**：面板无消息撤回与已读回执；`intercom_spawn_conversation` 依赖部署中存在带 `prepareContinuable` 的 subagent provider；描述按 commit `8c3b0ef…` 的现状撰写，上游 DSH 版本升级可能导致 Service 签名（如 `agents.resume`、`sessionPersistence.list`）漂移。

## 8. 复刻验收标准（给复刻 AI 的检查单）

- [ ] 面板入口两个 slot 都在，打开面板能看到「会话/群聊」双 Tab 与活跃会话列表（含状态徽标与 📁 工作区）。
- [ ] 休眠会话区列出历史会话（💤），选中后按钮变「唤醒并发送」，发送后该会话移入活跃列表且其 agent 开工。
- [ ] 对空闲对话发消息：目标立刻开新回合处理；对忙碌对话：排队不打断；对子代理 id 发送被拒绝并提示走 `send_message`。
- [ ] 跨工作区投递被拒；Windows 下大小写不一致的同目录不被误判。
- [ ] 建群、加/删成员、广播、合并读群记录全部可用；重启后群（含自动群）仍在。
- [ ] 新开对话的工具列表包含 13 个 `intercom_*` 工具，逐个可调用且输出 schema 校验通过。
- [ ] `verify-cordis-config.ts` 全绿；`tsc -b tsconfig.host.json` 与 `tsconfig.client.json` 无错；客户端 bundle 以 `window.__ModuleLoader__.load` 开头。
- [ ] `intercom_list_dormant_conversations` + `intercom_wake_send` 能唤醒一个真实历史会话并投递成功（延续工作场景端到端）。
