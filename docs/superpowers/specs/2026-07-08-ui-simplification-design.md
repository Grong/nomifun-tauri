# UI 简化设计：降低用户认知负担

> 日期：2026-07-08
> 状态：设计阶段（修订版 v2）
> 关联：无

## 问题

NomiFun 功能过于丰富，导致三个互相关联的问题：

1. **新用户上手困难** — 打开应用面对 13 个侧边栏入口和 5 个分组标题，不知道从哪开始
2. **大量功能闲置** — 许多功能用户从未触及，却占用界面和认知空间
3. **概念边界模糊** — 「伴侣 vs 智能体 vs 助手 vs 技能」等概念用户难以区分

当前侧边栏结构（13 个入口）：

```
常用          → 会话、桌面伙伴、创意工坊
对外服务      → 对外伙伴
数据空间      → 知识库、资产库
自动化        → 定时任务、需求平台
增强工具      → 助手&Skill、MCP
────────────────────────────
设置区        → 模型&Agent、开放能力、设置
```

## 设计目标

- 侧边栏从 13 个入口精简到 4 个固定入口 + 动态项目列表
- 新用户安装 → 填 API key → 开始聊天，中间零配置
- 功能通过对话自然暴露，用户无需预先理解概念

## 一、侧边栏

```
┌─────────────┐
│             │
│  + 新建对话  │  ← 固定操作
│  📚 知识库   │  ← 固定入口
│  🔌 插件     │  ← 固定入口（MCP + Skill + 扩展，统一管理）
│             │
│  项目        │  ← 分组标题，有项目时显示
│  📁 项目A    │  ← 动态列表：每个活跃工作目录一个条目
│  📁 项目B    │
│             │
│             │
│             │
│             │
├─────────────┤
│  ⚙️ 设置     │
└─────────────┘
```

### 入口变化

| 当前入口 | 变化 |
|----------|------|
| 会话 | 改为「+ 新建对话」操作入口，会话列表在内容区管理 |
| 知识库 | 保留在侧边栏 |
| 桌面伙伴 | 默认自动创建，无需入口；管理项在设置中 |
| 创意工坊 | 移入设置 |
| 对外伙伴 | 移入设置 |
| 资产库 | 合并到创意工坊设置中 |
| 定时任务 | 移入设置 |
| 需求平台 | 移入设置 |
| 助手&Skill | 合并到「插件」入口 |
| MCP | 合并到「插件」入口 |
| 扩展 | 合并到「插件」入口 |
| 模型&Agent | 移入设置 |
| 开放能力 | 移入设置 |

### 插件入口

将 MCP 服务器、Skill 包、扩展市场统一收纳。用户心智模型：**「给 AI 装插件」**。

**路由设计**：统一使用新的顶层路由 `/plugins`，内部以 Tabs 承载三个子系统：

```
/plugins
  ├── ?tab=mcp     → McpPage（已有 HubPageShell + ToolsModalContent）
  ├── ?tab=skills  → SkillsHubSettings（现有页面，从 /assistants?tab=skills 移入）
  └── ?tab=extensions → 扩展市场页（新增：列出已安装扩展，入口到各扩展独立设置）
```

**实现策略**：不合并三个子系统的渲染架构。通过 Tab 切换让各子系统保持其现有渲染方式不变（MCP 用 HubPageShell + Modal，Skills 用 HubPageShell + Tabs，Extensions 保留 iframe/webview 机制）。仅在 Tab 层面做统一导航。原有路由 `/mcp`、`/assistants?tab=skills` 保留重定向到 `/plugins?tab=xxx`。

**扩展市场新建**：该 Tab 为新增页面，展示已安装扩展列表，每个扩展链接到 `/settings/ext/:tabId`（保持现有 ExtensionSettingsPage 不变）。

**E4 — 插件推荐**：用户首次进入 `/plugins` 时，若本地未安装任何 MCP 服务器 / Skill 包 / 扩展，展示推荐面板。详见第八章。

### 项目列表

动态分组：复用现有 `projectWorkpaths.ts`（`ui/src/renderer/pages/conversation/SessionList/utils/projectWorkpaths.ts`）作为数据源。项目条目即已有的 project workpath 条目。

**与 SessionCreateBar 的关系**：`SessionCreateBar.tsx` 当前包含 `onNewChat`、`onNewTerminal`、`onCreateProject` 三个创建操作。主侧边栏接管「+ 新建对话」和「新建终端」后，`SessionCreateBar` 中移除 `onNewChat` 和 `onNewTerminal`，仅保留 `onCreateProject`、搜索、显示设置和批量操作。如此避免两处重复暴露相同操作。

**数据流**：
- 主侧边栏 `useEffect` 中订阅 `subscribeProjectWorkpaths()` 获取项目列表
- 点击项目条目导航至对应会话（复用 `ConversationShell` 的现有过滤逻辑）
- 无项目时不显示「项目」分组标题和条目

### 会话列表访问

「+ 新建对话」创建新会话并导航至 `/guid`（此时 ConversationShell 挂载，ContentSider 展示会话列表）。会话列表与对话内容共存于 `ConversationShell` 内，如同 IDE 的文件树与编辑区的关系。

**兜底方案**：在「+ 新建对话」旁边增加一个「历史」图标按钮（仅 icon，不展开列表），hover 时 tooltip 显示最近 3 个会话标题，点击跳转。此方案在实施时根据交互测试决定是否保留，不阻塞初始发布。

## 二、默认开启

用户安装后无需手动配置以下功能，开箱即用：

| 功能 | 默认策略 |
|------|----------|
| 桌面伙伴 | 首次启动自动创建默认伙伴，预置人设和头像 |
| Agent | 内置 `nomi` Agent 预配置完成 |
| Assistant | 「通用助手」preset 默认生效 |
| 知识库 | 预创建空的「我的知识库」 |
| 电脑/浏览器操控 | 桌面版默认可用 |

用户唯一需要做的：填写 API key，然后开始聊天。

### 初始化机制

首次启动检测：在 App 根组件挂载时，检查 `localStorage` 中的 `nomifun:initialized` 标记。若不存在则触发初始化流程。

**初始化流程**（顺序执行，任一失败即跳过不阻塞启动）：

1. 检查 `localStorage` 是否存在 `nomifun:initialized`
2. 若不存在，依次调用：
   - `ipcBridge.companion.createCompanion.invoke({ name: '默认伙伴', character: '...' })` — 创建默认桌面伙伴（API: `ipcBridge.ts:3349`）
   - 知识库预创建 — 调用知识库创建 API，若 API 暂不可用则标记 TODO
   - Agent/Assistant 预设由后端预配置完成（前端无需动作）
3. 写入 `localStorage.setItem('nomifun:initialized', '1')`
4. 若初始化过程中任一 API 调用失败，catch 错误记录日志但不阻塞应用启动

**重新安装/第二设备**：`nomifun:initialized` 跟随浏览器 localStorage。新设备自然触发初始化。若用户手动清除 localStorage，也会重新初始化——创建重复的默认伙伴，由后端去重或前端检查已有伙伴列表后再决定是否创建。

## 三、会话页简化

### 新建会话

从「配置向导」变为「直接开始」：

- 默认就是空白聊天框，用户直接打字
- 背后自动使用：`nomi` Agent + 通用助手 + 我的知识库
- 高级配置（模型选择、Agent 类型、知识库挂载、MCP 选择）折叠在「高级配置」中

### 能力自然暴露

Agent 的能力在对话中自然体现，用户无需预先理解概念：

- 用户说「帮我写个文档」→ agent 自动用文件工具
- 用户说「查一下知识库里的 xxx」→ agent 自动调知识库搜索
- 用户说「帮我打开浏览器查一下」→ agent 自动用浏览器操控

用户不需要知道这些能力叫「MCP」「Skill」「Tool」。

## 四、设置页重组

### SettingsSider 集成方案

**修订后的 BUILTIN_TAB_IDS**（在现有 `SettingsSider.tsx:25` 的 `['system', 'agent-runtime', 'browser-use', 'computer-use', 'about']` 基础上插入新条目）：

```
['system', 'agent-runtime', 'browser-use', 'computer-use',
 'nomi',            // 桌面伙伴管理（从 /nomi 移入）
 'requirements',    // 需求 & 自动化（汇总定时任务 + AutoWork）
 'public-companions', // 对外伙伴（从 /public-companions 移入）
 'workshop',        // 创意工坊 & 资产库（从 /workshop, /assets 移入）
 'about']
```

### 路由变更

| 旧路径 | 新路径 | 说明 |
|--------|--------|------|
| `/nomi` | `/settings/nomi` | 桌面伙伴管理页 |
| `/workshop` | `/settings/workshop` | 创意工坊画廊 |
| `/workshop/:id` | `/settings/workshop/:id` | 创意工坊画布编辑 |
| `/assets` | 合并入 `/settings/workshop` | 资产库作为 Workshop 设置的子 Tab |
| `/scheduled` | `/settings/requirements?tab=scheduled` | 定时任务 |
| `/scheduled/:job_id` | `/settings/requirements?tab=scheduled&job=:id` | 定时任务详情 |
| `/requirements` | `/settings/requirements` | 需求平台（保留 RequirementsLayout 内嵌路由） |
| `/requirements/*` | `/settings/requirements/*` | 继承所有子路由 |
| `/public-companions` | `/settings/public-companions` | 对外伙伴列表 |
| `/public-companions/:id` | `/settings/public-companions/:id` | 对外伙伴详情 |
| `/mcp` | `/plugins?tab=mcp` | 重定向 |
| `/assistants` | `/plugins?tab=skills` | 重定向 |
| `/open-capabilities` | 合并入 `/settings/system` | 开放能力 → 系统设置子 Tab |

**所有旧路由保留 301 重定向**，确保旧书签不中断。

### 关于创意工坊移入设置的理由

Workshop 虽然是一个完整应用（3 个路由、Canvas 编辑器），但其入口在侧边栏的「常用」分组中与「会话」「桌面伙伴」并列，增加了认知负担。将此入口移至设置是本次「降低新用户认知负担」目标的直接体现——新用户不应在第一时间面对创意工坊的复杂性。活跃 Workshop 用户可通过设置的「创意工坊 & 资产库」入口继续使用，路径深度仅增加一层点击。若反馈表明此决策负面影响显著，后续的「高级模式」可作为缓解方案。

## 五、不变的部分

以下功能保持现状，本次不做改动：

- 后端 API 和 agent 能力
- 桌面版电脑/浏览器操控的底层实现
- 11 个 IM 频道（保留在对外伙伴设置中，路径迁移至 `/settings/public-companions` 下的频道子页）
- 终端模式（通过 `ConversationShell` 中的 `ContentSider` 访问，路径 `/terminal-new`, `/terminal/:id` 不变）

## 六、迁移与渐进式发布

### 功能开关

引入 `localStorage` 中的 `nomifun:sidebar-v2` 标记：

- **未设置或 `'auto'`**（默认）：显示新侧边栏（4 固定 + 项目）
- **`'classic'`**：显示旧侧边栏（13 入口 + 分组标题）
- 通过设置页的系统设置中增加一个「导航模式」切换项，允许用户手动切换

### 发布策略

1. **Phase 1 — 暗发布**：默认使用新侧边栏，设置中提供切回「经典导航」的选项
2. **观察期**：收集反馈 2-4 周，监控设置中功能的使用频率变化（通过 `/settings/*` 路由的 PV 对比旧路由 PV）
3. **决策点**：若 Workshop/定时任务/对外伙伴的 PV 显著下降，考虑：
   - 为高频功能保留侧边栏快捷方式
   - 或启用「完整导航」模式

### 回滚路径

- 若出现严重用户投诉，通过设置中的导航模式切换即可恢复旧侧边栏，无需代码回滚
- 若需紧急回滚，将 `nomifun:sidebar-v2` 的默认值改为 `'classic'` 的下一个热修复版本即可

## 七、E1 — 新手引导向导（Onboarding Wizard）

### 触发条件

用户首次启动时。与「默认开启」共用 `nomifun:initialized` 标记：
- 若标记不存在 → 投递初始化流程 + 展示向导
- 向导完成后 → 写入标记

### 向导步骤

1. **欢迎页**：「欢迎使用 NomiFun，AI 助手已就绪。」— 展示默认伙伴形象
2. **API Key**：输入 API Key（必填，省略此步则无法使用）
3. **快速开始**：「试试跟我说：帮我写一份周报」— 引导用户发送第一条消息

**不含高级配置**：向导阶段不暴露 Agent 选择、知识库配置、MCP 等内容。用户在后续使用中通过对话自然发现这些能力。

### 实现

新建 `OnboardingWizard` 组件，在 `Router.tsx` 中增加 `/onboarding` 路由。App 根组件挂载时检测 `nomifun:initialized`，若不存在则导航至 `/onboarding`。向导不可跳过，但总共 3 步，预计耗时 < 30 秒。

### 状态持久化

完成向导后写入 `localStorage.setItem('nomifun:initialized', '1')`。重装/清除 localStorage 时会重新触发向导。

## 八、E4 — 插件推荐（Plugin Discovery）

### 触发条件

用户首次进入「插件」入口（`/plugins`）时，若本地未安装任何 MCP 服务器 / Skill 包 / 扩展，展示推荐面板。

### 推荐内容

- **推荐 MCP**：filesystem, fetch, git（硬编码在 `recommendedPlugins.ts`）
- **推荐 Skill**：code review, diagram
- 每个推荐项包含：名称、简介、安装按钮

### 安装流程

点击「安装」→ 分别调用：
- MCP：现有 `ToolsModalContent` 的添加逻辑
- Skill：现有 `AgentSkillImportDrawer` 的导入逻辑
- 扩展：navigate 至扩展市场

### 状态持久化

- `localStorage.setItem('nomifun:plugin-recommendations-dismissed', '1')` 记录用户是否已关闭推荐
- 已安装任意插件后，不再展示推荐面板
- 用户可手动关闭推荐面板，关闭后仅在「插件」页底部保留「查看推荐插件」链接

## 九、后续可做的

本次 UI 简化之后，可观察用户行为和反馈，进一步决定：

- 哪些设置中的功能可以彻底删除（而非仅仅隐藏）
- 是否需要为「高级用户」提供快捷方式切换回「完整导航」模式
- 插件入口是否需要扩增推荐插件的范围
