# ch13: SubAgent Spec

## 1. 背景

主 Agent 做大任务时会塞满上下文：研究、规划、写代码、跑测试都堆在一个对话里，单一窗口很快耗尽。这一章把"开一个上下文隔离的新 Agent 去做一件事"做成主 Agent 可以直接调用的工具，让主 Agent 学会分发工作，避免上下文污染，同时通过专门角色（Plan / Explore）和后台异步执行扩展并发能力。

## 2. 目标

提供 `Agent` 工具，主 Agent 在对话里写一次工具调用即可：1) 按 `subagent_type` 启动一个定义式专家子 Agent（系统提示词、模型、工具白名单都按 Markdown 定义文件来），2) 不带 `subagent_type` 且 `enable_fork=true` 时直接 fork 当前对话上下文跑一个临时子 Agent，3) 带 `team_name` 时把这个 spawn 注册成长期团队成员（衔接 ch15）。后台任务的完成通过 `<task-notification>` 反馈给主 Agent。

## 3. 功能需求

- F1: `AgentTool` 继承 `mewcode.tools.base.Tool`，注册到主 Agent 的 `ToolRegistry`，被 LLM 当成普通工具调用。
- F2: 三档内建 Agent 类型 `general-purpose` / `Plan` / `Explore`，每档可定制工具黑名单、最大轮数、模型、系统提示词。
- F3: 支持从用户级目录 `~/.mewcode/agents/*.md` 和项目级 `<work_dir>/.mewcode/agents/*.md` 加载自定义 Agent 定义；项目级覆盖用户级覆盖 builtin；Markdown frontmatter 解析为 `AgentDef`。
- F4: 三种执行路径：sync（前台阻塞、`await sub_agent.run_to_completion()`）/ background（asyncio task，立即返回任务 ID）/ fork（fork 父对话上下文，强制后台）。
- F5: `TaskManager` 跟踪后台子 Agent 生命周期（running / completed / failed / cancelled），完成时把任务 ID 写进 `asyncio.Queue`，主 Agent 下一轮通过 `poll_completed` 拿到，再用 `inject_task_notifications` 拼装 `<task-notification>` 注入对话。
- F6: 四层工具过滤：MCP 直通、全局禁（`ALL_AGENT_DISALLOWED_TOOLS` 七项，含 `Agent` / `AskUserQuestion` 防递归）、custom agent（`source != "builtin"`）额外禁、background 白名单（`ASYNC_AGENT_ALLOWED_TOOLS` 16 项）、definition 级 `disallowed_tools` + `tools` 白名单。
- F7: Fork 路径：构造完整 forked `ConversationManager`（`copy.deepcopy(history)` + 给悬挂的 `tool_use` 补 `"interrupted"` placeholder `ToolResultBlock`），追加 `FORK_BOILERPLATE` + `"你的任务：\n" + task`；fork-of-fork 通过扫描对话历史 `FORK_BOILERPLATE_TAG` 拒绝。
- F8: 可选 worktree 隔离与 `WorktreeManager` 配合，子 Agent 在临时 git worktree 中跑；执行结束按 `auto_cleanup` 返回的 `kept` 标志决定是否在结果里追加 `[Worktree preserved at ...]` 提示。
- F9: 可选团队模式与 `TeamManager` 配合，走 `_execute_as_teammate` 路径注册长期团队成员，按 backend（in-process / tmux / iterm2）路由（详见 ch15）。
- F10: 父 Agent 取消（中断）时，`TaskManager.adopt_running` 把当前正在跑的 Agent 实例挂为后台任务并继续执行，主流程不阻塞。
- F11: `AgentTool` 入口额外支持 `model`（运行时模型覆盖）/ `isolation`（仅 `worktree`）/ `name`（团队场景标识）参数；`isolation` 与 `team_name` 互斥（团队场景走自己的 worktree）。
- F12: 子 Agent 定义 frontmatter 可声明 `background: true` 强制后台运行，与调用侧 `run_in_background=true` 等价。
- F13: 可选 Verification 内置角色（找最后 20% bug），由 `enable_verification` flag 守开关；默认不出现在 Agent 列表里。
- F14: Fork 子 Agent 的工具池继承父池经四层过滤（MCP 直通 + 全局黑 + 白名单 + 定义级），让 API 请求前缀字节级一致以命中 prompt cache；嵌套 fork 通过扫描父对话消息内容 `FORK_BOILERPLATE_TAG` 阻止。
- F15: Fork 复制父对话用 `copy.deepcopy`，保留每条 `Message` 的全部字段（含 `tool_uses` / `tool_results` / `thinking`），保证 assistant 消息形状与父侧一致。
- F16: 子 Agent 接受 spec 级 `permission_mode` 覆盖，运行时用独立的 `PermissionChecker`，与父共享 `DangerousCommandDetector` / `RuleEngine` 类型，但 `PathSandbox` 按子 Agent 的 `work_dir` 重新分配。
- F17: `TraceManager` 给每个 spawn 出来的子 Agent 创建 `TraceNode`，父 / 子 / trace ID 三元组打通，配合 `trace` 命令做调用树查询。

## 4. 非功能需求

- N1: 子 Agent 不能再调 `Agent` 工具（防止无限递归 / 上下文爆炸），任意层级的子 Agent 都通过 `ALL_AGENT_DISALLOWED_TOOLS` 屏蔽。
- N2: 后台 Agent 通过 `asyncio.Task.cancel()` 受控；取消调用后状态置为 `cancelled`。
- N3: `TaskManager` 在 asyncio 单线程模型下顺序安全，`_tasks` / `_async_tasks` / `_notify_queue` 必须在事件循环内访问。
- N4: fork 操作必须先在父对话历史里扫 `FORK_BOILERPLATE_TAG` 字面量，命中即 `raise ForkError`。
- N5: Sync 路径要 `await` 子 Agent 的 `run_to_completion` 直到返回，不丢消息；异常路径要把 `trace_node` 标 `failed` 再向上抛。
- N6: Fork 子 Agent 必须复用父池工具与对话内容（含 thinking blocks），让请求前缀字节级一致；任何额外过滤都会破坏 prompt cache 命中。
- N7: 子 Agent 的 `PermissionChecker` 必须独立实例，不能直接共享父引用，`permission_mode` 覆盖时不允许污染父的权限状态。
- N8: `AgentDef` frontmatter 接受的字段集合在解析层完整保留：未来章节（hooks / mcpServers / skills / memory 等）的字段必须在解析层先存得下，避免重复迁移；当前已落地 `name / description / tools / disallowedTools / model / maxTurns / permissionMode / background / isolation`。

## 5. 设计概要

- 核心数据结构：
  - `AgentTool`：承载 `AgentLoader / TaskManager / TraceManager / parent_agent / provider_config / worktree_manager / team_manager / enable_fork` 等运行时依赖。
  - `AgentDef`：Markdown frontmatter 解出来的 dataclass，含 `agent_type / when_to_use / system_prompt / tools / disallowed_tools / model / max_turns / permission_mode / background / isolation / file_path / source`。
  - `AgentToolParams`：pydantic 模型，对应 `Agent` 工具的入参 schema（`prompt / description / subagent_type / model / run_in_background / name / isolation / team_name`）。
  - `TaskManager` / `BackgroundTask` / `ProgressInfo`：后台任务的状态机 + `asyncio.Queue` 通知。
  - `TraceManager` / `TraceNode`：父子 / trace 三元组追踪，token / 状态 / 时间。
  - 工具过滤层：四张 frozenset 控制四层过滤（MCP 豁免 → 全局禁用 → 自定义额外禁用 → 异步白名单 → 定义级黑名单 + 白名单）。
- 主流程：
  - 同步：用户消息 → 主 Agent → LLM 输出 `Agent` 工具调用 → `AgentTool.execute` → 解析 `subagent_type` → 工具过滤 → 创建 `PermissionChecker` → 实例化 `Agent` 子类 → `await sub_agent.run_to_completion(prompt)` → 返回结果。
  - 异步：同上但 `is_background=True` 走 `TaskManager.launch` 启动 `asyncio.Task`，立即返回任务 ID；任务完成时把 ID 写进 `_notify_queue`，主 Agent 在 `_check_completed_tasks` 通过 `poll_completed` 抽出来再用 `inject_task_notifications` 把 `<task-notification>` 注入下一轮 user message。
  - Fork：扫父对话 `FORK_BOILERPLATE_TAG` 拒绝嵌套 → `copy.deepcopy(history)` 复制父对话（保 byte-exact）→ 给悬挂 `tool_use` 补 `"interrupted"` placeholder `ToolResultBlock` → 追加 `FORK_BOILERPLATE + "\n\n你的任务：\n" + task` → 工具池四层过滤 → 始终后台 → 完成走通知。
  - 团队成员：校验 team 存在 → 同 team 内自动 rename `<base>-<n>` → 解析 spec → 创建 worktree → 检测 backend → 用 `build_teammate_tools` 装配（含 `TaskCreate/TaskGet/TaskList/TaskUpdate/SendMessage` 五件套）→ in-process 走 `task_manager.launch` / pane 走 `spawn_tmux_teammate` 或 `spawn_iterm2_teammate`。
- 调用链（模块层级）：
  - `mewcode.app:737-747` 装配 `AgentTool` 并注册进 `registry`；`app:725-728` 实例化 `AgentLoader` 并加载所有 agents；`app:788` 把 catalog 喂给主 Agent。
  - `app:1275-1279` 在主循环里调 `task_manager.poll_completed` + `inject_task_notifications`，把后台完成的子 Agent 结果灌进对话。
  - `app:1029-1031` 在中断路径调 `task_manager.adopt_running` 把当前正在跑的对话挂为后台任务。
  - `app:790 / 794` 注册 `tasks` / `trace` 两个 slash 命令以便用户主动查看后台任务和追踪树。
- 与其他模块的交互：
  - 依赖 `mewcode.agent`（创建子 Agent）、`mewcode.conversation`（forked 对话）、`mewcode.tools`（注册中心 + 过滤）、`mewcode.client`（model 路由）、`mewcode.permissions`（独立 Checker）、`mewcode.worktree`（隔离）、`mewcode.teams`（团队成员）。
  - 被 `mewcode.app` 和 `mewcode.cli` 调用。

## 6. Out of Scope

- 子 Agent 输出全在内存事件流里，不落盘 task 输出文件。
- 不实现 RemoteAgent / DreamTask / LocalWorkflow / MonitorMcp 这些 TaskType。
- 不实现 fork 路径下的 worktree notice 注入（仅 `_execute_with_worktree` 支持）。
- 不接入 plugin / flag / managed 加载源（`register_plugin_source` 仅保留接口，未实装）。
- 不消费 `skills` / `hooks` / `mcpServers` / `memory` / `omitMewcodeMd` 等字段——仅在解析层保留，运行时落地留给后续章节。
- 不实现 `PermissionMode.PLAN` 的复杂裁剪与 bubble。
- 不实现 120s 自动超时切后台 / 持久化后台恢复。
- 不实现 `isolation: remote` 远端运行后端。
- 不内置 Statusline-Setup / Code-Guide 等非核心 Agent。

## 7. 完成定义

见 [checklist.md](checklist.md)，所有条目勾上即完成。
