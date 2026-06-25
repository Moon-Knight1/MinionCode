# ch15: AgentTeam Spec

## 1. 背景

SubAgent（ch13）解决了一次性子任务的上下文隔离，但拓扑是星型：所有子 Agent 只能和主 Agent 通信，子 Agent 之间彼此看不见。当任务规模上来——四个模块同时重构、多角度并行调查 bug、一个 Agent 需要把发现告诉另一个——星型拓扑下主 Agent 成了信息中转瓶颈，子任务被迫串行。这一章把"长期协作团队"做成 MewCode 的一等概念：多个 Agent 组成 Team，并行干活、直接互发消息、共享任务列表和邮箱，主 Agent 可选切换为 Coordinator Mode 专职调度。

## 2. 目标

提供 `AgentTeam` / `TeammateInfo` / `TeamManager` / `Mailbox` / `SharedTaskStore` / `AgentNameRegistry` 一整套数据结构与服务，并暴露 `SendMessageTool` / `TeamCreateTool` / `TeamDeleteTool` 三个工具，让 LLM 在对话里：1) 调 `TeamCreate` 建团队（按环境自动选 tmux / iterm2 / in-process 后端，并在 `~/.mewcode/teams/<name>/` 落盘 config.json + tasks.json + mailbox/），2) 调 `Agent` 工具带 `team_name` 把队员 spawn 进团队（独立 worktree + 受限工具池），3) 队员之间通过 `SendMessage` 走 `Mailbox` 互发消息、按名字或 agent_id 寻址、支持 `to="*"` 广播，4) Lead 每轮迭代开头 `_consume_mailbox` 把收件箱里的消息转 user message 注入对话，5) 启用 `enable_coordinator_mode` 后 Lead 通过 `apply_coordinator_filter` 把工具集收窄到 12 项白名单。Tmux / iTerm2 后端时新 pane 由 `build_cli_command` 拼出 `mewcode -p` 命令字符串，通过 `MEWCODE_TEAM_NAME` / `MEWCODE_TEAMMATE_NAME` / `MEWCODE_MAILBOX_DIR` 环境变量与 Lead 共享同一份 mailbox 目录。

## 3. 功能需求

- F1: `BackendType` 枚举三档 `TMUX="tmux"` / `ITERM2="iterm2"` / `IN_PROCESS="in-process"`；`detect_backend(teammate_mode, is_interactive)` 按 `teammate_mode == "in-process" or not is_interactive` → `TMUX env` → `TERM_PROGRAM == "iTerm.app" + it2 可执行` → `tmux 可执行` 的优先级自动选择；都不命中抛 `BackendDetectionError` 而非静默回退（保证用户不会在不知情下失去进程隔离）。
- F2: `AgentTeam` dataclass 持有 `name / lead_agent_id / members: list[TeammateInfo] / config_path / description`，可 `to_dict` / `from_dict` / `save` / `load`；`get_member(name)` 同时按 `name` 或 `agent_id` 查找；`set_member_active(name, is_active)` 翻转活跃标志；`all_idle()` 返回所有成员是否都为 `is_active is False`。
- F3: `TeammateInfo` dataclass 字段 `name / agent_id / agent_type / model / worktree_path / backend_type / is_active`，`is_active: bool | None = None` 三值语义：`None` 或 `True` 表示活跃，`False` 表示空闲；删除直接从 members 列表移除不留墓碑。
- F4: `TeamManager` 提供 `detect_backend` / `create_team` / `get_team` / `get_task_store` / `get_mailbox` / `register_member` / `set_member_idle` / `register_inprocess_handle` / `register_pane_id` / `get_pane_id` / `delete_team` / `get_team_for_teammate` / `on_teammate_completed` 共 13 个公开方法；内部维护 `_teams` / `_task_stores` / `_mailboxes` / `_inprocess_handles` / `_pane_ids` / `_teammate_team_map` / `_detected_backend` 七个字典/缓存；`_detected_backend` 第一次检测后缓存复用。
- F5: `Mailbox` 基于 `<base_dir>/<agent_id>/<timestamp>_<id>.json` 单文件单消息模型：`write(agent_id, msg)` 落盘 ；`read(agent_id)` 只读不删；`consume(agent_id)` 读完立刻 `f.unlink()`；`broadcast(team_members, msg, exclude)` 按列表逐个 write 排除 exclude；`cleanup(agent_id)` / `cleanup_all()` 清空目录。
- F6: `MailboxMessage` 字段 `id / from_agent / to_agent / content / summary / message_type / timestamp / metadata`；`message_type` 三档 `text / shutdown_request / shutdown_response` 由 `SendMessageTool.VALID_MESSAGE_TYPES` 守门；`text` 类型必须带非空 `summary`（5-10 词）否则报错。
- F7: `create_message(from_agent, to_agent, content, summary, message_type, metadata)` 统一构造器，自动填 `id=uuid4().hex[:12]` 和 `timestamp=time.time()`。
- F8: `SharedTaskStore` 基于单文件 `tasks.json`，结构 `{"next_id": int, "tasks": [...]}`；`create / get / list_tasks / update / init_empty` 五个方法；`SharedTask` 字段 `id / title / description / status / assignee / blocks / blocked_by / created_by`，`status` 四档 `pending / in_progress / completed / blocked`。
- F9: `AgentNameRegistry` 进程内单例（线程安全 double-checked locking）；`register(name, agent_id)` / `resolve(name_or_id)`（先按 name 查再按 id 反查）/ `unregister(name)` / `list_all()` / `reset()` 五个方法。
- F10: `TeamManager.create_team(name, lead_agent_id, description, teammate_mode, is_interactive)` 调 `detect_backend` 决定后端 → `unique_team_name` 自动加 `-2/-3/...` 后缀避免同名 → 在 `~/.mewcode/teams/<slug>/` 建目录 → 写 config.json + tasks.json + mailbox/ → 缓存到 `_teams` / `_task_stores` / `_mailboxes`。
- F11: `TeamManager.delete_team(team_name)` 先校验所有成员都 idle（`is_active is False`），否则报 `Cannot delete team: active members: ...`；通过后遍历每个 member：unregister 名字、cancel in-process handle、kill pane、git worktree remove、trace manager remove；最后 cleanup mailbox + 删团队目录 + 弹出三个缓存字典。
- F12: `spawn_inprocess_teammate(agent, prompt, name, conversation)` 用 `asyncio.create_task` 起协程跑 `agent.run_to_completion`，返 `InProcessTeammateHandle{agent, task, name}`；`handle.done` 判完成、`handle.result` 安全取结果、`handle.cancel()` 取消未完成 task。
- F13: `spawn_tmux_teammate` 三级 fallback：先尝试 `split-window -h -t <team_name>` → 失败则 `new-window` + `split-window` → 再失败则 `new-session -d` + `list-panes`；用 `build_cli_command` 拼出 `MEWCODE_TEAM_NAME=X MEWCODE_TEAMMATE_NAME=Y MEWCODE_MAILBOX_DIR=Z mewcode -p --work-dir <wt> '<prompt>'` 字符串，prompt 内单引号转义为 `'\''`；最后 `send-keys -t <pane> <cmd> Enter` 启动；`kill_pane(pane_id)` best-effort 静默失败。
- F14: `spawn_iterm2_teammate` 复用 `build_cli_command`，通过 `it2 split-pane --command "/bin/zsh -c '<cmd>'"` 创建新 pane 返回 `ITermPaneInfo{session_id}`。
- F15: `save_transcript(team_name, agent_id, conv)` / `load_transcript(team_name, agent_id)` 把 `ConversationManager.history`（含 tool_uses / tool_results 块）序列化为 JSON 落到 `~/.mewcode/teams/<team>/transcripts/<agent_id>.json`，加载时 `env_injected = ltm_injected = True` 防止重复注入。
- F16: `Agent._consume_mailbox(conversation)` 在每轮迭代开头钩入：仅当 `self.team_name and self._team_manager` 非空时取 mailbox.consume 自己的 agent_id；每条消息前缀 `[Message from <sender>] ` 或 `[<message_type> from <sender>] ` 后 `conversation.add_user_message`；异常吞掉记 debug。
- F17: `TeamCreateTool` 暴露 `team_name` 必填 + `description` 可选；调 `detect_backend` 不通过返 IsError；通过后 `team_manager.create_team`；如 `is_coordinator_mode(enable_coordinator_mode)` 返 true 则把 `parent_agent.coordinator_mode = True`、备份 `_full_registry`、把 `parent_agent.registry = apply_coordinator_filter(registry)`，输出附带 "Coordinator Mode activated" 提示。
- F18: `TeamDeleteTool` 暴露 `team_name` 必填；调 `team_manager.delete_team` 捕获 `TeamError` 返 IsError；如 `parent_agent.coordinator_mode` 为 true 则恢复 `_full_registry` 并清零 flag，输出附带 "Coordinator Mode deactivated" 提示。
- F19: `SendMessageTool` 暴露 `to / message / summary / message_type / metadata`；先校验 `message_type in VALID_MESSAGE_TYPES`，再校验 `text` 类型必须有 `summary`；`to == "*"` 走 `mailbox.broadcast(member_ids ∪ {lead_agent_id} \ {self})`，否则用 `AgentNameRegistry.instance().resolve(to)` 解析目标 id；写完后 `_wake_pane(target_id)` 向 tmux pane send-keys 空行触发新消息读取（pane 后端唤醒机制）。
- F20: `AgentTool._execute_as_teammate(p)` 处理 `team_name != None` 分支：校验 team 存在、按 base_name 同名冲突自动加 `-2/-3/...`、可选解析 `subagent_type` 否则 fork、`worktree_manager.create(f"team-{team_name}/{teammate_name}", "HEAD")` 建独立 worktree、`build_teammate_tools` 按 backend 类型构造队员工具池（in-process 严格白名单 / pane 模式只剔除 `TeamCreate` 和 `TeamDelete`）、`register_member` 注册到团队 + AgentNameRegistry、按 backend 分发 `spawn_inprocess_teammate` 或 `_spawn_pane_teammate`。

## 4. 非功能需求

- N1: `Mailbox` 单文件单消息模型避免跨进程并发写覆盖：每条消息文件名 `<timestamp>_<id>.json` 全局唯一，写入无需文件锁；`consume` 按 `sorted(d.iterdir())` 时间排序保证 FIFO；`unlink` 单文件操作在 POSIX 文件系统上原子，不会丢消息。
- N2: `detect_backend` 检测失败不静默回退到 in-process——直接抛 `BackendDetectionError` 让用户显式选择：要么装 tmux / iTerm2+it2，要么在 config.yaml 设 `teammate_mode: "in-process"`。理由是 pane 后端提供的进程隔离是团队模式的核心保障，静默降级会让用户失去隔离能力还不自知。
- N3: `AgentNameRegistry` 是进程内单例，因此跨进程的 pane teammate 必须自己在子进程内重新注册名字 → agent_id 映射，不能依赖 Lead 进程的注册表；`resolve` 同时支持按 name 和按 agent_id 反查，给 Lead 端 SendMessage 用名字、给子进程端用 agent_id 都能命中。
- N4: `TeamManager._detected_backend` 一旦检测过就缓存，整个 team manager 生命周期内不变。同进程内多次 `create_team` 不会重新探测环境——保证一致性，避免中途装 tmux 导致前后行为不一致。
- N5: `_consume_mailbox` 必须放在 Agent 每轮迭代开头（在调 LLM 之前），不能放在迭代结束：放结束会让"工具调用完成 → idle → 下轮才看到新消息"出现一轮延迟；放开头保证 LLM 看到的对话历史里已经包含队员的最新消息，决策不滞后。
- N6: `TeamCreateTool` 启用 Coordinator Mode 时必须把原 `registry` 备份到 `parent_agent._full_registry`，`TeamDeleteTool` 恢复时从这里读回——不能依赖重新构造，因为 registry 里可能已经注入了运行时动态注册的工具（MCP / Skill）。
- N7: `SendMessageTool` 的 `_wake_pane` 在 pane teammate 场景必须 send-keys 触发新消息读取（pane 进程在 `mewcode -p` 单次执行模式下会阻塞在 stdin），否则消息只是写入 mailbox 但对方进程感知不到；in-process teammate 不需要 wake 因为同进程 `_consume_mailbox` 每轮自动跑。
- N8: `build_cli_command` 把 `prompt` 内的单引号转义为 `'\''`（关闭→插入字面单引号→重开），否则 prompt 里出现单引号会破坏 shell 解析；前缀环境变量 `MEWCODE_TEAM_NAME` / `MEWCODE_TEAMMATE_NAME` 通过空格分隔但不加引号，假设值是合法标识符。
- N9: `delete_team` 必须先校验所有 member `is_active is False`，活跃成员存在时拒绝删除——避免运行中的 in-process 协程或 pane 进程突然失去 mailbox 后悬挂。Active 检查用 `is_active is not False`（`None` 和 `True` 都算 active）。
- N10: 测试运行 `Mailbox` 和 `AgentTeam.save` 必须 `monkeypatch / patch("mewcode.teams.models.Path.home", ...)` 重定向 home 到 `tmp_path`，否则跑完测试会在用户主目录残留 `~/.mewcode/teams/` 目录污染。
- N11: `AgentNameRegistry.reset()` 在 pytest fixture 中 autouse 调用——单例在用例间共享会让 register 状态泄漏，导致 `test_register_and_resolve` 后跑的用例看到上个用例的残留映射。
- N12: `spawn_iterm2_teammate` 通过外部 `it2` CLI 而非直接 osascript——it2 是 iTerm2 官方提供的稳定 CLI，比 osascript 字符串拼接的 AppleScript 更可靠且支持版本演进。

## 5. 设计概要

- 核心数据结构:
  - `AgentTeam`：团队聚合 dataclass，`members: list[TeammateInfo]` 列表非 map（队员数不大，遍历足够），通过 `config_path` 持久化到 `~/.mewcode/teams/<slug>/config.json`。
  - `TeammateInfo`：队员元信息 dataclass，`is_active: bool | None` 三值语义；`agent_id` 是全局唯一进程标识；`worktree_path` 关联到 ch14 的 worktree。
  - `TeamManager`：全局团队注册表 + 多类资源缓存（mailbox / task store / inprocess handle / pane id / teammate→team 反查映射），是 Lead 进程的"团队服务总线"。
  - `Mailbox` + `MailboxMessage`：单文件单消息模型，靠时间戳前缀文件名保证 FIFO 且跨进程写入无冲突；支持 `text / shutdown_request / shutdown_response` 三种类型。
  - `SharedTaskStore` + `SharedTask`：JSON 文件实现的共享任务列表，团队内所有成员通过 `team_manager.get_task_store(team_name)` 读到同一份。
  - `AgentNameRegistry`：进程内单例（线程安全 double-checked），把人类可读的 name 映射到 agent_id，给 SendMessage 寻址用。
  - `InProcessTeammateHandle`：包装 `asyncio.Task`，供 `TeamManager` 跟踪 in-process 队员生命周期。
- 主流程（按生命周期）:
  - 创建：用户消息 → 主 Agent → LLM 调 `TeamCreate(team_name="X")` → `team_manager.detect_backend()` 选模式 → `team_manager.create_team` 在 `~/.mewcode/teams/x/` 落 config.json + tasks.json + mailbox/ → 可选切 Coordinator Mode（备份 `_full_registry` + `apply_coordinator_filter`）。
  - Spawn 队员：Lead LLM 调 `Agent(team_name="X", name="alice", prompt="...")` → `AgentTool.execute` 看到 `team_name` 非空走 `_execute_as_teammate` → 校验团队、解析子 agent type / fork、`worktree_manager.create` 建独立 wt、按 backend 调 `build_teammate_tools` 构造工具池、`register_member` 注册到团队 + `AgentNameRegistry.instance().register`。
  - In-process：`spawn_inprocess_teammate(agent, prompt, name)` 起 `asyncio.create_task` 跑 `agent.run_to_completion`，handle 注册到 `team_manager._inprocess_handles`。
  - Pane 后端：`spawn_tmux_teammate` / `spawn_iterm2_teammate` 用 `build_cli_command` 拼出带 `MEWCODE_TEAM_NAME` / `MEWCODE_TEAMMATE_NAME` / `MEWCODE_MAILBOX_DIR` env 的 `mewcode -p` 命令字符串 → tmux send-keys / it2 split-pane 启动 → pane_id 注册到 `team_manager._pane_ids`。
  - 通信：队员调 `SendMessage(to="bob", message="...", summary="...")` → `AgentNameRegistry.resolve("bob")` → target_id → `mailbox.write(target_id, msg)` → `_wake_pane(target_id)`（pane 队员需要）→ 对方下一轮 `_consume_mailbox` 拿到。
  - Lead 感知：每轮 Lead `agent.run_to_completion` 内部 `_consume_mailbox(conversation)` 把 `mailbox.consume(self.agent_id)` 的所有消息转 user message 注入 conversation，前缀 `[Message from X] ` 或 `[shutdown_request from X] `。
  - Idle 通知：`AgentTool` 后台任务完成回调 `team_manager.on_teammate_completed(agent_id)` → `set_member_idle` 把 `is_active=False` + 写一条 `"Teammate 'X' is now idle"` 到 Lead 邮箱。
  - Coordinator Mode：`apply_coordinator_filter(registry)` 把工具集筛到 `COORDINATOR_MODE_ALLOWED_TOOLS` 12 项 `{Agent, SendMessage, TaskCreate, TaskGet, TaskList, TaskUpdate, TeamCreate, TeamDelete, ReadFile, Glob, Grep, Bash}`（写工具 `WriteFile / EditFile` 被排除）；`TeamDeleteTool` 恢复 `_full_registry`。
  - Stop：`TeamDelete(team_name="X")` → `team_manager.delete_team` → 校验全员 idle → 遍历每个 member：unregister name、cancel handle、kill pane、git worktree remove、trace_manager.remove → cleanup mailbox + 删团队目录 + 弹出三个缓存。
- 调用链（模块层级）:
  - `mewcode/app.py:730-762` 在 `MewCodeApp.__init__` 后段创建 `TeamManager(worktree_manager, trace_manager)`，把 `team_manager` 注入 `AgentTool`，把 `TeamCreateTool / TeamDeleteTool / SyntheticOutputTool` 注册进 registry，把 `agent._team_manager = team_manager` 写回主 Agent。
  - `mewcode/agent.py:324-326` 主 Agent `__init__` 声明 `self.coordinator_mode / self.team_name / self._team_manager` 三个字段；`:433 / :957` 在 `run_to_completion` 主循环开头调 `self._consume_mailbox(conversation)`；`:471 / :937` 给 `build_system_prompt` 传 `coordinator_mode=self.coordinator_mode` 切提示词。
  - `mewcode/tools/agent_tool.py:86-87` `execute` 入口看到 `p.team_name` 非空时优先走 `_execute_as_teammate`（先于 fork / sync / async 分发）；`:246-414` 实现完整队员 spawn 流程。
- 与其他模块的交互:
  - 依赖 `mewcode/agent`（Agent 实例 / `run_to_completion` / 系统提示注入）、`mewcode/conversation`（ConversationManager / Message / ToolUseBlock / ToolResultBlock）、`mewcode/agents/tool_filter`（`apply_coordinator_filter` / `build_teammate_tools` / `COORDINATOR_MODE_ALLOWED_TOOLS` / `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`）、`mewcode/worktree`（每个队员独立 worktree）、`mewcode/tools/base`（Tool / ToolResult / ToolRegistry）。
  - 被 `mewcode/app.py`（注册三件套工具 + 写回 `agent._team_manager`）、`mewcode/tools/agent_tool.py`（`_execute_as_teammate` 调 spawn / register）、`mewcode/prompts.py`（`build_system_prompt(coordinator_mode=...)` 切系统提示词）调用。

## 6. Out of Scope

- 不实现 PR 文档里描述的 `planModeRequired` 字段和审批工作流——`TeammateInfo` 只保留基础元信息，审批门槛由后续章节扩展。
- 不实现 `shutdown_response` 完整双向握手协议——只保留 `message_type` 字段的三档枚举，握手语义由 LLM 在文本层约定。
- 不实现共享任务依赖图的拓扑排序自动调度——`SharedTask.blocks / blocked_by` 字段已存但 store 仅做 CRUD，依赖推断由 Lead LLM 从任务列表文本自己读出。
- 不实现"队员后从磁盘恢复对话续写"机制——in-process 队员 task 完成或 cancel 后即终止，transcript 落盘仅供事后回看，不支持 resume；要 Lead 想再用需要重新 spawn。
- 不实现"协调模式四阶段工作流"强制约束（Research / Synthesis / Implementation / Verification）——`get_coordinator_system_prompt` 写入提示词层引导，但工具层不强制顺序。
- 不实现 `MEWCODE_COORDINATOR_MODE` 自动激活——必须 `enable_coordinator_mode=True` 配合 env var 双开关同时打开才生效，避免 Lead 进程被意外切到协调模式。
- 不实现 mailbox 的跨节点分布式同步——团队只在单机内运作，所有 mailbox 文件在本地 `~/.mewcode/teams/<name>/mailbox/` 下；要跨机协作需要外部传输层。
- 不实现 worker pane 进程的自动重启——pane 进程 crash 后 Lead 端 `pane_id` 仍记录但实际 pane 已死；用户需手动 `TeamDelete` 然后重建。

## 7. 完成定义

见 [checklist.md](checklist.md)，所有条目勾上即完成。
