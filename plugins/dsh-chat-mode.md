# 【文字开源】dsh-chat-mode — DSH「对话」ChatGPT 纯聊天模式插件

> **这是什么**：本文件是「文字开源」描述，不是代码。把本文件全文粘贴给你 DSH 里的 AI 会话，
> AI 即可为你复刻出功能相同的插件；你也可以阅读本文件理解其原理，并自行微调需求。
>
> **对应源码**：https://github.com/zdjmrq/dsh-chat-mode.git （描述基于 commit `69aad47a68aaeccfacde4cc441ed4d25defdc31c`）｜许可证：MIT
>
> **最近更新**：2026-08-26

## 1. 插件概述

这是一个 DeepSeek Harness（DSH）插件：为 DSH 增加一个 **「对话」纯聊天模式**（ChatGPT 式），与已有的 **DSH 编码代理模式** 并列。侧边栏「新会话」按钮直接按**当前模式**开启新会话；按钮右侧新增一个小三角（与工作区分组展开箭头同款），点击展开「会话模式」菜单（**DSH** / **对话**），菜单项在切换模式的同时**立即开启对应模式的新会话**（与当前处于什么状态无关）。「对话」模式会话由 `chat` agent preset 组成：只带 `ask_user_question` 与 `web_search` 两个工具，没有工作区、shell、文件系统、子代理、工作流、目标、技能与计划模式；所有对话会话固定落在专属聊天工作区 `$DSH_HOME/chat`（首次使用时自动创建）。核心价值是给 DSH 用户一个"随时切到纯聊天问两句"的入口，同时保证 DSH 模式的新会话**永远不会有落进聊天工作区的歧义**。本插件形态为 **用户级 agent preset + 宿主/客户端补丁**（按源码方式集成进 DSH 工作区，不发布 npm 包，基线 `0.1.1-rc.2` 之后的结构），与 [dsh-text-open-source](https://github.com/zdjmrq/dsh-text-open-source)（文字开源枢纽）及官方上游 [deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) 配套。

## 2. 功能规格

用户可感知的行为清单（可验收）：

- **功能 A：「新会话」按当前模式直开**（侧边栏 →「新会话」）
  - 当前模式为 DSH → 直接运行 DSH 模式新会话流程（排除聊天工作区，见功能 F）；
  - 当前模式为「对话」→ 直接运行 `startChatSession`，开启一个对话新会话；
  - 侧边栏展开时按钮带「新会话」文字标签，窄栏（rail）只显示图标；收起状态下模式小三角隐藏。
- **功能 B：模式小三角（新会话右侧）**
  - 24×24 圆形小按钮，图标为三角（`IconTriangleRightFill14`），点击展开/收起菜单（`aria-haspopup="menu"`、`aria-expanded`），展开时三角旋转 90°；
  - `aria-label` = 「会话模式: <当前模式标题>」（zh）/「Session mode: <title>」（en）；
  - 菜单内两项：**DSH**（构建、调试和发布）/ **对话**（创建、学习和探索），**当前模式带勾选标记**；菜单经 `portal` 渲染（侧边栏列会裁剪 overflow）。
- **功能 C：菜单项 = 切换模式 + 立即直开**
  - 点「DSH」→ 模式记忆写为 `dsh`，并立即开启一个 DSH 模式新会话（无参 `startSession()`，含聊天工作区排除与预设守卫）；
  - 点「对话」→ 模式记忆写为 `chat`，并立即开启一个对话新会话（`startChatSession('对话')`）；
  - **无论当前处于什么状态**（正坐在聊天会话里、DSH 会话里，或没有任何会话）都各自落到正确模式的新会话。
- **功能 D：对话会话 = 纯聊天**
  - 工具目录最多只有 `ask_user_question`（澄清提问）与 `web_search`（`fetch: false`、`searchTimeoutMs: 60000`——DeepSeek 搜索 provider 服务端检索，模型不选择请求目标）；
  - 没有 shell/文件系统/子代理/工作流/目标/待办/技能/计划模式；persona 为"友好而博学的助手，由 {{model}} 驱动"；
  - 输入栏行为保持正常聊天体验（发送/停止/编辑/图片），但**权限/访问芯片隐藏**——判定依据是会话摘要的 `agentPreset === 'chat'`，没有危险工具就无需守卫芯片（`sessionId === undefined` 时按不隐藏处理）；
  - 对齐官方 preset 风格：`persona` 不设 `complete: true`，web 界面的 orientation 提示段照常生效。
- **功能 E：专属聊天工作区**
  - 固定路径 `$DSH_HOME/chat`（默认 `~/.dsh/chat`），由 `host.describe` 的 `chatDir` 字段下发；
  - 首次使用：客户端以 `workspace.create({ path: chatDir, createDirectory: true, title: '对话' })` 递归创建目录并注册工作区（`title` 命名新工作区，省略时用路径 basename）；
  - 该工作区与普通 DSH 工作区并列分组展示（用户侧显示「对话」组）。
- **功能 F：DSH 新会话永不落入聊天工作区**
  - `startSession()` 的 target 解析：显式 workspaceId 优先 → 当前会话的工作区（**当前会话在聊天工作区时不继承**）→ 最近使用回退（**过滤掉聊天工作区**）→ 都无则清空选择进入 New Session 视图；
  - 连接到空白会话后，若它仍带着 `chat` 预设（例如被暂存在普通工作区里的空白聊天会话），则切回**部署默认预设**（roster `isDefault`，缺省取第一个未 broken 的可选预设）再打开——该预设守卫**只作用于无参流程**（工作区分组里的显式 ＋ 等作用域流程不受影响）；
  - 切换被拒时 console.warn 后照常打开（不阻塞）；连接失败也只是 console 诊断，当前视图保持可用。
- **功能 G：模式记忆**：选择的模式持久化到 `localStorage`（key `dsh.sidebar.prefs.v1`），刷新页面后保持。
- **功能 H：双语**：菜单标题、描述、aria 均有 zh/en 两套文案（`session.new.*` 命名空间，见 `locales.ts`）。

## 3. 技术路线

- **插件形态**：**用户级 agent preset（Cordis 组合）+ 宿主/客户端补丁**，不是独立 npm 包：
  - `agent-presets/chat/`（`preset.yml` + `agent.cordis.yml`）——用户 preset，放进 `$DSH_HOME/.agent-presets/chat/`，由宿主 `agentPresets` 服务扫描登记（与官方标准 preset 同一机制，**不修改官方预设目录**）；
  - `install.patch`——对 DSH 工作区的源码补丁（host apiproxy + client runtime + ui-sidebar + ui-conversation + connection fixture + test-support + 测试）。
- **加载与集成方式**：源码方式安装；补丁触及 6 个包：
  1. `packages/host/apiproxy`（宿主契约）：`host.describe` 新增 `chatDir`；`workspace.create` 支持 `createDirectory`/`title`；
  2. `packages/client/connection`（fixture + 测试）：describe 载荷补 `chatDir`；
  3. `packages/client/runtime`（管道层）：session 摘要带 `agentPreset`、`session.create` 透传 `agentPreset`、`WorkspaceRuntime` 重写新会话语义 + `startChatSession`；
  4. `packages/client/ui-sidebar`（入口 UI）：模式小三角 + 菜单 + 持久化 store；
  5. `packages/client/ui-conversation`：对话模式隐藏权限芯片；
  6. `packages/test-support/client-runtime`：测试替身补 `startChatSession`/`create` 新参数；对应单测随仓库附带。
- **依赖的核心 Service / Event / API**：
  - Client：`ctx.workspaces.startSession(workspaceId?)` / `ctx.workspaces.startChatSession(title?)`（`@deepseek-ai/dsh-client-runtime/client` 的 `IWorkspaces`）；`ctx.slots.inject/register`（slot `sidebar.root`，props 经 `PropsStore<...>` 注入持久化 store）；`ctx.layout.toggleSidebar()`；
  - Wire：`host.describe`（`chatDir`）、`workspace.create`（`createDirectory`/`title`）、`session.create`（`agentPreset`）、`agentPresets.list`（roster `isDefault`/`broken`）、`agentPresets.select`（`{ sessionId, agentPreset }`，宿主只允许**空白会话**切换——已开始的会话保持原组成）；
  - UI：`ui-primitives` 的 `Menu`（`portal`、`anchor`、`onSelect`）、`Tooltip`、图标；zustand 持久化 store（`createSidebarPrefsStore`，immer）。
- **关键外部依赖**：Node.js 22+、pnpm（monorepo）；客户端 React 18.2；无新增 npm 依赖。
- **与其他 dsh-\* 仓库的配套关系**：`dsh-text-open-source`（文字开源枢纽，本文件即其桥梁）；官方上游 `deepseek-harness` 为集成目标。

## 4. 结构设计

仓库文件布局：

| 路径 | 职责 |
| --- | --- |
| `agent-presets/chat/preset.yml` | 预设登记元数据：`name: 对话`、`description: 创建、学习和探索（纯对话模式）` |
| `agent-presets/chat/agent.cordis.yml` | 预设组合：persona + `dsh-tool-ask-user` + `dsh-tool-web(fetch:false)` + compaction 隔离组（`compaction-basic`/`command-compact`/`tool-result-pruner`，与 `standard` 相同 realm 规则） |
| `install.patch` | 对 DSH 工作区的累计补丁（见第 3 节 6 处） |
| `docs/DSH-Chat模式技术报告.md` | 设计说明：P0/P1 分级、交互修正历程（2026-08-26 晚的 DSH 绝不落聊天区语义） |
| `README.md` | 安装 / 工作原理 / 双向链接 |

补丁触及文件与职责：

| 路径 | 改动 |
| --- | --- |
| `packages/host/apiproxy/src/api-proxy.ts` | `ApiProxyDefaults` 增加可选 `chatDir`；`fallbackChatDir()`（`DSH_HOME` 非空→`resolve`，否则 `~/.dsh/chat`）；`host.describe` 返回值带 `chatDir`；`workspace.create` 支持 `createDirectory`（递归 mkdir）与 `title`（新建时 `setTitle`） |
| `packages/host/apiproxy/src/api/host{schema,ts}.ts` | `chatDir: string` 加入 describe 契约 |
| `packages/host/apiproxy/src/api/workspace{schema,ts}.ts` | create 载荷加 `createDirectory?`/`title?`（`title` 非空白 refine） |
| `packages/client/runtime/src/client/workspaces/service.ts` | `chatDirCache`/`chatDirPending` + `resolveChatDir()`；`startSession()` 聊天排除 + 预设守卫；`startChatSession()`；`handleConnected()` 预热 chatDir |
| `packages/client/runtime/src/client/contract/workspaces.ts` | `IWorkspaces.startChatSession`/`create` 新签名 |
| `packages/client/runtime/src/client/contract/sessions-port.ts` | `SessionsPortSummary.agentPreset?` |
| `packages/client/runtime/src/client/sessions/{manager,service}.ts` | `create` 透传 `agentPreset` |
| `packages/client/ui-sidebar/src/client/SidebarRoot.tsx` (+`.module.css`) | `.newSessionRow`（按钮+三角）、模式 `Menu`、`onSelect` 直开、rail 隐藏三角 |
| `packages/client/ui-sidebar/src/client/stores.ts` | `createSidebarPrefsStore`（`newSessionMode`，persist `dsh.sidebar.prefs.v1`） |
| `packages/client/ui-sidebar/src/client/{index,contract/slots,locales}.ts` | 注入 `startChatSession`、PropsStore、`session.new.*` 文案 |
| `packages/client/ui-conversation/src/client/skeleton/InputBar.tsx` | `chatMode` 判定 + 权限芯片隐藏 |
| `packages/client/connection/src/client/fixture.ts`、`packages/test-support/client-runtime/src/workspaces.ts` | 测试替身同步新契约 |
| 其余测试文件 | `chatDir` 载荷、`onAgentPresetList`、新会话单测、截图基线 |

关键调用关系：

```
SidebarRoot(新会话按钮/三角菜单) ──► workspaces.startSession()           ──► connectWorkspace ──► session.create(workspaceId) / 复用空白
                                        workspaces.startChatSession(title) ──► resolveChatDir() ──► workspace.create({path: chatDir, createDirectory, title})
                                                                                        └─► connectWorkspace ──► agentPresets.select('chat') ──► sessions.open
                                                                                        宿主: host.describe.chatDir ──► fallbackChatDir ($DSH_HOME/chat)
```

## 5. 关键实现细节

- **`startSession()` 目标解析（DSH 模式，无参）**：
  1. `target = workspaceId`（显式，作用域流程）；
  2. 否则 `currentWorkspace`（当前会话所在工作区）**且不是聊天工作区**时采用；
  3. 否则 `recentWorkspace(baselinesReady ? items.filter(不是聊天工作区) : [], byId)`（最近使用回退，取 `updatedAt` 最大的非聊天组，无会话按 `createdAt`）；
  4. 全为 `undefined` → `sessions.clear()`（进入 New Session 视图）；
  5. 命中后 `connectWorkspace(target).then(sessionId => workspaceId === undefined ? openWithAgentPreset(sessionId) : sessions.open(sessionId))`。
- **聊天工作区判定（`isChatWorkspace`）**：`chatDirCache` 已解析 → 按 `path === chatDirCache` 精确排除；**未解析**（冷启动、describe 失败）→ 启发式：任意成员会话 `agentPreset === 'chat'` 的工作区视为聊天区。注意：普通工作区里"暂存了一个空白 chat 会话"**不因此被排除**——它靠预设守卫纠正（见下）。`chatDirCache` 由 `handleConnected()` 预热。
- **预设守卫（`openWithAgentPreset`）**：仅当 `summary.blank === true && summary.agentPreset === 'chat'` 时，取 `defaultAgentPreset()`（roster `isDefault` ?? 第一个 `broken === undefined`），若默认 ≠ `'chat'` 则 `agentPresets.select({ sessionId, agentPreset: default })`；拒绝时 warn 后照常 open。**只挂在无参分支**（`workspaceId === undefined`），显式作用域（分组 ＋）不做守卫。
- **`startChatSession(title?)`**：`resolveChatDir()`（缓存 + in-flight 共享，失败后清 `chatDirPending` 以便下次重试）→ `create({ path: chatDir, createDirectory: true, ...(title === undefined ? {} : { title }) })` → `connectWorkspace(workspaceId)` → 若空白且预设 ≠ `'chat'` → `agentPresets.select({ sessionId, agentPreset: 'chat' })`（被拒则 warn 继续）→ `sessions.open(sessionId)`。整体 try/catch，失败 console.warn（非致命）。
- **`connectWorkspace` 复用规则**（既有机制，本插件复用）：只复用**工作区成员且 canonical cwd 相等且未归档**的空白会话，否则 `session.create({ workspaceId })`；同工作区并发连接用 `connecting` Map 合并。
- **宿主侧 `workspace.create`**：`createDirectory === true` 先 `mkdir(recursive)`，成功后 `ensureWorkspace`；仅 `created === true` 且 `title` 非空时 `workspace.setTitle(title.trim())`；schema 用 `.refine` 保证 title 非空白。缺目录且无 `createDirectory` 时维持原 `workspace-invalid-path` 业务错。
- **`chatDir` 缺省**：`fallbackChatDir()` —— `process.env.DSH_HOME` 非空 → `resolve(DSH_HOME)`，否则 `homedir()`；再 `join(..., '.dsh', 'chat')`。与 `dsh-home-paths` 的 `resolveDshHome` 优先级一致；`ApiProxyDefaults.chatDir` 为**可选**，省缺即用该缺省（宿主测试无需逐处补参数）。
- **UI 细节**：新会话按钮与三角分置 `.newSessionRow`（flex 行，按钮 `flex: 1`）——避免早期"把三角嵌进按钮内"导致的按钮缩水；三角 24×24 圆钮、`IconTriangleRightFill14`、展开旋转 90°（`.modeArrowOpen`）；菜单 `portal` 渲染（侧边栏列 overflow 裁剪）；`onSelect` 里 `actions.setNewSessionMode(id)` 后立即 `startChatSession`/`startSession`（**一个点击 = 切换 + 直开**）。
- **InputBar 芯片隐藏**：`chatMode = useSessions(s => sessionId === undefined ? undefined : s.byId[sessionId]?.agentPreset === 'chat') ?? false`；`accessSelect` 在 `command === undefined || chatMode` 时渲染 null。`sessionId` 未定（如 Hero/未连接）不误隐藏。
- **模式记忆**：`createSidebarPrefsStore()`（zustand + persist `dsh.sidebar.prefs.v1`，immer），状态 `{ newSessionMode: 'dsh' | 'chat' }`，action `setNewSessionMode`；经 `PropsStore<ReturnType<typeof createSidebarPrefsStore>>` 注入 `SidebarRoot`。
- **与 DSH 版本相关的耦合点**：依赖宿主已存在的 `session.create` 的 `agentPreset` 参数、`agentPresets.list/select`（含"仅空白会话可切换"限制）、`SessionsPortSummary.agentPreset` 字段（本补丁补上）；`host.describe` 的 `chatDir` 为新契约（老版本宿主 → `chatDir` 解析失败 → `startChatSession` 仅 warn，DSH 模式不受影响）。

## 6. 集成与安装

从拿到本描述到插件生效的完整步骤（粘贴给 AI → AI 产出 → 放置 → 生效 → 验证）：

1. 把本文件全文粘贴给你的 DSH 会话，附言"请按此描述复刻 dsh-chat-mode"。AI 应产出：
   - `agent-presets/chat/preset.yml` 与 `agent-presets/chat/agent.cordis.yml`（按第 3、4 节内容）；
   - `install.patch`（按第 3 节 6 处补丁 + 第 4 节表格的文件清单）。
2. 放置与集成：
   - `agent-presets/chat/` → `$DSH_HOME/.agent-presets/chat/`（默认 `~/.dsh/.agent-presets/chat`；Windows 为 `C:\Users\<你>\.dsh\.agent-presets\chat`）；
   - 在 harness 源码工作区根目录 `git apply install.patch`（版本差异对照手工合并）；
   - `pnpm install` → `pnpm run build`（宿主/客户端 tsc + bundle + web dist）。
3. 重启后台（`pnpm dsh web`），刷新页面。
4. 验证（见第 8 节检查单）。

## 7. 已知边界与注意事项

- **版本耦合**：`chatDir` 依赖宿主补丁；`agentPreset` 透传依赖宿主 `session.create` 已支持该参数（`dsh-agent-presets` 提供，`0.1.1-rc.2` 之后可用）。缺失时对话模式表现为 warn + 不开启（DSH 模式完全不受影响）。
- **预设切换仅限空白会话**：宿主在会话已开始后拒绝 `agentPresets.select`；因此对话/守卫切换都必须在打开前的空白窗口内完成，失败只能 warn。
- **守卫范围故意收窄**：只对"无参 DSH 新会话"生效；显式作用域（如聊天组自身的 ＋）保留原预设，这是刻意的（否则无法在普通工作区暂存聊天会话）。
- **冷启动启发式**：`chatDir` 未解析时用 `agentPreset === 'chat'` 启发式——在首个 baseline 到达前 New Session 可能短暂把聊天组当普通组处理（仅影响"最近回退"的极小窗口；`handleConnected()` 已预热）。
- **无产出物**：对话会话没有文件系统，生成的长内容只能手动复制（可发图片给模型）。
- **安全**：对话预设没有危险工具，但宿主审批栈/沙箱照常在线；`web_search` 用服务端检索（`fetch: false`）。
- 删除聊天工作区注册**不会**删除目录与其中的会话历史（沿用宿主工作区删除语义）。

## 8. 复刻验收标准（给复刻 AI 的检查单）

- [ ] 侧边栏「新会话」右侧出现 24px 圆形三角按钮，展开菜单含 DSH / 对话两项，当前模式有勾选（zh/en 文案 + `aria-label` 正确）
- [ ] 点「新会话」：当前为 DSH 模式 → 开启 DSH 新会话（不落在「对话」组）；当前为对话模式 → 开启对话新会话
- [ ] 点菜单「DSH」/「对话」：**无论当前坐在哪个会话（含对话会话自身）**，都立即开启**对应模式**的新会话
- [ ] 对话新会话：工具目录只有 `ask_user_question` 与 `web_search`；会话落在「对话」工作区（标题「对话」）；无 shell/子代理/技能/计划入口
- [ ] 对话输入栏无权限/访问芯片；发送、停止、图片等常规行为正常
- [ ] DSH 新会话：坐在对话会话里点「新会话」→ 落到最近使用的**非聊天**工作区（或 New Session 视图），绝不开出对话会话
- [ ] 普通工作区内暂存一个空白 chat 预设会话 → 无参新会话会把它切回默认预设再打开（已开始的会话不受影响）
- [ ] 刷新页面后模式选择保持（`localStorage dsh.sidebar.prefs.v1`）
- [ ] 聊天工作区目录首次使用时自动创建（`$DSH_HOME/chat`），聊天组显示正常
- [ ] 补丁对应测试（runtime `workspaces-service`、ui-sidebar、connection/wire-events）通过；`tsc -b tsconfig.{host,client}.json` 0 errors
