# ch15: AgentTeam Tasks

> 任务粒度：每个任务可在一次会话内完成，可独立交付。

## T1: 定义 BackendType / TeammateInfo / AgentTeam 三个核心模型

- 影响文件: `mewcode/teams/models.py`（`BackendType` @ 10-13；`TeammateInfo` @ 16-31；`_sanitize_name` @ 34-37；`AgentTeam` @ 40-102；`resolve_team_dir` @ 105-107；`unique_team_name` @ 110-117）
- 依赖任务: 无
- 完成标准: `BackendType(str, Enum)` 三档常量 `TMUX / ITERM2 / IN_PROCESS`；`TeammateInfo` dataclass 7 字段含 `is_active: bool | None = None` 三值；`AgentTeam` 含 `get_member` / `add_member` / `remove_member` / `set_member_active` / `all_idle` / `active_members` / `to_dict` / `from_dict` / `save` / `load`；`get_member` 按 name 或 agent_id 双向查找；`resolve_team_dir` 落到 `~/.mewcode/teams/<slug>/`；`unique_team_name` 同名冲突自动加 `-2/-3/...` 后缀。

- [ ] 完成

## T2: 实现 Mailbox + MailboxMessage（单文件单消息）

- 影响文件: `mewcode/teams/mailbox.py`（`MailboxMessage` @ 11-27；`Mailbox` @ 30-102；`create_message` @ 105-122）
- 依赖任务: 无
- 完成标准: `MailboxMessage` 8 字段含 `id / from_agent / to_agent / content / summary / message_type / timestamp / metadata`；`message_type` 注释三档 `text | shutdown_request | shutdown_response`；`Mailbox.write` 以 `<timestamp>_<id>.json` 为文件名落到 `<base>/<agent_id>/` 目录；`read` 只读不删按 `sorted(d.iterdir())` 时间序；`consume` 读完立刻 `f.unlink()` 保证 FIFO；`broadcast(team_members, msg, exclude)` 逐个 write 排除 exclude；`cleanup` / `cleanup_all` 清目录；`create_message` 自动填 `uuid4().hex[:12]` 和 `time.time()`。

- [ ] 完成

## T3: 实现 detect_backend 优先级链

- 影响文件: `mewcode/teams/backend_detect.py`（`BackendDetectionError` @ 9-10；`_in_tmux_session` @ 13-14；`_in_iterm2` @ 17-18；`_it2_available` @ 21-22；`_tmux_installed` @ 25-26；`detect_backend` @ 29-51）
- 依赖任务: T1
- 完成标准: 优先级 `teammate_mode == "in-process" or not is_interactive` → `TMUX` env → `TERM_PROGRAM == "iTerm.app" + shutil.which("it2")` → `shutil.which("tmux")`；都不命中抛 `BackendDetectionError` 而非静默回退；错误消息含 `tmux: brew install tmux` 和 `iTerm2 + it2 CLI` 安装指引并提示 `teammate_mode: "in-process"` 选项。

- [ ] 完成

## T4: 实现 SharedTaskStore + SharedTask

- 影响文件: `mewcode/teams/shared_task.py`（`SharedTask` @ 9-25；`SharedTaskStore` @ 28-；`__init__` + `_load` + `_save`；`create` / `get` / `list_tasks` / `update` / `init_empty`）
- 依赖任务: 无
- 完成标准: `SharedTask` dataclass 8 字段含 `id / title / description / status / assignee / blocks / blocked_by / created_by`；`status` 注释四档 `pending | in_progress | completed | blocked`；`SharedTaskStore` 用单文件 `tasks.json` 结构 `{"next_id": int, "tasks": [...]}`；`create` 自增 id 返 `SharedTask` 实例；`list_tasks(status=None, assignee=None)` 双过滤；`update` 部分字段更新 + `add_blocks` / `add_blocked_by` 列表追加（去重）；`init_empty` 清空 + 重置 `_next_id=1` + save。

- [ ] 完成

## T5: 实现 AgentNameRegistry 单例

- 影响文件: `mewcode/teams/registry.py`（`AgentNameRegistry` @ 6-40）
- 依赖任务: 无
- 完成标准: 进程内单例（线程安全 double-checked locking with `_lock = threading.Lock()`）；`instance()` / `reset()` 类方法；`register(name, agent_id)` / `resolve(name_or_id)` 同时支持按 name 和按 id 反查 / `unregister(name)` / `list_all()` 实例方法；`_names: dict[str, str]` 内部存储 name → agent_id 映射。

- [ ] 完成

## T6: 实现 spawn_inprocess_teammate + InProcessTeammateHandle

- 影响文件: `mewcode/teams/spawn_inprocess.py`（`InProcessTeammateHandle` @ 14-40；`spawn_inprocess_teammate` @ 43-56）
- 依赖任务: T1
- 完成标准: `InProcessTeammateHandle(agent, task, name)` 属性 `done` / `result`（已完成时安全取结果异常返 None）/ `cancel()` 取消未完成 task；`spawn_inprocess_teammate(agent, prompt, name, conversation=None)` 用 `asyncio.create_task` 起协程跑 `agent.run_to_completion(prompt)` 或 `agent.run_to_completion("", conversation)`（传 conversation 走 fork 路径）；task name 设为 `f"teammate-{name}"`。

- [ ] 完成

## T7: 实现 spawn_tmux_teammate + build_cli_command + kill_pane

- 影响文件: `mewcode/teams/spawn_tmux.py`（`TmuxPaneInfo` @ 10-13；`TmuxSpawnError` @ 16-17；`_run_tmux` @ 20-29；`build_cli_command` @ 32-56；`spawn_tmux_teammate` @ 59-108；`send_keys_to_pane` @ 111-115；`kill_pane` @ 118-122）
- 依赖任务: T1
- 完成标准: `build_cli_command` 拼出 `MEWCODE_TEAM_NAME=X MEWCODE_TEAMMATE_NAME=Y MEWCODE_MAILBOX_DIR=Z mewcode -p --work-dir <wt> [--agent-type X] [--model X] '<prompt>'`，prompt 内单引号转义为 `'\''`；`spawn_tmux_teammate` 三级 fallback——先 `split-window -h -t <team_name>` → 失败则 `new-window` + `split-window` → 再失败则 `new-session -d` + `list-panes` 取第一个；最后 `send-keys -t <pane> <cmd> Enter`；`kill_pane` best-effort 静默失败；`send_keys_to_pane` 用于 wake pane。

- [ ] 完成

## T8: 实现 spawn_iterm2_teammate

- 影响文件: `mewcode/teams/spawn_iterm2.py`（`ITermPaneInfo` @ 10-12；`ITermSpawnError` @ 15-16；`_run_it2` @ 19-28；`spawn_iterm2_teammate` @ 31-58）
- 依赖任务: T7
- 完成标准: 复用 `build_cli_command` 拼命令；通过 `it2 split-pane --command "/bin/zsh -c '<cmd>'"` 创建新 pane；返回 `ITermPaneInfo{session_id}`；spawn 失败抛 `ITermSpawnError`（不静默吞）；使用外部 `it2` CLI 而非 osascript 字符串拼接。

- [ ] 完成

## T9: 实现 transcript 持久化

- 影响文件: `mewcode/teams/transcript.py`（`_serialize_conversation` @ 10-33；`_deserialize_conversation` @ 36-64；`save_transcript` @ 67-79；`load_transcript` @ 82-92）
- 依赖任务: T1
- 完成标准: `save_transcript(team_name, agent_id, conv)` 把 `conv.history`（含 tool_uses / tool_results 块）序列化 JSON 落到 `<team_dir>/transcripts/<agent_id>.json`；`load_transcript` 反序列化时 `env_injected = ltm_injected = True` 防止重复注入环境消息；tool_uses 用 `ToolUseBlock{tool_use_id, tool_name, arguments}` 结构，tool_results 用 `ToolResultBlock{tool_use_id, content, is_error}`。

- [ ] 完成

## T10: 实现 TeamManager 全套方法

- 影响文件: `mewcode/teams/manager.py`（`TeamError` @ 27-28；`TeamManager.__init__` @ 31-41；`detect_backend` @ 43-50；`create_team` @ 52-86；`get_team` @ 88-97；`get_task_store` @ 99-108；`get_mailbox` @ 110-119；`register_member` @ 121-134；`set_member_idle` @ 136-152；`register_inprocess_handle` @ 154-155；`register_pane_id` @ 157-158；`get_pane_id` @ 160-161；`delete_team` @ 163-201；`get_team_for_teammate` @ 203-210；`on_teammate_completed` @ 212-221；`_kill_pane` @ 223-229；`_cleanup_worktree` @ 231-245；`_remove_dir` @ 247-）
- 依赖任务: T1, T2, T3, T4, T5, T6
- 完成标准: `__init__` 七字段 `_teams / _task_stores / _mailboxes / _inprocess_handles / _pane_ids / _detected_backend / _teammate_team_map` 全初始化空 dict / None；`detect_backend` 第一次后缓存到 `_detected_backend`；`create_team` 链 `detect_backend → unique_team_name → mkdir → AgentTeam(...).save → SharedTaskStore.init_empty → Mailbox` 并缓存三个字典；`register_member` 同时 `AgentNameRegistry.register` + 写 `_teammate_team_map`；`set_member_idle` 翻 is_active 并写 idle 通知到 Lead 邮箱；`delete_team` 先校验全员 `is_active is not False` 必须 idle，否则抛 `TeamError`，通过后遍历清 name registry / handle.cancel / `_kill_pane` / git worktree remove / trace_manager.remove，最后 `mailbox.cleanup_all` + 删目录 + 弹三个缓存。

- [ ] 完成

## T11: 实现 Agent._consume_mailbox 接入

- 影响文件: `mewcode/agent.py`（`self.coordinator_mode / self.team_name / self._team_manager` 字段 @ 324-326；`_consume_mailbox` @ 718-733；`run_to_completion` 主循环钩入 @ 433 + @ 957；`coordinator_mode` 传 `build_system_prompt` @ 471 + @ 937）
- 依赖任务: T2, T10
- 完成标准: `Agent.__init__` 加 `self.coordinator_mode: bool = False / self.team_name: str = "" / self._team_manager: Any = None` 三字段；`_consume_mailbox(conversation)` 在 `team_name` 和 `_team_manager` 都非空时取 `team_manager.get_mailbox(team_name).consume(self.agent_id)`；每条消息前缀 `[Message from <sender>] ` 或 `[<message_type> from <sender>] ` 后 `conversation.add_user_message`；异常吞掉记 `log.debug`；在 `run_to_completion` 主循环开头（每轮迭代前）和 `iterate_once` 开头都调一次。

- [ ] 完成

## T12: 实现 coordinator 系统提示词 + 工具过滤

- 影响文件: `mewcode/teams/coordinator.py`（`is_coordinator_mode` @ 7-11；`match_session_mode` @ 14-36；`get_coordinator_system_prompt` @ 39-；`get_coordinator_user_context`）；`mewcode/agents/tool_filter.py`（`COORDINATOR_MODE_ALLOWED_TOOLS` 12 项 @ 66-79；`TEAMMATE_COORDINATION_TOOLS` 5 项 @ 50-56；`IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` @ 58-64；`apply_coordinator_filter` @ 187-193；`build_teammate_tools` @ 129-184）
- 依赖任务: T5, T10
- 完成标准: `is_coordinator_mode(enable_flag)` 双锁判定（flag false 直接 false；flag true 时读 `MEWCODE_COORDINATOR_MODE` env 三档 `1/true/yes`）；`match_session_mode` 实现恢复会话时的 env var 同步；`get_coordinator_system_prompt` 输出含 `Research / Synthesis / Implementation / Verification` 四阶段、`<task-notification>` XML 格式、`based on your findings` anti-pattern；`COORDINATOR_MODE_ALLOWED_TOOLS = {Agent, SendMessage, TaskCreate, TaskGet, TaskList, TaskUpdate, TeamCreate, TeamDelete, ReadFile, Glob, Grep, Bash}` 12 项（写工具 `WriteFile / EditFile` 被排除）；`apply_coordinator_filter(registry)` 把 registry 筛到白名单；`build_teammate_tools` 按 backend 类型分流：in-process 严格白名单 `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`，pane 模式只剔除 `TeamCreate` 和 `TeamDelete`。

- [ ] 完成

## T13: 实现 SendMessageTool / TeamCreateTool / TeamDeleteTool 三个工具

- 影响文件: `mewcode/tools/send_message.py`（`SendMessageParams` @ 16-21；`VALID_MESSAGE_TYPES` @ 24；`SendMessageTool` @ 27-；`execute` @ 51-109；`_wake_pane` @ 111-119；`_wake_pane_members` @ 121-123）；`mewcode/tools/team_create.py`（`TeamCreateParams` @ 14-16；`TeamCreateTool` @ 19-85）；`mewcode/tools/team_delete.py`（`TeamDeleteParams` @ 14-15；`TeamDeleteTool` @ 18-53）
- 依赖任务: T2, T5, T10, T12
- 完成标准:
  - `SendMessageTool.execute`：先校验 `message_type in VALID_MESSAGE_TYPES`，`text` 类型必须有 `summary`；`to == "*"` 走 broadcast（member_ids 不含 self，添加 lead_agent_id 如果 self 不是 lead）；否则 `AgentNameRegistry.instance().resolve(to)` 解析后 `mailbox.write`；写完调 `_wake_pane(target_id)` 唤醒 pane 后端；非法 to 返 IsError `Cannot resolve recipient '...'`。
  - `TeamCreateTool.execute`：先 `team_manager.detect_backend` 不通过返 IsError；通过后 `team_manager.create_team`；如 `is_coordinator_mode(enable_coordinator_mode)` 返 true 则 `parent_agent.coordinator_mode = True`、`parent_agent._full_registry = parent_agent.registry`、`parent_agent.registry = apply_coordinator_filter(registry)`，输出附 "Coordinator Mode activated" 提示。
  - `TeamDeleteTool.execute`：调 `team_manager.delete_team` 捕获 `TeamError` 返 IsError；如 `parent_agent.coordinator_mode` 为 true 则 `parent_agent.registry = parent_agent._full_registry` 恢复并清零 flag，输出附 "Coordinator Mode deactivated" 提示。

- [ ] 完成

## T14: 实现 AgentTool._execute_as_teammate（team_name 分支）

- 影响文件: `mewcode/tools/agent_tool.py`（`AgentToolParams.team_name` 字段 @ 28；`TEAMMATE_ADDENDUM` 常量 @ 38；`AgentTool.__init__` 加 `team_manager` 参数 @ 72；`_team_manager` 字段 @ 81；`execute` 入口分支 @ 86-87；`_execute_as_teammate` @ 246-414）
- 依赖任务: T10, T12, T13（ch13/ch14 的 AgentTool / WorktreeManager）
- 完成标准:
  - `AgentToolParams` 加 `team_name: str | None = None` 字段；
  - `AgentTool.__init__` 加 `team_manager` 关键字参数和 `_team_manager` 实例字段；
  - `execute` 入口看到 `p.team_name` 非空时优先走 `_execute_as_teammate`（先于 fork / sync / async 分发）；
  - `_execute_as_teammate`：校验 team_manager / worktree_manager 配置、`team_manager.get_team(team_name)` 不存在返 IsError；base_name 同名冲突自动加 `-2/-3/...`；可选解析 `subagent_type`，无 type + enable_fork 走 `build_forked_messages` 否则用空白 builtin AgentDef；`worktree_manager.create(f"team-{team_name}/{teammate_name}", "HEAD")` 建独立 wt；`detect_backend` 决定后端；`build_teammate_tools` 按 backend 构造工具池；用 `AgentClass(agent_id, registry, ...)` 创建 sub-agent 注入 `TEAMMATE_ADDENDUM`；`AgentNameRegistry.instance().register(teammate_name, agent_id)`；构造 `TeammateInfo` 后 `team_manager.register_member`；按 backend 分发 in-process（`spawn_inprocess_teammate`）或 pane（`_spawn_pane_teammate`）。

- [ ] 完成

## T15: app.py 注册三件套 + 注入 team_manager

- 影响文件: `mewcode/app.py`（`MewCodeApp.__init__` 加 `teammate_mode / enable_coordinator_mode` 参数 @ 519-520；`_teammate_mode / _enable_coordinator_mode` 字段 @ 530-531；team 系统设置块 @ 730-762；`agent._team_manager = team_manager` 注入 @ 801；`on_teammate_completed` 回调 @ 1287-1288；shutdown 清理 @ 1592-1598）；`mewcode/__main__.py`（`teammate_mode / enable_coordinator_mode` 透传 `MewCodeApp` @ 57-58）；`mewcode/config.py`（`AppConfig.teammate_mode` / `enable_coordinator_mode` 字段 + load_config 校验 `teammate_mode in {"", "in-process"}`）
- 依赖任务: T11, T13, T14
- 完成标准:
  1. `MewCodeApp.__init__` 加 `teammate_mode: str = ""` 和 `enable_coordinator_mode: bool = False` 参数；
  2. 在 AgentTool 注册之前 `self.team_manager = TeamManager(worktree_manager, trace_manager)`；
  3. AgentTool 构造时传 `team_manager=self.team_manager`；
  4. 注册 `TeamCreateTool(team_manager, parent_agent, teammate_mode, is_interactive=True, enable_coordinator_mode)`；
  5. 注册 `TeamDeleteTool(team_manager, parent_agent)`；
  6. 注册 `SyntheticOutputTool()`；
  7. `self.agent._team_manager = self.team_manager` 写回主 Agent；
  8. 后台 task 完成回调里调 `self.team_manager.on_teammate_completed(task.agent.agent_id)`；
  9. shutdown 时遍历所有团队强制 `set_member_active(False)` 后 `delete_team` 释放资源；
  10. `mewcode/__main__.py main()` 把 `config.teammate_mode` / `config.enable_coordinator_mode` 透传 `MewCodeApp`；
  11. `config.py` 加两个字段及 `teammate_mode` 校验（合法值仅 `""` 和 `"in-process"`）。

- [ ] 完成

## T16: 端到端验证

- 影响文件: 无（仅运行验证）
- 依赖任务: T1-T15
- 完成标准:
  - `ruff check mewcode/teams mewcode/tools/team_create.py mewcode/tools/team_delete.py mewcode/tools/send_message.py` 通过；
  - `pytest tests/test_teams.py -v` 通过（覆盖 10 大类 30+ 用例：TestModels 7 + TestSharedTaskStore 6 + TestMailbox 5 + TestAgentNameRegistry 4 + TestBackendDetect 6 + TestToolFilter 3 + TestCoordinatorMode 11 + TestConfigExtensions 3 + TestTranscript 2 + TestAgentCoordinatorIntegration 3）；
  - `pytest tests/test_subagent.py -v` 仍全部通过（确保 AgentTool 改造未破坏 ch13 功能）；
  - 主流程接线验证：`grep -n "TeamManager\|TeamCreateTool\|TeamDeleteTool\|team_manager" mewcode/app.py` 命中至少 8 处；`grep -n "_consume_mailbox\|_team_manager\|coordinator_mode" mewcode/agent.py` 看到主 Agent 三处接入；`grep -n "_execute_as_teammate" mewcode/tools/agent_tool.py` 命中入口分发 + 函数体。

- [ ] 完成

## 进度

- [ ] T1 / [ ] T2 / [ ] T3 / [ ] T4 / [ ] T5 / [ ] T6 / [ ] T7 / [ ] T8 / [ ] T9 / [ ] T10 / [ ] T11 / [ ] T12 / [ ] T13 / [ ] T14 / [ ] T15 / [ ] T16
