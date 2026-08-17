# 【文字开源】dsh-pluginmanager — DSH「插件管理」总览页（原生三层 / 用户扩展 / 运行中临时，启停 / 卸载 / 补登记 / 描述编辑）

> **这是什么**：本文件是「文字开源」描述，不是代码。把本文件全文粘贴给你 DSH 里的 AI 会话，
> AI 即可为你复刻出功能相同的插件；你也可以阅读本文件理解其原理，并自行微调需求。
>
> **对应源码**：https://github.com/zdjmrq/dsh-user-plugins-manager （描述基于 commit `8e36d9b`）｜许可证：MIT
>
> **上游来源**：当前实现来自 https://github.com/buhuikongpan/dsh-pluginmanager @ `8dbae51`，本仓库将其收录为 `dsh-pluginmanager` 的发布仓库。
>
> **最近更新**：2026-08-17

## 1. 插件概述

本插件在 DeepSeek Harness（DSH）的 **设置 → 插件** 页新增一个 **「插件管理」标签页**，把整个 web profile 的插件体系从一张 100+ 行的大平铺整理成**三层架构视图**：

1. **原生扩展**（只读承重墙）：自动分成 **系统层 / WebUI 层 / 工具层**，不提供卸载按钮；
2. **用户扩展**（自由区）：补丁行插件、扩展包（bundle）、依赖插件，可停用 / 启用 / 彻底卸载 / 补登记，并区分「未登记依赖」；
3. **运行中（临时）**：当前会话动态创建并运行的 Cordis 插件，只读展示，进程退出即消失。

核心价值：把“插件体系里谁是 Agent 大脑、谁是界面、谁是模型工具、哪些是你自己装的、哪些能拆”一眼讲清楚。操作上，启停与卸载都**优先热生效**：先持久化到 `cordis.patch.yml` / `package.json`，再对运行中的 Loader 条目做精准热切换，能不停服务就不停服务。

它通过宿主半的 `pluginManager` Typert Remote 提供服务（`snapshot` / `setEnabled` / `uninstall` / `saveDescription` / `register`），客户端半注册在 DSH 的设置页槽位。包名是 **`dsh-pluginmanager`**，GitHub 仓库名沿用 `dsh-user-plugins-manager`。

## 2. 功能规格

### 页面总览

- 顶部图例说明各层含义；搜索框按名称 / 显示名 / 描述 / 来源过滤，全局生效。
- 三个入口卡片：**原生扩展**、**用户扩展**、**运行中（临时）**，各自带数量徽章，点击展开/收起。
- 原生扩展内再分 **系统层 / WebUI 层 / 工具层** 三个 Section；用户扩展按来源分 **补丁行插件 / 扩展包 / 依赖插件 / 其它**。
- 操作成功/失败有顶部 notice；需要重启时显示重启横幅（若运行在桌面壳里可一键 `restartService`）。

### 原生扩展（只读）

- 来源判定基于 `@deepseek-ai/dsh-base` 与 `@deepseek-ai/dsh-web-app` 两个官方 bundle 的依赖 + 它们 `cordis.patch.yml` 里声明的插件名，加上 Loader 内置 `cordis:include` / `cordis:group`，以及 CLI 自身 `@deepseek-ai/dsh` 的依赖。
- 原生行只读，不显示卸载按钮；可编辑描述（写本地描述文件）。

### 用户扩展（可管理）

- 每行显示：中文显示名 + 包名、版本、来源徽章、启停/运行状态、工具插件标记、描述。
- **停用 / 启用**：
  - 对用户扩展条目，向 `~/.dsh/profiles/web/cordis.patch.yml` 写入或删除顶层停用覆盖块（`- id: <entryId>` + 两空格 `disabled: true`）。
  - 写文件前自动备份为 `cordis.patch.yml.bak-<时间戳>`。
  - 随后对运行中的 Loader 条目执行 `entry.update({disabled}, false, true)` 热切换（最多 3 次带 200ms 重试），目标插件即时重启。
- **彻底卸载**：
  - 若插件在 `dependencies` 里，先调用官方 `dsh plugin --profile web remove <包名>`（等价 pnpm remove + bundles 收尾），失败则**不继续**清理 patch，避免半状态。
  - 成功后清理 `cordis.patch.yml` 中该插件的 insert 行/停用覆盖/普通配置块，并删除描述记录。
  - 运行态释放：补丁层插件可热释放；bundle 层插件只能尽力热停用，并提示需重启才彻底消失。
- **补登记**：把手工丢进 `node_modules`、未写进 `package.json` 的插件，以 `file:./node_modules/<包名>` 写入依赖；随后尝试热挂载（成功则无需重启）。
- **未登记依赖**标签：`dependencies` 里没有的插件会显示，提示可补登记。

### 运行中（临时）

- 数据来自 `remote.dynamicCordisRunner.inventory()`；只读展示当前会话动态插件，标注为动态来源，不提供启停/卸载。

### 描述系统

- 内置 90+ 核心原生插件的中文名与一句话简介（`BUILTIN_META`）。
- 每个插件可点「编辑描述」，显示名/描述保存到 `~/.dsh/profiles/web/plugin-manager/descriptions.json`（原子写入）。
- 工具插件会尽力识别：原生工具层用内置 `TOOL_PROVIDED` 表；第三方插件扫描入口源码，检测 `defineTool` / `registerTool` / `ctx.tools.register` 等痕迹与 `output + execute` 结构签名。

### 预设感知

- 工具类插件（尤其模型工具）的真实启停由**当前 agent 预设**（`settings.yaml` 的 `agent-presets.default` → `agent.cordis.yml`）决定，而非宿主 Loader 树。
- 快照会读取预设行，解析 `disabled` 值（支持 `!!js` 表达式，沙箱只暴露 `process.platform/arch/version`），把预设控制的条目标记「预设」徽章，并以其判定结果覆盖 Host Loader 状态。

## 3. 技术路线

- **插件形态**：双半（Host + Client），一个 npm 包、零构建。
  - 宿主半：`lib/index.js`，默认导出 `PluginManagerGateway extends TypertRemoteService`，服务名 `pluginManager`；静态 `inject = ["loader"]`。通过 `Remote(name)` 装饰器暴露 5 个方法。
  - 客户端半：`lib/client.js`，手写 `window.__ModuleLoader__.load({ id: "dsh-pluginmanager", factory })` bundle；不编译 TS/JSX，使用 `React.createElement`（`react/jsx-runtime`）与手写 CSS 字符串。
- **加载与集成方式**：
  1. `package.json` 声明 `dsh.bundle.patch: "./cordis.patch.yml"`、`dsh.client.platform: "web"`，并声明客户端所需注入：`@deepseek-ai/dsh-api-remotes`、`@deepseek-ai/dsh-client-runtime`、`@deepseek-ai/dsh-client-ui-settings`、`@deepseek-ai/dsh-client-locale`。
  2. `cordis.patch.yml` 是 bundle 补丁：`- insert: [{ id: pluginmanager, name: dsh-pluginmanager }]`。
  3. 安装到 `~/.dsh/profiles/web`：`pnpm add "git+https://github.com/zdjmrq/dsh-user-plugins-manager.git"`，并把 `dsh-pluginmanager` 加入 profile `package.json` 的 `dsh.profile.bundles`。
  4. 客户端半由 DSH 内置 client-modules 机制发现并下发。
- **核心依赖**：
  - 宿主：`@deepseek-ai/dsh-typert-protocol`（Remote / TypertRemoteService）、`@deepseek-ai/cordis-plugin-include`（热挂载 Include 树）、`node:fs` / `node:path` / `node:child_process` / `node:module` / `node:os` / `node:url`。
  - 客户端：`slots`、`locale`、`remote`（`ctx.remote.$mount`）。
  - 宿主服务：`loader`（硬依赖 inject）；可选读取 `agentPresets`、`remote.dynamicCordisRunner`（客户端侧）。
- **关键外部依赖**：无 npm 运行时第三方依赖；依赖 DSH 的 web 设置页槽位 `settings.plugins.tab`、client-modules、Typert Remote 基础设施、`dsh plugin` CLI 与 pnpm。

## 4. 结构设计

| 路径 | 职责 |
| --- | --- |
| `lib/index.js` | 宿主半（约 1396 行）：`PluginManagerGateway` Typert Remote；profile/patch/descriptions 读写；原生判定与分层；预设合并；CLI 卸载；热挂载；Loader 热切换 |
| `lib/client.js` | 客户端半（约 749 行）：`__ModuleLoader__` bundle；locale zh/en；`settings.plugins.tab` 槽位注入；总览卡片 / Section / 行组件 / 描述编辑器 / 操作与重启横幅 |
| `cordis.patch.yml` | bundle 补丁：把宿主半以 `id: pluginmanager` 插入组合树 |
| `package.json` | 包名 `dsh-pluginmanager`、Typert/客户端声明、exports、repository |
| `README.md` / `LICENSE` | 使用文档 / MIT 许可 |

**模块调用关系**：

```
浏览器 (client.js, ManagerTab)
   │ ctx.remote.$mount(REMOTE) → remote.pluginManager
   ▼
lib/index.js PluginManagerGateway (Typert Remote)
   │
   ├── snapshot()        → profileManifest + patchText + descriptions + loader.entries()
   │                       + agentPresets 解析 → buildRow 分类/分层/来源/工具扫描
   ├── setEnabled()      → patch add/remove disabled 覆盖块 → setEntryDisabled 热切换
   ├── uninstall()       → dsh plugin remove → 清 patch 行/描述 → 按来源热释放
   ├── saveDescription() → descriptions.json 原子写
   └── register()        → package.json 写 file: 依赖 → hotMount Include 热挂载
        │
        ▼
   cordis.patch.yml / package.json / plugin-manager/descriptions.json
   （DSH CLI HMR / Loader update / Include 热树 → 当前进程即时生效）
```

## 5. 关键实现细节

### 核心逻辑（步骤化）

1. **快照构建**（`snapshot`）：
   - 读取 profile `package.json`、`cordis.patch.yml`、`descriptions.json`；`ensureDescriptionsFile()` 保证描述文件存在。
   - `ctx.loader.entries()` 收集实时条目，跳过 `options.group`；以 `entry.fiber !== undefined && state 非 3/5` 判断“启用”。
   - `nativeNameSet()` 与 `nativeLayerSets()` 基于两个官方 bundle 的 scoped 依赖和 patch 声明，加上 `FRAMEWORK_BUILTINS` 与 CLI 依赖。
   - 先遍历 live 条目按 `name` 去重生成 row，再补上未在运行树中出现的用户 patch 行（如加载前就停用）。
   - `buildRow` 决定 `native / layer / source / registered / enabled / active / phase / preset / displayName / description / tools / isToolPlugin`。
2. **预设解析**（`parsePresetRows` / `presetDisabledValue`）：
   - 读当前预设的 `agent.cordis.yml`，解析 `- id:` / `name:` / `disabled:` 行；同一 name 多行时按“全部 disabled 才 disabled”合并。
   - `!!js` 表达式用 `new Function("process", ...)` 在仅暴露 `platform/arch/version` 的极小沙箱中求值；失败视为启用。
3. **补丁文本手术**：
   - `parseBlocks` 按顶格 `- ` 切块；`blockOf` 提取 `id/name/disabled/kind`。
   - `patchRows` 遍历 insert 块的子行；`plainBlocks` 收集顶层裸块。
   - 停用 = `addDisabledOverride` 在文末补 `- id: x` + `  disabled: true`（已有相同 disabled 块则跳过）；启用 = `removeDisabledOverride` 删除该块。
   - 卸载 = `removePluginRows` 删除承载该插件名的整个 insert 块 + 对应 id 的停用覆盖；再 `removePlainBlocksForId` 清理非 disabled 的普通配置块。
   - 所有写前调用 `backupPatch`（同一调用内只备份一次，写 `.bak-<时间戳>`）。
4. **CLI 卸载**：
   - `runDshPlugin` 复用启动当前进程的 `dsh`（`process.argv[1]` 形如 bin/dsh 时直接 exec；否则 Windows 走 shell 启动 `dsh`），带 5 分钟超时、Windows `taskkill /T /F`。
   - `pluginArgsFor` 在 pnpm workspace 根自动注入 `-w`。
   - `withHoistRecovery` 处理三类 pnpm 坑：旧版 hoist pattern 差异（先 `install --no-frozen-lockfile` 重建）、release-age 锁（一次性 `--config.minimumReleaseAge=0` 重试）、瞬时网络失败（重试一次）。
   - 卸载顺序：先 CLI remove 成功 → 再清 patch/描述 → 最后热释放；CLI 失败直接返回，不产生半状态。
5. **热挂载**（`hotMount`）：
   - 启动时 `cleanHotDir()` 清掉上次遗留的 `hot-*.yml`。
   - 若包有 `cordis.patch.yml` 且是“纯 insert”（`parseSimplePatch` 只允许 id+name），把行写入 `plugin-manager/hot/hot-<n>.yml`，用 `@deepseek-ai/cordis-plugin-include` 的 Include 子类 `ctx.plugin(HotTree, { path })` 挂载。
   - 若包没有 patch 但声明 `dsh.client`，则把它加入 `shimNames`，客户端半照常下发、宿主半用 no-op shim 顶替，实现 client-only 插件热挂载。
   - 复杂 patch（含 config/表达式）拒绝热挂载，提示重启后生效。
6. **热切换**（`setEntryDisabled`）：
   - 遍历 `loader.entries()` 中 `options.name === name` 的条目；`entry.update({ disabled: flag ? true : null }, false, true)` 强制更新。
   - 最多 3 次尝试，每次检查 `entry.fiber !== undefined` 是否达到预期，未达到等 200ms 再试。
7. **客户端槽位**：
   - `ctx.remote.$mount(REMOTE)` 注册远程描述符（loose JSON codec）；调用前 `ctx.get("remote.pluginManager")` 懒解析。
   - `ctx.slots.inject("settings.plugins.tab", ...)` 注册 `id: "manager"`、`order: 30`、label 随 locale。
   - 动态列表通过 `ctx.get("remote.dynamicCordisRunner")?.inventory()` 获取，接口缺失时静默降级为空列表。

### 重要边界处理与坑

- **自我保护**：`SELF_NAMES = {"dsh-pluginmanager", "dsh-plugin-manager", "pluginmanager"}`，不能停用/卸载/补登记自身。
- **原生只读**：`native` 行不提供卸载；`setEnabled`/`uninstall`/`register` 都先 `findUserRow` 要求非 native。
- **BUNDLE 层卸载需重启**：DSH 的 HMR 只监听 `cordis.patch.yml`，不监听 `dsh.profile.bundles`；bundle 层插件从配置移除后仍需重启才从运行树彻底消失，页面会提示。
- **热挂载只支持纯 insert**：含配置/表达式的 bundle patch 无法热挂载，需重启。
- **未登记依赖**：`dependencies` 里没有、但已出现在 Loader/补丁里的插件显示「未登记」；补登记用 `file:./node_modules/...` 指向本地目录，离线安全。
- **直接文件写入**：宿主半使用 `node:fs` 直接读写 profile 文件，不经过 DSH 的 fs 服务/会话沙箱策略。这是与旧“用户插件”实现的重要差异：使用前应自行确认运行环境信任该插件。
- **只管理 web profile**：路径固定 `~/.dsh/profiles/web`；headless / tui 组合树不在范围内。
- **动态插件只读**：当前会话动态插件进程退出即消失，不提供持久化管理。
- **并发保护**：`this.mutating` 串行化写操作；另一个操作进行中直接返回“请稍后再试”。
- **Windows 兼容**：CLI 子进程用 taskkill 杀进程树；路径用 `node:path`；补丁文本按 `\r?\n` 切分、统一 `\n` 写出。

### 配置项 / 常量

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `PROFILE_NAME` | `"web"` | 固定管理 web profile |
| `NATIVE_BUNDLES` | `["@deepseek-ai/dsh-base", "@deepseek-ai/dsh-web-app"]` | 官方 bundle，原生判定依据 |
| `FRAMEWORK_BUILTINS` | `{"cordis:include", "cordis:group"}` | Loader 静态内置组件 |
| `SELF_NAMES` | `{"dsh-pluginmanager", "dsh-plugin-manager", "pluginmanager"}` | 不可自我管理 |
| `PLUGIN_TIMEOUT_MS` | `5 * 60 * 1000` | `dsh plugin` 子进程超时 |
| `FIBER_PHASE` | `{0:pending,1:loading,2:active,3:failed,5:unloading}` | FiberState 数值 → 阶段 |
| 热挂载目录 | `<profile>/plugin-manager/hot` | 仅当前进程的 Include 输入，启动清空 |
| 描述文件 | `<profile>/plugin-manager/descriptions.json` | 用户编辑的显示名/描述 |
| `order: 30` | — | 标签页槽位排序（位于内置页右侧） |

### 与 DSH 版本相关的耦合点

- `@deepseek-ai/dsh-typert-protocol` 的 `Remote` / `TypertRemoteService` 契约；
- `loader.entries()` 条目形状：`id` / `options.name` / `options.group` / `fiber`（`FiberState` 数值）；
- `entry.update({disabled}, false, true)` 强制热更新行为；
- `agentPresets.resolve() / read(id)` 与预设 `agent.cordis.yml` 结构；
- 客户端 `slots.inject("settings.plugins.tab", ...)` / `slots.register` / `locale` / `ctx.remote.$mount`；
- `window.__ModuleLoader__.load({ id, factory })` 与 client-modules 发现；
- `dsh plugin --profile web add/remove` CLI 与 pnpm workspace 行为；
- `@deepseek-ai/cordis-plugin-include` 的 `Include` 可继承/可 `ctx.plugin` 挂载。

## 6. 集成与安装

从本描述到插件生效的完整步骤：

1. **粘贴本文件**给你的 DSH AI 会话，要求「复刻 dsh-pluginmanager 插件」。AI 应产出 4 个文件：`lib/index.js`（宿主半）、`lib/client.js`（客户端半）、`cordis.patch.yml`（bundle 补丁，`id: pluginmanager` / `name: dsh-pluginmanager`）、`package.json`（`dsh.bundle.patch` + `dsh.client.platform: web` + client inject + exports）。要求严格按第 5 节实现，并按第 8 节验收清单自测。
2. **放哪**：任意目录，`package.json` 的 `name` 必须为 `dsh-pluginmanager`（客户端 `__ModuleLoader__.load` id 与之一致）。
3. **装入 profile**：
   ```bash
   cd ~/.dsh/profiles/web
   pnpm add "git+https://github.com/zdjmrq/dsh-user-plugins-manager.git"
   # 或 pnpm add "file:<你的插件目录>"
   ```
   并把 `"dsh-pluginmanager"` 追加进该 profile `package.json` 的 `dsh.profile.bundles` 数组末尾。
4. **生效**：重启 DSH（宿主半 Typert Remote 注册）；客户端半从磁盘下发，页面 **Ctrl+F5** 刷新即可看到「插件管理」标签页。
5. **验证**：打开 设置 → 插件 → 插件管理，按第 8 节检查单逐项验收。

## 7. 已知边界与注意事项

- **不是目录散件管理器**：当前版本管理“已进入 Loader / 依赖 / 补丁体系”的插件，不扫描 `~/.dsh/plugins` 目录，也不提供 `.mjs` 文件挂载。
- **直接文件写**：宿主半绕过 DSH fs 服务的会话沙箱围栏，直接读写 `~/.dsh/profiles/web` 下的文件；仅应在可信的个人 DSH 环境使用。
- **原生插件不可卸载**：这是设计行为；卸载按钮只出现在用户扩展。
- **bundle 层卸载需重启**：从 `dsh.profile.bundles` 移除的插件，DSH HMR 不会监听该文件，需重启才彻底消失。
- **热挂载范围有限**：只有纯 insert 的 bundle patch 或 client-only 插件可热挂载；含配置/表达式的插件补登记后仍需重启。
- **描述文件覆盖**：`descriptions.json` 的 `plugins[name]` 会覆盖内置 `BUILTIN_META`；删除文件可恢复默认。
- **预设控制覆盖 Host 状态**：工具插件如果由当前 agent 预设控制，页面显示的是预设判定结果，不是直接改 Host 补丁。
- **并发与备份**：写操作串行；`cordis.patch.yml` 每次首次变更前备份，但并发外部编辑仍可能造成文本覆盖（无版本冲突检测）。
- **依赖 CLI/pnpm**：卸载依赖 `dsh` CLI 与 pnpm；CLI 不存在或网络失败时卸载会中止并提示。

## 8. 复刻验收标准（给复刻 AI 的检查单）

- [ ] **安装集成**：`pnpm add "git+https://github.com/zdjmrq/dsh-user-plugins-manager.git"` 装入 web profile 并把 `dsh-pluginmanager` 加入 `dsh.profile.bundles` 后，重启 DSH，设置 → 插件 → 插件管理 出现（id `manager`，order 30）；宿主 `pluginManager` Remote 可调用 `snapshot()` 返回 `{ ok:true, value:{ profile:"web", rows:[...] } }`。
- [ ] **原生分层**：快照中 `@deepseek-ai/dsh-llm` 等核心为 `native` 且 `layer:"system"`，`@deepseek-ai/dsh-client-ui-*` 为 `webui`，`@deepseek-ai/dsh-tool-*` 为 `tool`；原生行不显示卸载按钮。
- [ ] **用户扩展启停**：对一个用户补丁行/bundle/依赖插件点停用 → `cordis.patch.yml` 出现 `- id: <entryId>` + `  disabled: true` 顶层块，运行中条目即时变「已停用」；点启用 → 该块删除、恢复「已启用」。
- [ ] **热切换与备份**：`setEnabled` 返回 `hot:true`（或明确 `needsRestart`）；写 patch 前生成 `cordis.patch.yml.bak-*`。
- [ ] **卸载**：用户扩展点卸载 → 二次确认 → 若在依赖里先执行 `dsh plugin --profile web remove` 成功 → patch 中该插件 insert/覆盖/配置块被清除、描述记录删除；bundle 层返回 `needsRestart:true`，补丁层返回 `hot:true`。
- [ ] **补登记与热挂载**：手工放入 `node_modules` 且未登记的插件显示「未登记依赖」；点补登记 → `package.json` 写入 `file:./node_modules/<包名>` → 若 patch 为纯 insert 则 `hot:true` 无需重启；复杂 patch 则提示重启。
- [ ] **描述编辑**：点编辑描述并保存 → `descriptions.json` 原子更新，刷新后显示名/描述仍在；内置原生插件有默认中文名/简介。
- [ ] **预设感知**：当前预设里被 `disabled:true`（含 `!!js` 表达式）的模型工具显示「预设」徽章且为已停用，即使 Host Loader 条目看似启用。
- [ ] **自我保护与并发**：对 `dsh-pluginmanager` 自身调用 setEnabled/uninstall/register 返回错误；连续快速操作第二个返回“另一个插件操作正在进行”。
- [ ] **临时动态组**：当前会话运行动态 Cordis 插件时，该组出现对应只读行；接口缺失时页面不崩溃、该组为空。
- [ ] **结构与集成点**：宿主默认导出 `PluginManagerGateway`（`TypertRemoteService` 子类、`inject:["loader"]`、5 个 Remote 方法）；客户端 `__ModuleLoader__.load({id:"dsh-pluginmanager"})`、`slots.inject("settings.plugins.tab")` 注册 `id:"manager"`；`cordis.patch.yml` 为 `- insert: [{ id: pluginmanager, name: dsh-pluginmanager }]`。
