# 【文字开源】dsh-careful-full-access — 只在 careful-full-access 模式生效的命令守卫：静态四档分级 + WhatIf 预演 + model-check 三问复核 + 红色人工确认 + 轮转审计

> **这是什么**：本文件是「文字开源」描述，不是代码。把本文件全文粘贴给你 DSH 里的 AI 会话，
> AI 即可为你复刻出功能相同的插件；你也可以阅读本文件理解其原理，并自行微调需求。
>
> **对应源码**：https://github.com/zdjmrq/dsh-careful-full-access （描述基于 commit `07a00f1ad2578d7137db62c106fb6adcd3bd9633`）｜许可证：MIT
>
> **最近更新**：2026-08-16

---

## 1. 插件概述

这是 DeepSeek Harness（DSH）的一个 **Host 半命令守卫插件**，只在沙箱模式 `careful-full-access` 下生效：对每一条 `pwsh`/`bash` 工具调用，在派发前做「静态四档分级 →（pwsh 删除）WhatIf 干跑解析真实删除范围 → model-check 三问复核 → 灾难级红色人工确认 → 双重审计」的完整管线，目标是把「解析错误的删除命令误删整个盘 / 整个工作区」这一事故挡在执行之前。

`careful-full-access` 是 DSH 核心的沙箱枚举值，**第三方插件无法自行添加**，因此本仓库同时附带 `patches/careful-full-access.patch`（对 DSH 源码树的核心补丁，与插件代码版本配套）。插件代码与补丁的分工：**补丁负责"注册模式与接线"**（第四档 `SandboxMode`、权限预设、UI 档位与图标、审批红色标注链路、工作区根 ACL 防删、`cordis.patch.yml` 挂载行），**src/ 负责"模式内的一切行为"**（分级、预演、复核、确认、审计）。两者缺一不可，共同构成完整功能。适用场景：需要"全权限体验"（文件访问不受限）但又不愿放弃删除防护的 AI 编码工作；配套仓库见 [dsh-plugin-suite](https://github.com/zdjmrq/dsh-plugin-suite)（把本插件与 dsh-restart-plugin 合并为累计 `install.patch`）与官方上游 [deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)。

## 2. 功能规格

> 行为按「触发条件 → 行为 → 结果」穷举。所有行为仅在 `careful-full-access` 模式下存在；其余模式一律透传。

### 2.1 模式门控与作用域

- **非 careful 模式**：会话模式为 `read-only` / `workspace-write` / `danger-full-access`（或策略未挂载）时，任何 `pwsh`/`bash` 调用**原样放行**，守卫零工作、零审计记录。
- **非 shell 工具**：`web_search` 等非 `pwsh`/`bash` 工具调用不经过守卫。
- **无命令参数**：`arguments` 不是对象、缺 `command` 字段、或 `command` 为空串/纯空白 → 直接放行（`next()`），无审计。
- **无 agent 执行**（agentless）：`exec.agent` 缺失时在策略门之前返回，放行、无审计。
- **双方言**：`pwsh` 与 `bash` 均被守卫；bash 无 AST/WhatIf 环节（见 2.3 与第 7 节）。
- **判定结果返回给工具管线**：`allow` → 继续执行；`deny` → 物化为错误（携带守卫原因文本）；`ask` → 进入审批通道，`severity: 'danger'` 时审批面板红色突出，审批策略 `never` 时自动拒绝。

### 2.2 静态四档分级（每条调用、派发前）

| 档位 | 判定规则 | 处置 |
| --- | --- | --- |
| **normal** | 无任何破坏信号的非破坏命令；顶层 `git` 的非破坏子命令（含 `git rm --cached`/`--staged`/`-n`/`--dry-run`——只操作索引不删工作区文件） | **直接放行**，不拉起任何辅助进程 |
| **elevated** | 所有删除/回收动词族：单个显式删除（`Remove-Item`/`rm`/`del`/`erase`/`rd`/`rmdir`/`ri` 及 cmd 别名）、批量删除（静态可见目标 > 1）、递归删除（动态目标 / 无静态目标 / 递归强制越出工作区或含通配 / 裸盘符形式 `C:`）、`git clean`、`git reset --hard`、`git rm`（无 `--cached`）、清空回收站 `Clear-RecycleBin`、`find -delete`（bash）、`.NET` 删除调用命中受保护根或受保护根内、`robocopy /MIR` 普通目录 | **model-check 三问复核**；模型判"本意且安全"则放行 |
| **disaster** | 盘符根（`C:\`）、根通配（`C:\*`、`C:\*.*`、`C:*`）、UNC 根（`\\server\share\`）、扩展根（`\\?\C:\`）、POSIX 根 `/`；用户主目录（`USERPROFILE`/`HOME`）、系统目录（`SystemRoot`/`ProgramFiles`/`ProgramFiles(x86)`/`ProgramW6432`）、工作区根、`extraProtectedPaths` 配置的根；`Format-*`/`Clear-Disk`/`Initialize-Disk`/`Remove-Partition`（及 bash `mkfs*`/`mkswap`/`fdisk`/`wipefs`）；`diskpart clean`；向受保护根 `robocopy /MIR`；受保护根的递归 `.NET` 删除（`Delete(path, $true)`）；受保护根的递归删除 | **model-check 复核 + 永不自动放行**：即使模型确认"本意且安全"仍走人工确认（红色） |
| **unparseable** | 动态执行动词 `iex`/`Invoke-Expression`（词法或 AST 不可静态分析）；有破坏信号但 AST 解析失败/不可读（spawn 失败、超时、解析器报错） | **按 disaster 对待**：人工确认（红色）；`never` 策略下拒绝。`iex` 绝不漏过闸门 |

### 2.3 WhatIf 真实范围预演（仅 pwsh 的 delete 族）

- 触发：`dialect === 'pwsh'` 且词法事实 `families` 含 `delete`，且档位非 normal。
- **干跑**：以 `$WhatIfPreference = $true` 前缀执行原命令（`pwsh -NoLogo -NoProfile -NonInteractive -EncodedCommand`），通配、变量、`$env:` 由 PowerShell 自身展开——守卫的解析不可能是误读的一环。
- **解析目标**：从输出解析 `What if: ... on target "..."` 行（英文与 zh-CN 本地化 `假设:` 两种格式）。
- **结果分支**：
  - 干跑证明删除零目标（`zero-targets`）→ **直接放行**，且不再调用 model-check。
  - 解析到具体范围（`previewed`）→ 生成有界摘要（对象总数 / 文件数 / 目录数 / 前 N 个目标样例 / 截断标记），附给 model-check 与人工确认。
  - 干跑解析出**受保护根**（`protected-hit`）→ 档位**升级为 disaster**，走红色人工确认。
  - 干跑不可用（spawn 失败、超时、非零退出且无目标、枚举失败）→ `unpreviewable`，失败细节附在人工确认理由中。
- **递归目录目标补枚举**：WhatIf 对目录只打印顶层，故对解析出的目录目标再跑一次**只读子树枚举**（`Get-ChildItem -Recurse -Force -ErrorAction SilentlyContinue`），把子文件/子目录数并入摘要，样例上限默认 10。
- **副作用权衡（文档化坑）**：WhatIf 只抑制 `ShouldProcess` 的输出与执行，删除语句之前的非删除副作用**会真实执行**——这是"让 shell 自己说真话"的代价。
- bash **没有** WhatIf 等价物：复核不带解析出的范围摘要（文档化降级）。

### 2.4 model-check 三问复核

- 触发：任何非 normal 档位（含 unparseable），但 `zero-targets` 预演结果除外。
- **有界旁路调用**：一次对会话当前路由模型（`agent.session.requestHeader()` 或 `agent.options` 的 provider/model）的调用；温度 0、输出上限默认 300 token、整体超时默认 20 s；**不进入会话历史**（`createUserMessage` 的 source 为 `{ kind: 'plugin', plugin: 'command-guard' }`）。
- **输入内容**：命令全文（代码块内）、静态档位（elevated/disaster/unparseable 的人类可读标签）、标记原因、可选 WhatIf 范围摘要。
- **要求输出**：严格 JSON `{"intent":"yes"|"no","assessment":"safe"|"dangerous","explanation":"一句话"}`；解析容错：容忍代码围栏与前后散文（抽取 JSON 对象），字段越界/缺失一律 fail-closed。
- **结果映射**：
  - `intent:"no"`（模型否认）→ **直接拒绝**（deny），拒绝理由携带模型自己的解释；不再人工确认。
  - `intent:"yes" + assessment:"safe"` → **elevated 放行**；**disaster/unparseable 仍走人工确认**（红色）。
  - `assessment:"dangerous"`（模型自称危险）→ **一律人工确认**；disaster/unparseable 档带红色，elevated 档为普通 ask。
  - 不可用（无路由/无 completer/超时/外部中止/流错误/答案解析失败）→ **fail-closed 按 disaster 处理**：人工确认（红色），elevated 的确认文案档位显示为 unparseable。

### 2.5 红色人工兜底（审批通道）

- `ask` 决策携带 `severity: 'danger'` 时：命令全文、档位标题（`DISASTER tier` / `unparseable (treated as disaster)`）、模型复核结论、预演摘要一并进入审批面板，以红色条带/边框/圆点突出（`data-approval-severity="danger"`）。
- 审批策略 `never` 时自动拒绝——该会话中被标记的命令不可执行。
- 确认/拒绝结果通过工具管线正常落回（`allowed-once` 才执行）。

### 2.6 双重审计

- **完整流水 → 轮转文件**：每个判定（allow/deny/ask）写一行 JSON 到 `$DSH_HOME/logs/command-guard.log`（默认路径，可配），默认 5 MB × 3 份轮转（`.1`…`.N`）；重复命令在 TTL 内只追加一行 `{"event":"repeat","fingerprint","count"}` 紧凑标记。
- **会话日志 → 有界窗口**：每次判定追加 `command-guard/decision` 会话事件（`toolName`/`decision`/`tier?`/`mode`/`reason?`（仅 deny/ask）/`modelCheck?`/`callId?`），默认每会话最多 20 条；相同命令指纹（空白折叠归一化）在 TTL（默认 10 分钟）内合并为一条、计数递增，不重复追加、不占上限。
- 审计失败（文件写失败 / 会话 append 抛错）**绝不翻转判定**，只写 logger.warn。

### 2.7 系统提示注入

- `enablePrompt` 默认 true：通过 `systemPrompt.context` 注册 `command-guard:deletion-discipline`（order 112）段落，内容为删除纪律（优先 `-WhatIf` 干跑、不递归盘根/用户目录/系统目录、`$env:` 未定义按错误处理、careful 模式下被标记删除先复核后执行）。

### 2.8 补丁侧行为（由 patches/ 提供，非 src/ 逻辑）

- 注册第四档 `SandboxMode = 'careful-full-access'` 与 `isUnconfinedMode`（danger 与 careful 都算"不受限"），各 sandbox executor（bash/pwsh/terminal/fs-sandbox）改为按 `isUnconfinedMode` 判断，careful 模式文件访问等同全权限。
- 权限预设 `careful-full-access`（approval: ask，描述：全权限，删除走守卫预演与复核）。
- UI：PermissionSelect 新增 `careful-full-access` 档位与**眼睛盾牌图标**；档位升级链 `WIDER_MODES`/`ESCALATION_TARGETS` 插入 careful（read-only → workspace-write → careful-full-access → danger-full-access）。
- 审批红色链路：`PreToolDecision` 增加可选 `severity: 'danger'` → approval service `ApprovalRequest.severity` → api-proxy mux 帧 `approval/requested` 携带 severity → Client `PendingApproval.severity` → ApprovalPanel 红色样式。
- 工作区根 ACL 防删：`grantWrite` 由单个 OI|CI 全 Modify ACE 改为**两条 ACE**——子孙继承的完整 `GRANT_MASK`（0x00110156，子项可删/改名/跑 git）+ 根对象自身无 `DELETE`/`FILE_DELETE_CHILD` 的 `ROOT_GRANT_MASK`（0x00100116，不继承）；旧单 ACE 形态在 `grantWrite` 时**原地迁移**为双 ACE（一次合并完成撤销+重建）。
- 挂载行：`packages/bundle/base/cordis.patch.yml` 增加 `- id: command-guard` / `name: 'dsh-careful-full-access'` 行，Host 平面每会话的 shell 调用都会经过它；`packages/bundle/base/package.json` 增加 `dsh-careful-full-access: workspace:^` 依赖；`tsconfig.host.json` 增加 `packages/guard/careful-full-access` 引用。

## 3. 技术路线

- **插件形态**：**Host 半**（纯 Node.js Host 插件，无 Client 代码），Cordis 插件；多模块单包（`src/` 下 15 个模块 + `./invariant` 伴生导出）；另有 `patches/` 宿主核心补丁与之配套。
- **加载与集成方式**（两种）：
  1. **源码树安装（推荐，唯一可用）**：`git apply patches/careful-full-access.patch` 把模式注册进宿主 → 将 `src/`、`tests/` 放入树内 `packages/guard/careful-full-access/`，用 `harness/` 里的骨架替换该包的 `tsconfig.json` 与 `package.json`（树式 tsconfig 引用 + `workspace:^` 依赖，运行时代码与宿主完全同源）→ `pnpm install && pnpm run build` → 重启 → 会话切到 `careful-full-access`。挂载行已由补丁写入 `packages/bundle/base/cordis.patch.yml`。
  2. **npm 包（当前不可用）**：`dsh-careful-full-access` 尚未发布（`pnpm add` 会失败），且发布前需先构建 `lib/`（`main`/`types` 指向 `lib/`）——待确认。
- **依赖的核心 Service / Event / Tool**（名称以仓库代码为准）：
  - `tools` 服务（`inject: ['tools']`，硬依赖）→ 监听 **`tools/pre-execute`** 事件（waterfall，返回 `PreToolDecision`：`allow` / `deny{reason}` / `ask{reason?, severity?: 'danger'}`）。
  - `sandboxPolicy` 服务（可选，`ctx.get`）→ `resolve({ session })` 取 `mode` 与 `workspaceRoot`；模式按字符串比较（双兼容公开发布类型）。
  - `llm` 服务（可选，`ctx.get`）→ `stream({ provider, model, messages, system, temperature, maxTokens, signal })` 的 `AsyncIterable<StreamChunk>`（`text-delta` / `finish`）。
  - `systemPrompt` 服务（条件注入）→ `scope.systemPrompt.context({ name, order: 112, text })`。
  - `session.append('command-guard/decision', …)` —— 通过 `declare module '@deepseek-ai/dsh-session/types'` 扩展 `SessionEventMap`。
  - `approval` 服务：插件**不直接调用**，由工具管线的 `ask` 决策触发。
  - `invariants` 服务（伴生插件 `command-guard-invariant`）→ `ctx.invariants.register('dsh-careful-full-access', noopInstaller)`（空实现：本插件无可运行时校验的不变量）。
- **关键外部依赖**：Node.js 运行时；`pwsh` 可执行（Windows PowerShell 7+，`Parser::ParseInput` AST 解析与 WhatIf 干跑均经 `-EncodedCommand` 拉起辅助进程）；bash 方言无需辅助进程；npm 依赖全部为 DSH 公开包（`@deepseek-ai/cordis`、`dsh-agent`、`dsh-invariants`、`dsh-llm`、`dsh-sandbox`、`dsh-sandbox-policy`、`dsh-session`、`dsh-system-prompt`、`dsh-tools`、`schemastery`，及运行依赖 `@deepseek-ai/dsh-home-paths`）。补丁侧还牵动 Windows ACL（`sandbox-windows-acl`，koffi 绑定）与 Client 审批面板。
- **与其他 dsh-\* 仓库的配套关系**：[dsh-plugin-suite](https://github.com/zdjmrq/dsh-plugin-suite)（局部 fork 套件：本插件与 [dsh-restart-plugin](https://github.com/zdjmrq/dsh-restart-plugin) 合并为一张累计 `install.patch` 一次接线）；官方上游 [deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（补丁目标树）。

## 4. 结构设计

| 路径 | 职责 |
| --- | --- |
| `src/index.ts` | 插件入口：导出 `name='command-guard'`、`inject=['tools']`、`Config` schema；`apply()` 组装受保护根/分析器/预演/模型复核/引擎/审计，注册 `tools/pre-execute` 监听、审计写入与去重、`systemPrompt` 注入；全量 re-export |
| `src/engine.ts` | `GuardEngine`：单次判定的编排——lex 快速门 →（pwsh）AST → 分级 →（pwsh 删除）预演 → model-check → 决策映射（allow/deny/ask+severity） |
| `src/lexer.ts` | 零成本词法预筛：tokenize/unquote、危险信号扫描（动词族/.NET/diskpart/robocopy/动态标记/通配/git 顶层分派）；`hasDestructiveSignal` 快速放行门 |
| `src/analyzer.ts` | `PwshAnalyzer` + `nodeSpawner` + 内嵌 `ANALYZER_SCRIPT`：`-EncodedCommand`（UTF-16LE base64）拉起 pwsh 只解析不执行，输出紧凑 JSON 报告 |
| `src/preview.ts` | `PreviewRunner`：WhatIf 干跑 + 递归目录只读子树枚举 + 摘要渲染；`parseWhatIfLines`/`parseEnumeration`/`renderPreviewSummary` |
| `src/model-check.ts` | `ModelCheckRunner`：`llm.stream` 三问、`parseModelAnswer` 容错解析、全部失败 fail-closed 为 `unavailable` |
| `src/tiers.ts` | `classifyPwsh`/`classifyBash`：AST 报告 + 词法事实 + 受保护根 → 四档判定（纯函数，无进程） |
| `src/verbs.ts` | 危险动词/别名表：pwsh 删除/格式/回收族、bash 删除/格式族、cmd 开关、`.NET` 删除正则、`iex`、`diskpart clean`、`robocopy /MIR`、`find -delete` |
| `src/protected.ts` | 受保护根注册表（系统根/主目录/额外根 + 结构判定：盘符根/根通配/UNC/扩展根/POSIX 根）+ 归一化/包含关系判定 |
| `src/git.ts` | 顶层 `git` 子命令分派：`rm`（`--cached`/`-n` 非破坏）、`clean`、`reset --hard` 破坏性判定；全局值选项（`-c`/`-C`/`--git-dir`/`--work-tree`/`--exec-path`）消费处理 |
| `src/audit.ts` | `AuditLogger`（串行化 JSONL 追加 + 大小轮转）+ `SessionAuditGate`（每会话上限 + TTL 去重计数） |
| `src/fingerprint.ts` | `fingerprintCommand`（trim + 空白折叠）+ `DedupeWindow`（滑动窗口 TTL 去重） |
| `src/invariant.ts` | 空 invariant 伴生插件（`command-guard-invariant`），注册包名所有权 |
| `src/types.ts` | 共享类型：`PwshReport`/`PreviewOutcome`/`GuardTier`/`SpawnResult`/`Spawner` 等 |
| `patches/careful-full-access.patch` | 宿主核心补丁（见 2.8），相对上游 commit `47f943859` 生成 |
| `harness/package.json` / `harness/tsconfig.json` | 树内安装骨架：包名 `@deepseek-ai/dsh-careful-full-access`、`workspace:^` 依赖、树式 tsconfig 引用 |
| `tests/*.spec.ts` | vitest 单元 + 集成测试（独立仓库 227 通过 + 1 按需跳过；树内 228 全过、100% 覆盖） |
| `package.json` / `tsconfig.json` / `vitest.config.ts` | 独立仓库自验证构建配置（tsc 零错误、vitest run） |

一次判定的调用关系（pwsh 路径）：

```
tools/pre-execute (waterfall)
  └─ GuardEngine.judge
       ├─ lexPwsh ── hasDestructiveSignal? ──否──> allow（零开销放行）
       │    └─是
       ├─ classifyPwsh(词法事实) ── disaster / git 分派已定？──是──> 直接进 review
       │    └─否
       ├─ PwshAnalyzer.analyze（spawn pwsh，Parser::ParseInput 只解析不执行）
       ├─ classifyPwsh(AST 报告 + 词法事实) ── normal ──> allow
       └─ review（elevated/disaster/unparseable）
            ├─ 家族含 delete？ ──> PreviewRunner（WhatIf 干跑 → 子树枚举）
            │     ├─ zero-targets ────────────────> allow（不调模型）
            │     ├─ protected-hit ────────────────> 档位升 disaster
            │     ├─ previewed → scopeSummary / unpreviewable → previewDetail
            ├─ ModelCheckRunner（llm.stream 三问）
            │     ├─ not-intended ──> deny（附模型解释）
            │     ├─ safe ──> elevated: allow；disaster/unparseable: ask(severity:'danger')
            │     ├─ dangerous ──> ask（disaster/unparseable 带 severity:'danger'）
            │     └─ unavailable ──> ask(severity:'danger')，fail-closed
            └─ 审计：AuditLogger.write（轮转文件）+ session.append('command-guard/decision')（有界去重）
```

bash 路径同构，但无 AST、无预演（仅词法分级 → model-check → 决策）。

## 5. 关键实现细节

### 5.1 核心判定流程（步骤化，以 pwsh 为例）

1. 门控：`mode !== 'careful-full-access'` → `next()` 原样放行（模式按**字符串**比较，见 5.5）。
2. `lexPwsh(command)` 得词法事实；`hasDestructiveSignal`（families / netDeleteCall / diskpartClean / robocopyMir / dynamicVerb / git.destructive）为假 → 直接 `allow`——绝大多数命令在此零开销放行。
3. 有信号时先用**纯词法** `classifyPwsh` 快速判定：档位已是 `disaster`（盘符根、格式族、diskpart、robocopy 命中受保护根、递归 .NET 删受保护根等）或有 `git` 分派 → 直接进 review，**不拉起 AST 进程**；否则才 `PwshAnalyzer.analyze` 拿 AST 报告做精判。
4. 精判后 `normal` → allow；否则进 review：pwsh 且家族含 delete → 先 `PreviewRunner.preview`（WhatIf 干跑，见 5.2），再 `ModelCheckRunner.check`。
5. 按 model-check 结果映射决策（见 2.4），`ask` 时按档位决定是否带 `severity: 'danger'`。
6. 决策落定后**先写审计**（文件 + 会话事件，含去重合并），再把决策返回给 `tools/pre-execute` 管线。

### 5.2 关键算法与内嵌脚本

- **AST 分析脚本（ANALYZER_SCRIPT）**：`$env:DGUARD_CMD` 取命令（命令走环境变量、不跨命令行，无引号注入面）→ `[Parser]::ParseInput($cmd, [ref]$tokens, [ref]$errors)` → `$ast.FindAll({$n -is [CommandAst]}, $true)` 遍历所有命令，收集 verb、字面字符串、可展开字符串、变量、参数 → 另用正则找 `.NET` 删除 member calls → 输出一行压缩 JSON（`{ok, parseErrors, commands, memberCalls}`）。**只解析绝不执行**（`$ErrorActionPreference='Stop'`）。
- **AST 报告的容错读取**：`parsePwshReport` 只取**最后一行**非空输出做 JSON 解析，逐字段校验（数组截断上限：命令数组项 64、memberCalls 16），恶意/截断输出 fail-closed 为 `ok:false`。
- **WhatIf 干跑**：`PREVIEW_SCRIPT` = `$WhatIfPreference = $true` + `$ErrorActionPreference='Continue'` + `$ProgressPreference='SilentlyContinue'` 后拼接原命令；正则解析 `What if: ...operation "X" on target "Y"`（英文）与 `假设: 正在目标"Y"上执行操作"X"`（zh-CN）。**注意**：干跑退出码非 0 且无目标 → 判 `unpreviewable`（不是 zero-targets）。
- **子树枚举脚本**：对每个解析出的目录目标 `Get-ChildItem -LiteralPath -Recurse -Force -ErrorAction SilentlyContinue` 计数文件/目录、取前 10 个样例、标记截断；目标不存在则标 `missing` 跳过。
- **model-check 解析**：`parseModelAnswer` 依次尝试 ①原文 ②``` 代码围栏内文本 ③正则抽出的第一个 `{...}` 对象；必须 `intent ∈ {yes,no}`、`assessment ∈ {safe,dangerous}`、`explanation` 非空字符串才接受。
- **git 分派**：首动词为 `git`/`git.exe` 时不再做通用动词扫描；跳过全局值选项后取第一个非选项 token 为子命令；`rm` 带 `--cached`/`--staged`/`-n`/`--dry-run` 非破坏，`reset` 仅 `--hard` 破坏，`clean` 恒破坏。
- **受保护路径判定**：结构判定优先（盘符根 `^[A-Za-z]:\\$`、根通配 `^[A-Za-z]:\\?\*+(?:\.\*+)?$`、UNC `^\\\\[^\\]+\\([^\\]+)\\?$`、扩展根 `^\\\\\?\\[A-Za-z]:\\?$`、POSIX `/`）；注册表比对前做归一化（剥引号、剥尾随 `*` 与分隔符）+ Windows 形式大小写折叠（`isWindowsForm` 才 fold，POSIX 字节精确）；`isInside` 按子串 + 分隔符判定（`\` 或 `/` 按内容选择）。
- **双 ACE ACL（补丁侧）**：`grantWrite` 先 `grantShape` 识别当前 DACL 形态（`exact` 双 ACE / `legacy` 单 ACE / `absent`）——`exact` 直接跳过 SetNamedSecurityInfoW（避免整树重复传播，大工作区可省数分钟）；`legacy` 一次 merge 撤销旧 ACE + 安装双 ACE；`absent` 直接 merge 双 ACE。ACE 的 SID 是内联的（无指针），逐字段偏移读取比对。

### 5.3 重要边界处理与坑（踩过的雷）

- **模式比较用字符串而非类型**：`mode !== 'careful-full-access'` 让仓库能在**公开发布版 DSH 类型**（其 `SandboxMode` 尚无该枚举）下独立编译——这是独立仓库 `pnpm run typecheck` 零错误的基石，也是补丁与插件解耦的关键。
- **删除命令的 AST 必须可信**：`report === undefined || !report.ok` 时，所有非 disaster 的破坏信号一律 fail-closed 为 `unparseable`——"宁多问、不误放"。
- **`iex` 无解**：动态执行动词直接 `unparseable`，payload 静态不可见，绝不静默放行。
- **WhatIf 的副作用**：预演会真实执行删除前的非删除副作用（见 2.3）——不要把预演当完全无副作用；这是文档化的取舍。
- **zh-CN 本地化行**：PowerShell 中文环境的 `What if:` 前缀是 `假设:`，两套正则都要支持，否则中文系统上预演永远"零目标"。
- **干跑非零退出**：有 `$ErrorActionPreference='Continue'` 时命令可能部分失败，无目标 + 非零退出判 `unpreviewable` 而非放行。
- **审计并发**：`AuditLogger` 用 promise 链串行化追加，并发决策不会交错；旋转用 `rename` 移位（`ENOENT` 容忍），大小超限才旋转；所有失败进 `onError`，**绝不抛回判定路径**。
- **会话审计 append 抛错**：try/catch 后仅 warn——判定已成立，审计失败不能翻转结果。
- **`v8 ignore next` 标注的分支**：多为"不可测试的防御分支"（hostile 输入、closed union 穷尽守卫），复刻时保持 fail-closed 语义即可，不必逐行对齐。
- **工作区根 vs 根下内容**：`hitsProtected` 对含通配的目标（`C:\ws\*`）跳过"等于根"判定——通配命中的是**内容**而非根本身；只有字面根才 disaster。
- **补丁内混入配套插件引用（待确认）**：`tsconfig.client.json` 增加的 `./packages/client/ui-settings-restart` 与 `tsconfig.host.json` 增加的 `./packages/host/restart` 两条引用属于配套的 dsh-restart-plugin（与 dsh-plugin-suite 累计补丁合并的产物），**守卫功能本身不依赖它们**；若你的树里没有 restart 包，这两条引用可能使构建报错——复刻时可删除，或按套件方式一并安装（此处理解来自补丁内容与 README 的套件说明，未经独立验证，标注待确认）。

### 5.4 配置项与常量（`Config` schema，全部可选、有默认值）

| 字段 | 默认值 | 含义 |
| --- | --- | --- |
| `extraProtectedPaths` | `[]` | 追加受保护根（绝对路径，追加进注册表） |
| `dedupeTtlMs` | `600_000`（10 min） | 相同命令审计合并窗口 |
| `analyzeTimeoutMs` | `15_000` | AST 分析 spawn 的杀死时限 |
| `previewTimeoutMs` | `15_000` | 每次预演/枚举 spawn 的杀死时限 |
| `previewSampleLimit` | `10` | 预演摘要中的样例路径上限 |
| `modelCheckTimeoutMs` | `20_000` | 整个 model-check 调用的时限（fail-closed） |
| `modelCheckMaxTokens` | `300` | model-check 输出预算 |
| `auditLogPath` | `''` → `join(resolveDshHome(), 'logs', 'command-guard.log')` | 审计文件路径 |
| `auditLogMaxBytes` | `5 * 1024 * 1024` | 轮转阈值 |
| `auditLogRotations` | `3` | 保留轮转份数（`.1`…`.N`） |
| `sessionDecisionCap` | `20` | 每会话 `command-guard/decision` 事件上限 |
| `pwshPath` | `'pwsh'` | PowerShell 辅助可执行 |
| `enablePrompt` | `true` | 是否注册删除纪律提示段落（order 112） |

其他关键常量：`PROMPT`（删除纪律文本，注入 order 112）；补丁侧 `ROOT_GRANT_MASK = 0x00100116`、`GRANT_MASK = 0x00110156`、`INHERIT_ONLY_ACE = 0x8`。

### 5.5 与 DSH 版本的耦合点（随版本漂移清单）

- **必须由补丁提供的宿主改动**（描述基于补丁相对上游 `47f943859`——`Merge pull request #2519 from deepseek-harness/feat/npm-public`）：`SandboxMode` 第四档、`isUnconfinedMode`、`SANDBOX_MODES`、权限预设、`PreToolDecision.severity`、approval→api-proxy→client 的 severity 链路、`PermissionSelect` 档位与图标、`cordis.patch.yml` 挂载行、`base/package.json` 依赖、ACL 双 ACE、tsconfig 引用。上游前进后 `git apply` 可能不干净，需手工合并（改动点均为组合注册/依赖声明/tsconfig 引用）。
- **插件侧只读的公开 API**（随 DSH 发布版变化）：`tools/pre-execute` 事件契约、`PreToolDecision` 形状、`sandboxPolicy.resolve` 返回、`llm` 服务 `stream`/`createUserMessage`、`session.append`、`systemPrompt.context`、`invariants.register`、`resolveDshHome`。仓库用**字符串比较**隔离了模式枚举漂移，其余 API 以 `package.json` 中 `peerDependencies`（`^0.1.0-rc.6` 系）为准。
- **插件与补丁的版本配套**：补丁与 src/ 在同一仓库、同一 commit 维护（本次描述 commit `07a00f1ad`，补丁基线上游 `47f943859`）。

## 6. 集成与安装

> 对复刻 AI：请按本描述第 2–5 节生成等价实现（TypeScript 源码 + 补丁），再按下述步骤接入。

1. **AI 产出**：`src/` 全部模块（含 `index.ts`/`engine.ts`/`lexer.ts`/`analyzer.ts`/`preview.ts`/`model-check.ts`/`tiers.ts`/`verbs.ts`/`protected.ts`/`git.ts`/`audit.ts`/`fingerprint.ts`/`invariant.ts`/`types.ts`）、`tests/`、`patches/careful-full-access.patch`、`harness/` 骨架（`package.json` + `tsconfig.json`）。独立自验证：`pnpm install && pnpm run typecheck && pnpm test`（对照：227 通过 + 1 按需跳过）。
2. **准备 DSH 源码树**：克隆官方上游 [deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（补丁基线 `47f943859` 附近；上游已前进时手工合并，见 5.5）。
3. **应用补丁**：`git apply patches/careful-full-access.patch`——注册模式/预设/UI/红色链路/ACL/挂载行。若你的树不含配套 restart 包，删除补丁中 `ui-settings-restart` 与 `packages/host/restart` 两条 tsconfig 引用（见 5.3 待确认项）。
4. **放入包**：把 `src/`、`tests/` 复制到 `packages/guard/careful-full-access/`，用 `harness/package.json` 与 `harness/tsconfig.json` 替换该包骨架（`workspace:^` 依赖、树式引用）。
5. **构建与重启**：仓库根 `pnpm install && pnpm run build`（或 `pnpm dsh web` 前构建），重启后端。
6. **切换模式**：把会话权限切到 `careful-full-access`，守卫即对每条 `pwsh`/`bash` 调用生效。
7. **验证**（见第 8 节检查单）：零风险冒烟 `Remove-Item -Recurse -Force Z:\`（不存在的盘符）→ 应出现灾难级复核、命令未执行；`git rm -r --cached src` → 直接放行；`Get-ChildItem C:\ws` → 放行且审计有 `decision: 'allow'`。
8. **树内测试**（可选）：应用补丁的环境下全量 228 例（100% 行/分支/函数覆盖）；`DSH_GUARD_CORE_PATCH=1 pnpm test` 强制跑红色链路用例。

**npm 方式暂不可用**：`dsh-careful-full-access` 未发布 npm；发布前需构建 `lib/`（`main`/`types` 指向 `lib/`），当前请使用源码树方式（待确认）。

## 7. 已知边界与注意事项

- **`iex`/脚本块动态构造无法静态分析** → fail-closed 按 disaster（人工确认，`never` 下拒绝）。
- **bash 无 WhatIf 等价物**：POSIX 上复核不带解析出的范围摘要；bash 也无 AST 精析（只靠词法）。
- **只有顶层 `git` 调用获得子命令分派**：管道或嵌套的 `git` 退回通用扫描，可能误读其子命令语义（宁可误报）。
- **model-check 成本**：每条被标记命令消耗一次模型调用（延迟与 token），判断质量取决于复核模型——这正是 disaster 档与模型自称危险的命令**永远以人工收尾**的原因；`unavailable` 时 fail-closed 为人工。
- **careful 模式本质是全权限**：文件访问不受沙箱限制，守卫是删除防护的**唯一**主防线；第二道防线是补丁的 ACL 双 ACE（工作区根本身删不掉），但守卫被绕过时其余文件仍无保护。
- **预演副作用**：WhatIf 干跑会真实执行删除语句之前的非删除副作用（见 2.3/5.3）。
- **未实现项**（README 明示）：manual/auto 确认策略、持久化规则表（"始终允许此模式"）、软删除恢复层——均为后续项，复刻时无需实现。
- **待确认事项汇总**：① npm 发布状态与 `lib/` 构建（当前不可用）；② 补丁内 `ui-settings-restart`/`host/restart` 两条 tsconfig 引用的归属与独立使用时的处理；③ 补丁对晚于 `47f943859` 的上游版本的手工合并点（结构清晰但需人工核对）；④ 挂载行 `name: 'dsh-careful-full-access'` 与 `harness/package.json` 包名 `@deepseek-ai/dsh-careful-full-access` 的命名关系（README 以挂载行/依赖名为准，未深入核对包名一致性问题）。

## 8. 复刻验收标准（给复刻 AI 的检查单）

- [ ] **模式门控**：`workspace-write` 与 `danger-full-access` 模式下 `Remove-Item -Recurse -Force C:\` 原样执行、零守卫、零审计事件（对照 index.spec.ts「passes every other sandbox mode through」）。
- [ ] **快速放行**：`Get-ChildItem C:\ws` 放行并产生 `command-guard/decision`（`decision:'allow'`）；`git rm -r --cached src` 放行（normal，不拉起 pwsh 辅助进程）。
- [ ] **disaster 红色链路**：`Format-Volume D`（或 `Remove-Item -Recurse -Force Z:\` 冒烟）→ 档位 `disaster`、模型判 safe 后仍 `ask` 且 `severity:'danger'`，理由含 `DISASTER tier`；审批 `never` 时自动拒绝。
- [ ] **模型否认即拒**：模型回 `{"intent":"no",...}` → 直接 `deny`，理由含模型自己的解释，不再人工确认。
- [ ] **elevated 放行**：`Clear-RecycleBin -Force` 且模型判 safe → `allow`（`tier:'elevated'`, `modelCheck:'safe'`）。
- [ ] **零目标预演放行**：干跑无目标 → `allow` 且**不调用** model-check（engine.spec.ts「allows zero-target previews」）。
- [ ] **预演命中受保护根升级**：干跑解析出受保护根 → 档位升级 `disaster` 的红色 ask。
- [ ] **bash 同构**：`rm -rf /` → `ask`、`tier:'disaster'`、红色；bash 无预演无 AST。
- [ ] **审计双重写入**：轮转文件出现完整 JSONL 判定行；相同命令 TTL 内第二次仅追加 `{"event":"repeat","count":2}`；会话事件不超过 `sessionDecisionCap`（默认 20）。
- [ ] **fail-closed**：AST 分析失败/模型不可用/超时/答案解析失败 → 一律人工确认（红色），绝不静默放行。
- [ ] **提示注入**：`enablePrompt` 默认 true 时 systemPrompt 装配结果含 `Deletion discipline (enforced by the command guard`；false 时不含。
- [ ] **补丁生效**：应用补丁后 `SandboxMode` 含 `careful-full-access`、UI 权限选择出现第四档（眼睛图标）、`cordis.patch.yml` 含 `id: command-guard` 挂载行、审批面板支持红色（`DSH_GUARD_CORE_PATCH=1` 下红色链路测试通过）、工作区根 ACL 为双 ACE（根无 DELETE）。
- [ ] **测试对齐**：独立仓库 `pnpm run typecheck` 零错误、`pnpm test` 227 通过 + 1 按需跳过；树内 228 例、100% 覆盖。
- [ ] **结构/集成点**：模块划分、配置项默认值、事件名（`tools/pre-execute`、`command-guard/decision`）、插件名（`command-guard`）与挂载方式与本描述一致。
