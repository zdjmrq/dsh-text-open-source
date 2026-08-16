# 【文字开源】dsh-user-plugins-manager — DSH「用户插件」统一管理页（挂载/启停/卸载，写入 cordis.patch.yml 热生效）

> **这是什么**：本文件是「文字开源」描述，不是代码。把本文件全文粘贴给你 DSH 里的 AI 会话，
> AI 即可为你复刻出功能相同的插件；你也可以阅读本文件理解其原理，并自行微调需求。
>
> **对应源码**：https://github.com/zdjmrq/dsh-user-plugins-manager （描述基于 commit `c09d93a142164f59346f19700c2a798a57203583`，即 tag v1.2.3）｜许可证：MIT
>
> **最近更新**：2026-08-16

## 1. 插件概述

本插件在 DeepSeek Harness（DSH）的 **设置 → 插件** 页里新增一个 **「用户插件」标签页**，把三种插件形态统一到一个页面管理：① `~/.dsh/plugins` 目录下的散件插件文件（`.mjs/.js/.cjs`）的挂载/停用/启用/卸载；② profile `node_modules` 里已安装的 npm dsh 插件包（`package.json` 声明 `dsh.bundle.patch`）的按包名挂载/卸载与来源展示；③ Loader 运行树里其他插件（部署自带、插件包、动态挂载等）的按条目 id 停用/启用，并标注「自装 / 官方」来源徽章与实时运行阶段。

核心价值：所有写操作都落在 `cordis.patch.yml` 补丁层（行级文本手术，只动目标条目、其余行含注释原样保留），由 DSH CLI 的 HMR 监听器热重载组合树——**无需重启 DSH**，页面内即时生效。本插件不修改 `node_modules`、不破坏插件包，卸载只移除补丁行，文件/包保留在磁盘。

它复用 DSH 内置能力：与内置「已配置 / 全部」标签页并列挂在同一个 `settings.plugins.tab` 槽位（零覆盖内置 UI），运行树数据与内置「全部」页同源（`ctx.loader.entries()`）。适合在 DSH 的 web profile（`~/.dsh/profiles/web`）中安装使用；与「文字开源」仓库生态的其它 dsh-* 插件无代码依赖，仅依赖 DSH 宿主自身服务。

## 2. 功能规格

页面由三组可折叠卡片组成（默认全部折叠，标题行整行即折叠开关，展开状态保存在组件 state、页面会话内保持）。以下为用户可感知行为（可验收清单）：

### 全局行为

- **刷新**：点击「刷新」并行拉取 3 个数据源（state / loader / packages），期间按钮禁用；任一失败独立显示错误行，不阻塞其它两组。
- **元信息区**（数据加载成功后展示在页面顶部）：
  - 插件目录路径 `~/.dsh/plugins`；目录不存在时显示「(不存在,请先创建该目录)」。
  - 两个补丁文件路径（profile 级 `~/.dsh/profiles/web/cordis.patch.yml` 与 home 级 `~/.dsh/cordis.patch.yml`），各自标注是否存在；未创建时显示「(未创建,首次挂载时自动创建)」。
  - 当前会话文件策略模式；非 `danger-full-access` 时以警告色显示「(写入补丁层需要 danger-full-access,请在权限设置中切换)」。
- **错误处理**：任何操作返回 `ok:false` 时把 `error` 消息显示在页面顶部（红色）；涉及策略不足时宿主会把提示拼进 error 一并显示。

### 第一组：插件目录（DSH_HOME/plugins）

- 扫描 `~/.dsh/plugins` 顶层目录，只列 `.mjs / .js / .cjs` 文件（大小写不敏感），按文件名排序；每行显示文件名与 `file:///` URL。
- 每行状态徽章：**已启用**（绿色）/ **已停用**（橙色）/ **未挂载**（灰）；挂载的行额外显示条目 id（`id: xxx · file:///...`）。
- 运行阶段徽章（若该 id 在运行树中）：等待中 / 加载中 / 运行中 / 加载失败 / 卸载中。
- **挂载**（未挂载行）：校验文件名合法（非 `.`/`..`、不含路径分隔符、扩展名合法、文件真实存在）→ 自动生成唯一 id（见第 5 节 sanitizeId 规则，与两补丁层全部条目 id 去重，冲突追加 `-2/-3`…）→ 在补丁层文件追加 `- insert:` 块（`- id: <id>` + `name: '<file:///url>'`，单引号包裹）→ HMR 热生效。
- **停用**（已启用行）：在条目上就地改写/插入 `disabled: true` 行（缩进与条目键对齐）→ loader 跳过该条目并释放 fiber。
- **启用**（已停用行）：删除条目上的 `disabled: true` 行。
- **卸载**（任意已挂载行）：删除整个条目行区间；若该 `- insert:` 块内已无剩余条目则连块头一并删除。插件文件本身保留。
- 若文件已挂载（按 name 匹配 URL），再次挂载会被拒绝并提示已有条目 id。

### 第二组：已安装的 npm 插件包（profile/node_modules）

- 扫描 `~/.dsh/profiles/web/node_modules` 顶层目录（跳过隐藏目录），只识别 `package.json` 中 `dsh.bundle.patch` 为字符串的包；每行显示包名、版本（`v1.2.3`，缺省显示 `?`）、依赖来源（profile `package.json` 里 `dependencies[包名]` 的 specifier，如 `git+...` / `file:...`）。
- 状态徽章三态：**全局挂载**（已列入 profile `dsh.profile.bundles`）/ **补丁层挂载**（补丁层有按包名 insert 的条目）/ **已安装未挂载**。
- **挂载**（未挂载）：校验包名（`PKG_OK` 正则）、经 `scanPackages` 确认真实存在且未挂载（bundles/patch 挂载的均拒绝）→ 按**包名**追加 `- insert:` 条目（与文件挂载同机制）→ HMR 热生效，无需改 `dsh.profile.bundles`、无需重启。
- **卸载**（补丁层挂载）：删除对应补丁条目行；npm 包保留在 node_modules。
- **全局挂载**的包：无操作按钮，提示「启停见下方"其他已挂载插件"组」。
- 重复挂载（已 bundle 或已 patch）会被拒绝并提示来源。

### 第三组：其他已挂载插件（Loader 运行树）

- 数据源 `ctx.loader.entries()`（与内置「全部」页同源），过滤掉 `options.group` 的条目与属于第一组的目录散件条目；每行显示：短模块名（归一化：去 `@scope/`、`cordis:`、`cordis-plugin-`、`dsh-host-`/`dsh-client-` 前缀）、条目 id、原始模块名（未声明 name 时显示「(未声明 name)」）、有效启停状态、fiber 阶段、来源。
- **来源徽章**：自装（品牌色）= 目录散件条目 id ∪ 用户补丁层 external 条目 ∪ 包清单里的包名；官方（灰）= 其余（随 dsh 部署自带、运行时基础设施）。
- **停用**（已启用条目）：若条目不在用户补丁层里（即部署/包插件）→ 弹 `window.confirm` 确认（提示恢复方法：手动删除补丁文件中对应的停用两行）→ 在补丁层文件末尾追加顶层「停用覆盖」裸行（`- id: x` + 两空格缩进 `disabled: true`）→ HMR 热生效。若条目本就在用户补丁层（自装），就地改 `disabled: true`，不弹确认。
- **启用**（已停用条目）：自装条目删 `disabled: true` 行；覆盖裸行条目整块删除（该块若含 `name/config/insert` 等其它键则拒绝并提示手动编辑，防误删）。
- **卸载**：仅自装条目（`source` 非空）且当前停用状态下显示「卸载」按钮，删除补丁条目行；部署/包插件提示「不能从用户补丁层卸载，请使用停用」。
- **悬空条目**：external 补丁行在运行树中找不到对应条目时仍保留展示，并标注「(未在运行树中)」。
- **阶段徽章**：等待中/加载中/运行中/加载失败/卸载中；失败以橙色显示。
- **优雅降级**：loader 接口不可用（宿主半未升级）时该组显示错误提示「(运行树接口由宿主半插件提供,更新插件后重启 dsh 即可;未升级时以下只显示补丁层自有条目)」，页面其它两组照常工作；packages 接口同理。

### 数据交互细节

- 每次操作成功后 700ms（等 HMR 重载）再拉一次 loader 与 packages 快照，保证运行树组与包组的展示跟上补丁层变化。
- 页面底部固定展示一段说明文字（三类插件的管理方式、HMR 热生效、卸载保留文件/包、停用系统插件风险提示）。

## 3. 技术路线

- **插件形态**：双半（Host + Client），一个 npm 包、零运行时依赖、零构建。
  - 宿主半：纯 ESM 的 Cordis 插件（`src/index.js`，导出 `name` / `inject` / `apply`），以 `export const name = 'user-plugins-manager'`、`inject = ['webServer', 'loader']` 声明；其余服务用 `ctx.get()` 可选读取。
  - 客户端半：**手写的 module-loader bundle**（`client/client.js`），即 `window.__ModuleLoader__.load({ id, factory })` 形状（`factory: (require) => ...` 返回 `module.exports`），与 DSH client-modules 的加载契约一致；不做 TS/JSX 转换，全部用 `React.createElement`（`const e = React.createElement`）+ 手写 CSS 字符串。
- **加载与集成方式**（npm 包 / bundle 补丁 / 客户端下发）：
  1. `package.json` 声明 `dsh.bundle.patch: "./cordis.patch.yml"` 与 `dsh.client.platform: "web"`；`exports` 暴露 `.`、`./client`、`./package.json`。
  2. `cordis.patch.yml` 是 bundle 补丁：`- insert: [{ id: user-plugins-manager, name: dsh-user-plugins-manager }]`，作为 profile 层应用，把宿主半插入组合树。
  3. 安装方式：`pnpm add "git+https://github.com/zdjmrq/dsh-user-plugins-manager.git"`（或 `file:<本地路径>`）装入 `~/.dsh/profiles/web`，并把包名追加进该 profile `package.json` 的 `dsh.profile.bundles` 数组。
  4. 客户端半由 DSH 内置 client-modules 插件扫描同一棵宿主树发现并下发到页面。
- **依赖的核心 Service**（宿主半，名称以仓库代码为准）：
  - `webServer`（硬依赖，inject）：`register({ kind: 'exact', path, handler(req, res) })` 注册 7 个 JSON 路由，返回 dispose。
  - `loader`（硬依赖，inject）：`loader.entries()` 取运行树条目（条目含 `id` / `options.name` / `options.group` / `disabled` / `fiber.state`）。
  - `fs`（可选）：`resolve(path)` / `stat(target)` / `readText(target)` / `writeText(target, content, expected?, signal?, sandboxPolicy?)` / `listDir(target)` / `fileUrl(target)`——所有磁盘读写都经 fs 服务，不直接碰 Node fs。
  - `settings`（可选）：`prepareDocument()` 返回文档路径，用于反推 DSH_HOME。
  - `sandboxPolicy`（可选）：`resolve({ session })` / `resolve()` 取沙箱策略（含 `mode`）。
  - `sessions`（可选）：`get(sessionId)` 取会话对象。
  - `launchEnvironment`（可选）：`get(name)` 取启动环境快照条目（`{ value }`），作为 DSH_HOME 定位兜底。
  - 客户端半：`slots`（`slots.inject('settings.plugins.tab', () => slots.register({ name, id: 'user-plugins', order: 20, label: '用户插件' }, renderFn))`）；`fetch` 调宿主路由；`useSessions` 钩子（由槽位 props 注入）取当前会话 id；CSS 使用 `--dsw-alias-*` 主题 token。
- **关键外部依赖**：无 npm 运行时依赖；需要 DSH 的 web 前端（提供设置页与 `settings.plugins.tab` 槽位）、CLI 对 `cordis.patch.yml` 的 HMR 监听、client-modules 模块发现机制。
- **与其它 dsh-* 仓库的配套关系**：属于 DSH 用户侧生态插件（主题 `dsh-plugin`，Oh-My-DSH 生态可检索）；不依赖任何其它插件，但依赖 DSH 内置 `ui-settings-plugins`（槽位宿主）与 `plugin-inventory`（同源的运行树数据）存在。

## 4. 结构设计

| 路径 | 职责 |
| --- | --- |
| `src/index.js` | 宿主半（558 行）：7 个 `/dsh-user-plugins/*` JSON 路由 + 补丁层行级文本手术（parse/mutate）+ 目录/包扫描 + 会话沙箱策略围栏 |
| `client/client.js` | 客户端半（425 行）：`window.__ModuleLoader__.load` 手写 bundle，注册「用户插件」标签页 UI（三组可折叠 + 刷新/操作 + 徽章） |
| `cordis.patch.yml` | bundle 补丁：把宿主半 `insert` 进组合树（id `user-plugins-manager`，name 指向包名） |
| `package.json` | `dsh.bundle.patch` + `dsh.client.platform: web` 声明、exports、files 白名单 |
| `README.md` / `LICENSE` | 使用文档 / MIT 许可（打包发布包含） |

**模块调用关系**：

```
浏览器页面 (client/client.js, ManagerView)
   │ fetch GET /dsh-user-plugins/{state,loader,packages,mount,unmount,enable,disable}
   ▼
src/index.js 路由层 (webServer.register, kind:'exact')
   │
   ├── scanState(sessionId)     → locateHome() → fs 扫 ~/.dsh/plugins + 两补丁层 parsePatch
   ├── loaderSnapshot()         → ctx.loader.entries() → 条目投影 (FiberState 数值→阶段文本)
   ├── scanPackages(sessionId)  → fs 扫 profile/node_modules 的 dsh 包 + bundles 清单 + 补丁层条目
   ├── mountPlugin/mountPackage → sanitizeId 去重 → 追加 '- insert:' 块 → writeLines
   └── mutatePatch(id,source,kind) → parsePatch/BARE 定位 → 行级增删 → writeLines
        │ 所有写路径统一走 fs.writeText(..., policyFor(sessionId)) 沙箱围栏
        ▼
   cordis.patch.yml (profile 级 / home 级) ── DSH CLI HMR 监听 ──► 组合树热重载
```

客户端内部数据流：`refresh()` 并行拉 state/loader/packages → `buildOtherItems(data, loaderData, packagesData)` 把目录行、external 补丁行、运行树条目合并成第三组（按 id 吸收 source、标记悬空、判定自装/官方）→ 操作后 `setTimeout 700ms` 重拉 loader/packages。

## 5. 关键实现细节

### 核心逻辑（步骤化）

1. **DSH_HOME 定位**（`locateHome`）：优先 `settings.prepareDocument()` 成功且非空 → 取其路径的 dirname（Windows/Unix 分隔符都兼容）；失败则读 `launchEnvironment` 的 `DSH_HOME`（原值）/ `USERPROFILE` / `HOME`（后者拼 `/.dsh`）。全部不可得返回 undefined，扫描接口报「无法确定 DSH_HOME」。补丁文件路径固定为 `[home + '/profiles/web/cordis.patch.yml', home + '/cordis.patch.yml']`（profile 级在前，home 级在后）。
2. **补丁层解析**（`parsePatch`，只读 insert 块、其余行原样保留）：
   - `TOP = /^-\s+([A-Za-z0-9_-]+):\s*$/` 识别块头（必须是 `insert`）；
   - `ITEM = /^(\s*)-\s+id:\s*(.*)$/` 识别条目，`idIndent` 记录缩进；
   - `KEY = /^(\s*)(name|disabled):\s*(.*)$/` 只认缩进恰为 `idIndent + 2` 的 `name`/`disabled` 键；
   - 块结束判定：遇到任何顶格 `- ` 行（下一个块头或顶层裸覆盖行），或缩进 ≤ idIndent 的 `- id:`；`endLine` 推进到最后一个非空、非注释行（保证注释/空行不算条目的一部分）。
3. **顶层裸行解析**（`parseBare`，供 enable 整块删除）：顶层 `- id: x` + 缩进键；记录键名集合，`name|disabled|config|insert` 都算真实条目键。
4. **条目级变更**（`mutatePatch`）：
   - id 白名单 `ID_OK = /^[A-Za-z0-9_-]{1,64}$/`，不匹配直接拒绝并提示手动编辑；
   - `source` 参数必须属于两个已知补丁路径，否则拒绝；
   - 条目在 insert 列表里：`disable` = 就地改写已存在的 `disabled: true` 行（缩进 `idIndent+2`）或在其后插入一行；`enable` = 删除该行；`unmount` = splice 条目行区间，若该块无剩余条目则连块头行删除；
   - 条目不在 insert 列表里：先用 `loaderSnapshot()` 校验 id 真实存在于运行树（否则报「运行树中不存在条目」）；`disable` = 文件尾补两行 `- id: x` / `  disabled: true`（先清尾部空行；文件不存在则先写带头注释的模板）；`enable` = parseBare 定位整块删除，块含其它键则拒绝；`unmount` = 拒绝（提示用停用）。
5. **挂载**（`mountPlugin` / `mountPackage`）：
   - 文件名防御：非空、非 `.`/`..`、**不含 `/` 或 `\`**（防路径穿越）、扩展名 `.mjs/.js/.cjs`、经 `fs.resolve` + `fs.stat` 确认为文件；
   - `sanitizeId(basename)`：去扩展名 → 小写 → 非 `[a-z0-9-]` 替换为 `-` → 去首尾 `-` → 空则 `user-plugin` → 首字符非字母则前缀 `plugin-`；
   - id 去重：遍历两个补丁层的全部条目 id，冲突则 `base-2`、`base-3`…；
   - 写模板：文件不存在时先写 `# dsh 用户插件补丁层(由"设置 → 插件 → 用户插件"页管理)。` + `# 顶层 YAML 数组:loader 补丁条目(insert/覆盖/disable 列表)。` 两行注释 + 空行；追加 `''`、`- insert:`、`    - id: <id>`、`      name: '<url 或包名>'`（单引号包裹）；
   - 包挂载前必须 `scanPackages` 确认包存在且 `mounted === 'none'`（bundle/patch 均拒绝），包名长度 ≤ 128 且匹配 `PKG_OK`。
6. **沙箱围栏**（`policyFor`）：`sandboxPolicy.resolve({ session })`（带 sessionId 且 `sessions.get` 命中时）或 `resolve()`；该策略作为 `fs.writeText` 的第 5 参传入。fs-sandbox 后端在省略策略时回落到部署默认（workspace-write）并拒绝工作区外的补丁文件，带上会话策略后按会话自身模式（如 danger-full-access）围栏——这就是「会话必须 danger-full-access 才能写补丁层」的实现。非危险模式时 `policyHint` 把提示拼进错误消息。
7. **HTTP 路由**：`ctx.effect(() => { ...; return () => disposes })` 注册 7 个路由并在卸载时全部释放；handler 用 `new URL(req.url, 'http://dsh.local').searchParams` 解析 query；响应 `Content-Type: application/json; charset=utf-8` + `Cache-Control: no-store`；异常统一 catch 成 `{ ok:false, error }`。
8. **运行树投影**：`FIBER_PHASE = { 0:'pending', 1:'loading', 2:'active', 3:'failed', 5:'unloading' }`（4 = disposed → null），与 DSH 的 FiberState 数值枚举一致；`options.group` 的条目被过滤；`disabled !== true` 视为 enabled。

### 重要边界处理与坑

- **不解析 YAML 语义，做行级文本手术**：注释、空行、其它块原样保留；endLine 只到最后一个非空非注释行，避免误删注释。
- **YAML 引号**：`stripQuotes` 剥掉单/双引号再比较/使用；name 比较用 `sameName`（先小写、再 `decodeURIComponent` 兜底，兼容 file URL 编码差异）。
- **停用本管理器自身**：写入后 HMR 立即生效、页面中断——confirm 文案专门给出恢复路径（删除 `~/.dsh/profiles/web/cordis.patch.yml` 中对应停用两行后刷新）。
- **BARE 块含其它键时 enable 拒绝删除**（`name/config/insert`），防止误删有配置的条目。
- **重复挂载防护**：文件按 name 匹配 URL、包按 name 匹配，已挂载一律拒绝并给出条目 id。
- **包清单健壮性**：node_modules 扫描对每个包独立 try/catch（manifest 损坏/缺失跳过）；profile package.json 解析失败按空处理；隐藏目录（`.` 开头）跳过。
- **Windows 兼容**：dirname 同时处理 `\` 与 `/`；补丁行分隔统一 `\n`（读入时按 `\r?\n` 切分，写出 `lines.join('\n')` 并保证末尾换行）。
- **目录不可读/不存在**：按空列表处理（`pluginDirExists:false` 提示用户创建）。
- **HMR 时序**：客户端操作后 700ms 再拉快照；若 HMR 更慢，运行树组可能短暂滞后，再次刷新即可。

### 配置项 / 常量

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `patchPaths(home)` | `[<home>/profiles/web/cordis.patch.yml, <home>/cordis.patch.yml]` | 双补丁层；挂载默认写入已存在的第一个，都不存在写 home 级 |
| `ID_OK` | `/^[A-Za-z0-9_-]{1,64}$/` | 自动写补丁的条目 id 白名单 |
| `PKG_OK` | `/^(@[a-z0-9][a-z0-9._-]*\/)?[a-z0-9][a-z0-9._-]*$/i` | 包名合法性（含 scoped） |
| `FIBER_PHASE` | 见上 | FiberState 数值 → 阶段文本 |
| `order: 20` | — | 标签页槽位排序（内置「全部」为 10，本页在右侧） |
| 700ms | — | 操作后重拉 loader/packages 的延迟 |
| 128 | — | 包名字符长度上限 |
| 环境变量 | `DSH_HOME` / `USERPROFILE` / `HOME` | 仅作为 DSH_HOME 定位兜底，无其它配置项 |

### 与 DSH 版本相关的耦合点（随版本漂移，复刻时以目标 DSH 为准）

- `webServer.register({ kind:'exact', path, handler })` 路由注册契约；
- `loader.entries()` 条目形状：`id` / `options.name` / `options.group` / `disabled` / `fiber.state`（FiberState 数值枚举）；
- `settings.prepareDocument()`、`sandboxPolicy.resolve({ session })`、`sessions.get()`、`launchEnvironment.get(name).value`；
- `fs.writeText(target, content, expected?, signal?, sandboxPolicy?)` 第 5 参沙箱策略；
- 客户端 `slots.inject('settings.plugins.tab', ...)` / `slots.register({ name, id, order, label }, render)` 契约与槽位 props（`useSessions` 注入）；
- `window.__ModuleLoader__.load({ id, factory })` 与 client-modules 的发现机制；
- DSH CLI 对 `cordis.patch.yml` 的 HMR 监听行为。

## 6. 集成与安装

从本描述到插件生效的完整步骤：

1. **粘贴本文件**给你的 DSH AI 会话，要求「复刻 dsh-user-plugins-manager 插件」。AI 应产出 4 个文件：`src/index.js`（宿主半）、`client/client.js`（客户端半）、`cordis.patch.yml`（bundle 补丁）、`package.json`（`dsh.bundle.patch` + `dsh.client.platform: web` + exports）。要求 AI 严格按第 5 节的正则、常量和流程实现，并按第 8 节验收清单自测。
2. **放哪**：任意目录，`package.json` 的 `name` 建议保留 `dsh-user-plugins-manager`（客户端半的 `__ModuleLoader__.load` id 与之一致）。
3. **装入 profile**：
   ```bash
   cd ~/.dsh/profiles/web
   pnpm add "file:<你的插件目录>"   # 或 git+https://github.com/zdjmrq/dsh-user-plugins-manager.git
   ```
   并把 `"dsh-user-plugins-manager"` 追加进该 profile `package.json` 的 `dsh.profile.bundles` 数组末尾。
4. **生效**：重启 DSH（宿主半路由生效）；客户端半从磁盘下发，页面 **Ctrl+F5** 刷新即可看到新标签页，无需重启。
5. **验证**：打开 设置 → 插件 → 用户插件，按第 8 节检查单逐项验收。

## 7. 已知边界与注意事项

- **会话级插件不显示**：agent 预设（隔离域）与动态 Cordis 插件不在根 Loader 树里（与内置「全部」页一致），本页不管理它们。
- **只显示当前 web profile**：headless / tui 组合树各自独立，本页展示当前 web 组合。
- **停用 id 白名单**：非 `[A-Za-z0-9_-]{1,64}` 的条目无法自动写覆盖行，需手动编辑补丁文件。
- **停用系统关键插件有风险**：如停掉 `webserver` 等核心行可能让页面不可用；恢复方法 = 删除补丁文件中对应的停用两行。页面有确认弹窗，但仍需谨慎。
- **沙箱限制**：会话文件策略非 `danger-full-access` 时写入补丁层会被 fs 服务拒绝，页面会提示切换；这是设计行为（所有写入经会话策略围栏），不是 bug。
- **路由无额外鉴权**：宿主半的 7 个 GET 路由不校验调用方身份，仅携带 `sessionId` 参数用于解析沙箱策略——依赖 DSH web 服务本身的访问控制（**待确认**：DSH 内置 web 是否对内部路由有鉴权层；复刻时建议沿用仓库现状，不额外加鉴权）。
- **卸载语义**：只移除补丁行，插件文件 / npm 包保留在磁盘；卸载 npm 包后若想彻底删除需手动 `pnpm remove`。
- **双补丁层**：profile 级与 home 级都会被读写，操作精确落在条目所在文件；两文件都可能被 DSH 自身或其它工具编辑，行级手术基于文本快照，若并发修改可能丢失更新（**待确认**：宿主未对写前冲突做版本校验，`fs.writeText` 的 expected 参数未使用）。
- **目录散件仅顶层**：`~/.dsh/plugins` 的子目录不被扫描，文件名不得含路径分隔符。
- **客户端降级**：宿主半未升级（缺 loader/packages 路由）时对应组显示提示并降级，不报错。
- **HMR 依赖**：热生效依赖 DSH CLI 对补丁文件的监听；若所在部署关闭了该监听，改动需重启生效。

## 8. 复刻验收标准（给复刻 AI 的检查单）

- [ ] **安装集成**：`pnpm add` 装入 web profile 并把包名加入 `dsh.profile.bundles` 后，重启 DSH，设置 → 插件 → 用户插件 标签页出现（与「已配置 / 全部」并列，id `user-plugins`，order 20），宿主路由 `GET /dsh-user-plugins/state` 返回 `{ok:true,...}`。
- [ ] **目录散件全生命周期**：放入 `my-plugin.mjs` 到 `~/.dsh/plugins` → 刷新 → 显示「未挂载」→ 挂载 → 补丁层出现 `- insert:` 条目（id 为 `my-plugin`，name 为 `file:///` URL）且页面变「已启用」→ 停用 → 补丁条目出现 `disabled: true` → 启用 → 该行消失 → 卸载 → 条目整块消失，文件仍在目录。
- [ ] **运行树启停**：对任一部署/包插件点停用 → confirm 弹出 → 确认后补丁层文件末尾出现 `- id: x` + `  disabled: true` 裸行，运行树组该条目变「已停用」；点启用 → 裸行整块删除、恢复「已启用」；来源徽章对目录散件/补丁条目/包名显示「自装」、对部署基础设施显示「官方」。
- [ ] **npm 包管理**：`pnpm add` 一个声明 `dsh.bundle.patch` 的包 → 包清单出现（含版本与依赖 specifier）→ 未挂载点挂载 → 补丁层出现按包名的 insert 条目 → 卸载后条目消失、包仍在 node_modules；已列入 bundles 的包显示「全局挂载」且无挂载按钮。
- [ ] **沙箱围栏**：在 `workspace-write` 会话下执行写操作 → 返回 `ok:false` 且错误信息含策略提示，补丁文件未被修改；切到 `danger-full-access` 后正常。
- [ ] **幂等与防错**：重复挂载同一文件/包被拒绝并提示；`id` 含非法字符（如空格、中文）的操作被拒绝并提示手动编辑；`mount?file=../x` 被拒绝；停用不存在的 id 报「运行树中不存在条目」。
- [ ] **结构与集成点**：宿主半导出 `name='user-plugins-manager'`、`inject=['webServer','loader']`，路由经 `ctx.effect` 注册并在插件停止时释放；客户端半经 `window.__ModuleLoader__.load` 注册、`slots.inject('settings.plugins.tab')` 挂标签页；补丁文件写经 `fs.writeText` 并传会话沙箱策略；注释与其它 YAML 块在行级手术中原样保留。
