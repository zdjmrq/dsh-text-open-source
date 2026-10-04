> [!WARNING]
> **本仓库已废弃** —— 相关能力已随 dsh 正式版本内置发布，无需再安装本插件；仓库仅作历史存档，不再维护。

# dsh-text-open-source

> **DSH 插件「文字开源」枢纽** —— 不存代码，只存"可以复刻插件"的文字描述（提示词）。
> 每个描述文件 = 一个插件的完整说明书：功能、技术路线、结构、关键实现、复刻验收标准。
> 把描述粘贴进你的 DSH 会话，AI 就能复刻出功能相同的插件；你也能读懂原理，按需微调。

本仓库同时维护一份 [人类可读实现原理解释规范](docs/implementation-explanation-standard.md)，用于指导 AI 以“最少充分解释”描述代码，使人类无需深入源码也能准确理解系统的主要机制、判断、状态、失败方式和能力边界。

---

## 为什么是"文字开源"？

我们把 DSH 插件代码在 GitHub 上开源后，发现**代码开源 ≠ 能力开源**：

- **代码黑箱**：光看源码，很难完整掌握插件的功能边界、技术架构和设计取舍；别人拿到代码也未必敢改。
- **分发受限**：一部分功能依赖宿主内部改动或与特定 DSH 版本强耦合，难以用独立仓库干净分发。
- **理解成本高**：代码是写给机器读的，人要从几千行源码里提炼"它到底做了什么、怎么做到的"非常费力。

所以我们换一种方式开源：**让 AI 通读源码，把插件"翻译"成一份自包含的文字描述**。这份描述就是"提示词"，也是插件的"数字孪生说明书"：

| 优势 | 说明 |
| --- | --- |
| **跨版本兼容最好** | 文字不绑定某个 DSH 版本的构建产物，描述里的思路、接口、结构可以迁移到任何版本复现 |
| **人类可读、可微调** | 描述按"功能 → 技术 → 结构 → 细节"组织，人读一遍就懂；想改需求，直接改描述再让 AI 重做 |
| **保存量极小** | 一个插件描述只有几 KB，而代码仓库动辄几 MB；整个生态的文字开源可以收敛到这一个仓库 |
| **AI 时代原生** | 描述即程序——粘贴即复刻，且复刻过程本身就带着"为什么这样做"的完整上下文 |

> 一句话：**代码会过时，描述会进化。** 本仓库与各插件源码仓库互为镜像——代码是"实现"，这里是"理解"。

## 怎么用

1. 在下方索引表找到你要的插件，打开它的描述文件；
2. **全文复制**描述文件的内容；
3. 粘贴到你自己的 DSH 会话，告诉 AI：*"请按以下描述为我复刻这个插件"*；
4. AI 产出插件文件后，按描述第 6 节「集成与安装」接入你的 DSH，按第 8 节「复刻验收标准」逐项验收。

## 两种说明层级

| 层级 | 面向对象 | 目标 | 推荐文件 |
| --- | --- | --- | --- |
| **实现原理说明** | 希望理解和判断代码的人类 | 用最少充分信息解释系统为什么有效、如何运行、如何失败以及能力边界 | 项目中的 `HOW_IT_WORKS.md` 或 `实现原理.md` |
| **文字开源规格** | 需要复刻或迁移功能的 AI 与实现者 | 完整描述行为、技术路线、结构、集成方式和验收标准 | 本仓库的 `plugins/*.md` |

实现原理说明不追求足以复刻代码，文字开源规格也不应替代面向人类的简洁原理说明。两者使用不同的信息密度，共享相同的实现事实。

编写实现原理说明时，请遵循 [docs/implementation-explanation-standard.md](docs/implementation-explanation-standard.md)。核心原则是：每个模块必须提供理解其作用所需的最少充分解释；触发条件、判断规则、失败处理、状态、副作用、时序和能力边界，仅在它们会改变读者对行为的预测时出现。

## 插件索引（枢纽 ↔ 源码 ↔ 描述）

每个插件在下方同时给出**文字开源描述**（本仓库）与**源码仓库**（独立仓库），两处互相链接：

| 插件 | 文字开源描述 | 源码仓库 | 形态 | 一句话简介 |
| --- | --- | --- | --- | --- |
| dsh-chat-mode | [plugins/dsh-chat-mode.md](plugins/dsh-chat-mode.md) | [dsh-chat-mode](https://github.com/zdjmrq/dsh-chat-mode) | agent preset + 宿主/客户端补丁 | 「对话」ChatGPT 纯聊天模式：新会话按当前模式直开、模式小三角切换 DSH/对话，对话会话仅提问+搜索工具，专属 `$DSH_HOME/chat` 聊天工作区 |
| dsh-intercom | [plugins/dsh-intercom.md](plugins/dsh-intercom.md) | [dsh-intercom](https://github.com/zdjmrq/dsh-intercom) | Cordis 双半 | 顶层对话间通信中心：聊天面板 + `intercom_*` 工具（求助/协作/唤醒休眠会话） |
| dsh-usage-balance | [plugins/dsh-usage-balance.md](plugins/dsh-usage-balance.md) | [dsh-usage-balance](https://github.com/zdjmrq/dsh-usage-balance) | Cordis Client 半 | 侧边栏「用量 / 余额」标签行 + 悬停详情卡 |
| dsh-careful-full-access | [plugins/dsh-careful-full-access.md](plugins/dsh-careful-full-access.md) | [dsh-careful-full-access](https://github.com/zdjmrq/dsh-careful-full-access) | 插件 + 宿主补丁 | 命令守卫：静态分级 + WhatIf 预演 + model-check 复核，中文准确说明待批命令与删除范围 |
| dsh-restart-plugin | [plugins/dsh-restart-plugin.md](plugins/dsh-restart-plugin.md) | [dsh-restart-plugin](https://github.com/zdjmrq/dsh-restart-plugin) | Cordis Client 半 | 设置页一键「关闭后台服务 / 刷新前端」，刷新保留热插件 |
| dsh-plugin-suite | [plugins/dsh-plugin-suite.md](plugins/dsh-plugin-suite.md) | [dsh-plugin-suite](https://github.com/zdjmrq/dsh-plugin-suite) | 改动切片套件 | 定制插件套件（局部 fork）：只带改动切片 + 累计补丁，收纳需动宿主的插件 |
| dsh-pluginmanager | [plugins/dsh-user-plugins-manager.md](plugins/dsh-user-plugins-manager.md) | [dsh-user-plugins-manager](https://github.com/zdjmrq/dsh-user-plugins-manager) | Cordis 双半 | 设置→插件 新增「插件管理」页：原生三层 / 用户扩展 / 运行中临时，支持启停、卸载、补登记、描述编辑 |
| dsh-attention-notifier | [plugins/dsh-attention-notifier.md](plugins/dsh-attention-notifier.md) | [dsh-attention-notifier](https://github.com/zdjmrq/dsh-attention-notifier) | Cordis 宿主半 | 微信式任务栏提醒（判定端），配合 dsh-shell 呈现 |
| dsh-shell | [plugins/dsh-shell.md](plugins/dsh-shell.md) | [dsh-shell](https://github.com/zdjmrq/dsh-shell) | Electron 桌面壳 | 把 DSH Web UI 装进原生窗口，只注入窗口边框层，不碰页面 UI |

## 双向链接

本仓库是生态的**枢纽**：向上连接各插件源码仓库（见索引表），向下连接每个插件的文字描述。

- **仓库 → 枢纽**：每个插件源码仓库的 README 均带有「📖 文字开源描述」段落，指向本仓库对应描述文件（见各仓库 README）。
- **枢纽 → 仓库**：本 README 索引表给出每个插件的源码链接，描述文件头部也标注对应源码与 commit。
- 修改任一插件的功能后，请同步更新其描述文件（描述基于的 commit 见文件头部）。

## 如何为你的插件添加文字开源描述

1. 复制 [docs/template.md](docs/template.md) 作为模板；
2. 通读插件源码（README、源码、配置文件），完整理解功能 / 技术路线 / 结构 / 关键实现；
3. 让 AI 按模板 8 节结构撰写描述，写入 `plugins/<插件名>.md`；
4. 头部标注对应源码 commit，按第 8 节验收复刻效果；
5. 在下方索引表登记一行，并在插件源码仓库 README 中添加指向本仓库的链接。

## 许可证

[MIT](LICENSE) © 2026 zdjmrq —— 描述文件与代码同样开放，可自由使用、修改、再分发。
