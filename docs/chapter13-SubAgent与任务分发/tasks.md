# ch13: SubAgent Tasks

> 任务粒度：每个任务可在一次会话内完成，可独立交付。

## T1: 定义 `AgentDef` dataclass + 三档 builtin Markdown

- 影响文件: `mewcode/agents/parser.py`（`AgentDef` @ 23-35），`mewcode/agents/builtins/general-purpose.md` / `plan.md` / `explore.md`
- 依赖任务: 无
- 完成标准: `AgentDef` 含 12 个字段（含 `agent_type / when_to_use / system_prompt / tools / disallowed_tools / model / max_turns / permission_mode / background / isolation / file_path / source`），默认 `model="inherit" / max_turns=50 / permission_mode="default"`；`Plan` builtin 设 `disallowedTools: [Agent, EditFile, WriteFile, NotebookEdit]` + `maxTurns: 15`；`Explore` builtin 设 `model: haiku` + `maxTurns: 30`。

- [ ] 完成

## T2: 实现 `parse_frontmatter` + `parse_agent_file` + 校验

- 影响文件: `mewcode/agents/parser.py`（`parse_frontmatter` @ 38-58，`_validate_agent_meta` @ 61-94，`parse_agent_file` @ 97-119）
- 依赖任务: T1
- 完成标准: 解析 `---\nyaml\n---\nbody`；缺 `name` / `description` 抛 `AgentParseError`；非法 `model`（非 `inherit / haiku / sonnet / opus / ""`）抛错；非法 `permissionMode`（非 `default / acceptEdits / dontAsk / ""`）抛错；非法 `isolation`（非 `worktree / ""`）抛错；非正 `maxTurns` 抛错；YAML 解析失败抛错。

- [ ] 完成

## T3: 实现 `AgentLoader`，按 project → user → builtin 优先级加载

- 影响文件: `mewcode/agents/loader.py`（`AgentLoader` @ 15-22，`_scan_directory` @ 24-39，`_load_builtins` @ 41-83，`load_all` @ 85-107，`get` @ 109-126，`list_agents` @ 128-131）
- 依赖任务: T2
- 完成标准: `load_all` 顺序 = 项目级 `<work_dir>/.mewcode/agents/*.md`（`source="project"`）→ 用户级 `~/.mewcode/agents/*.md`（`source="user"`）→ builtin（`importlib.resources` 读 `mewcode/agents/builtins`）；同名先注册者胜出（项目级覆盖 builtin）；`enable_verification=False` 时 `Verification` 不加入；`get` 支持热重载（`file_path` 存在时重新解析）；bad file 通过 try/except + log.warning 跳过。

- [ ] 完成

## T4: 实现四层工具过滤 `resolve_agent_tools`

- 影响文件: `mewcode/agents/tool_filter.py`（`ALL_AGENT_DISALLOWED_TOOLS` @ 12-20，`CUSTOM_AGENT_DISALLOWED_TOOLS` @ 22-30，`ASYNC_AGENT_ALLOWED_TOOLS` @ 32-49，`_is_mcp_tool` @ 79-80，`resolve_agent_tools` @ 83-126）
- 依赖任务: 无
- 完成标准: `ALL_AGENT_DISALLOWED_TOOLS` 含 `TaskOutput / ExitPlanMode / EnterPlanMode / Agent / AskUserQuestion / TaskStop / Workflow` 七项；MCP 工具（`mcp__` 前缀）一律放行；`source ∈ {project, user, plugin}` 触发 custom layer；`is_background=True` 时只保留 `ASYNC_AGENT_ALLOWED_TOOLS` 白名单；definition 级 `disallowed_tools` / `tools` 生效。

- [ ] 完成（测试覆盖 `tests/test_subagent.py::TestToolFilter` 六个用例）

## T5: 实现 `Fork` 模式（`build_forked_messages` + `ForkError`）

- 影响文件: `mewcode/agents/fork.py`（`FORK_BOILERPLATE_TAG` @ 7，`FORK_BOILERPLATE` @ 9-23，`ForkError` @ 26-27，`build_forked_messages` @ 30-79）
- 依赖任务: 无
- 完成标准: 检测父对话历史里任意 `msg.content` 含 `FORK_BOILERPLATE_TAG` 即 `raise ForkError`；`copy.deepcopy(conversation.history)` 复制对话保 byte-exact；最后一条 assistant 消息有未完成 `tool_uses` 时补 `"interrupted"` placeholder `ToolResultBlock`；末尾 `add_user_message(f"{FORK_BOILERPLATE}\n\n你的任务：\n{task}")`。

- [ ] 完成

## T6: 实现 `TraceManager` 调用树追踪

- 影响文件: `mewcode/agents/trace.py`（`TraceNode` @ 8-17，`TraceManager` @ 20-82）
- 依赖任务: 无
- 完成标准: `create(agent_type, parent_id, trace_id)` 自动生成 `agent_id`（uuid hex 12 位），无 `trace_id` 自动生成；`update(agent_id, **kw)` 改 `input_tokens / output_tokens / status` 等字段；`complete(agent_id, status)` 写 `end_time + status`；`get_tree(trace_id)` 返回同 trace 全节点；`get_total_tokens(trace_id)` 汇总 in/out tokens；操作不存在 ID 时 no-op。

- [ ] 完成

## T7: 实现 `TaskManager` + `BackgroundTask` 状态机

- 影响文件: `mewcode/agents/task_manager.py`（`BackgroundTask` @ 19-31，`TaskManager` @ 34-50，`launch` @ 52-72，`_run_background` @ 74-99，`adopt_running` @ 101-122，`_continue_background` @ 124-145，`get / list_tasks / cancel / poll_completed` @ 147-178）
- 依赖任务: 无
- 完成标准: 状态机覆盖 `running / completed / failed / cancelled`；`launch` 启动 `asyncio.create_task(self._run_background(...))`，task 完成后把 `task_id` 写进 `_notify_queue`；`poll_completed` 用 `get_nowait` 一次性抽空队列；`cancel` 仅对 `running` 任务有效，调 `asyncio.Task.cancel()`；`adopt_running` 把已有 Agent 实例挂为后台任务继续执行，partial result 拼接。

- [ ] 完成

## T8: 实现 `format_task_notification` + `inject_task_notifications`

- 影响文件: `mewcode/agents/notification.py`（`MAX_NOTIFICATION_RESULT_LENGTH=5000` @ 12，`format_task_notification` @ 15-44，`inject_task_notifications` @ 47-51）
- 依赖任务: T7
- 完成标准: `format_task_notification` 输出 `<task-notification>` 标签包裹的文本，含 `Task ID / Agent / Status / Elapsed / Tokens / Result`；超过 5000 字符的 result 截断为 `...\n... (truncated)`；`inject_task_notifications(conv, completed)` 把每个 task 包成 user message 追加到 conversation。

- [ ] 完成

## T9: 实现 `AgentToolParams` + `AgentTool` 类壳

- 影响文件: `mewcode/tools/agent_tool.py`（`AgentToolParams` @ 21-30，`PERMISSION_MODE_MAP` @ 33-37，`TEAMMATE_ADDENDUM` @ 40-51，`AgentTool` @ 54-83）
- 依赖任务: T1, T3, T4, T5, T6, T7
- 完成标准: `AgentToolParams` 8 字段（`prompt / description` 必填，其余可选）；`AgentTool.name = "Agent"`，`category = "command"`，`is_concurrency_safe = False`；构造函数接受 `agent_loader / task_manager / trace_manager / parent_agent / enable_fork / provider_config / worktree_manager / team_manager`。

- [ ] 完成

## T10: 实现 `AgentTool.execute` 五条分支

- 影响文件: `mewcode/tools/agent_tool.py`（`execute` @ 85-238）
- 依赖任务: T9
- 完成标准: 按 `team_name → isolation=="worktree" → subagent_type=="" (fork) → default sync/background` 顺序分发；`subagent_type` 给但 `loader.get` 返 None 报错列出可用类型；fork 路径在 `enable_fork=False` 时报错；`is_background = run_in_background or definition.background or enable_fork`；background 路径走 `task_manager.launch` 返回 `Task ID` 文案；前台路径异常时把 `trace_node` 标 `failed` 并返回错误。

- [ ] 完成

## T11: 实现 `_execute_with_worktree`（isolation=worktree 路径）

- 影响文件: `mewcode/tools/agent_tool.py`（`_execute_with_worktree` @ 491-625）
- 依赖任务: T10
- 完成标准: `worktree_manager is None` 报错；`worktree_manager.create(wt_name, "HEAD")` 创建临时分支；任务前缀拼 `build_worktree_notice(parent.work_dir, wt.path)`；同步 `await sub_agent.run_to_completion(task)`；结束调 `worktree_manager.auto_cleanup(wt_name, wt.head_commit)`，`cleanup.kept` 为真时结果尾部追加 `[Worktree preserved at {cleanup.path}, branch {cleanup.branch}]`。

- [ ] 完成

## T12: 实现 `_execute_as_teammate`（团队成员路径，衔接 ch15）

- 影响文件: `mewcode/tools/agent_tool.py`（`_execute_as_teammate` @ 240-419，`_spawn_pane_teammate` @ 421-471）
- 依赖任务: T10（ch15 的 `TeamManager.detect_backend` / `register_member` / `build_teammate_tools`）
- 完成标准: 校验 `team_manager` / `worktree_manager` 非空；team 存在；同 team 内自动重命名 `<base>-<n>`；`build_teammate_tools` 装配（含 `TaskCreate / TaskGet / TaskList / TaskUpdate / SendMessage`）；`backend ∈ {TMUX, ITERM2}` 走 `_spawn_pane_teammate`；in-process 走 `task_manager.launch`；spec system_prompt 后拼 `TEAMMATE_ADDENDUM`。

- [ ] 完成

## T13: 实现模型路由 `_select_llm` + `_create_client_for_model`

- 影响文件: `mewcode/tools/agent_tool.py`（`_select_llm` @ 473-489，`_create_client_for_model` @ 627-654）
- 依赖任务: T9
- 完成标准: `params.model` 优先，其次 `definition.model`（`!= "inherit"`），fallback 父 client；`_create_client_for_model` 用 `model_map` 把 `haiku/sonnet/opus` 别名解析为完整 model id，调 `create_client(ProviderConfig)`；失败返回 None 退到父 client。

- [ ] 完成

## T14: 接入主流程（app 装配 + 主循环 hooks）

- 影响文件: `mewcode/app.py`（`AgentLoader` import @ 66，`TaskManager` @ 67，`TraceManager` @ 68，`inject_task_notifications` @ 69，`AgentTool` @ 78；`self.agent_loader` 字段 @ 559-561；`AgentLoader` 实例化 @ 725-728；`AgentTool` 注册 @ 737-747；agent catalog 喂回 agent @ 764-788；slash 命令 `tasks` / `trace` 注册 @ 790-794；`adopt_running` 调用 @ 1029-1031；`poll_completed` + `inject_task_notifications` 调用 @ 1275-1279）
- 依赖任务: T1-T13
- 完成标准:
  1. `self.registry.register(agent_tool)` 在 `app.py:747` 注册；
  2. `self.agent_loader = AgentLoader(...)` 在 `app.py:725` 实例化，`load_all` 立即调；
  3. `self.agent.set_agent_catalog(...)` 在 `app.py:788` 把 catalog 喂给主 Agent；
  4. 中断路径在 `app.py:1029` 调 `task_manager.adopt_running` 把当前 stream 转后台；
  5. 主循环 `_check_completed_tasks` 在 `app.py:1275` 调 `task_manager.poll_completed` + `inject_task_notifications(self.conversation, completed)`。

- [ ] 完成

## T15: 端到端验证

- 影响文件: 无（仅运行验证）
- 依赖任务: T14
- 完成标准:
  - `ruff check mewcode tests` 无新增告警；
  - `pytest tests/test_subagent.py -v` 11 个测试类全部通过（`TestAgentParser / TestAgentLoader / TestToolFilter / TestForkMode / TestTraceManager / TestTaskManager / TestNotification / TestConfig / TestPermissionMode / TestAgentToolParams / TestAgentExtensions`）；
  - 端到端路径通过现有测试覆盖：Markdown 解析、builtin / project 覆盖、四层过滤所有分支、fork 嵌套拒绝、TaskManager 状态机、notification 注入。

- [ ] 完成

## 进度

- [ ] T1 / [ ] T2 / [ ] T3 / [ ] T4 / [ ] T5 / [ ] T6 / [ ] T7 / [ ] T8 / [ ] T9 / [ ] T10 / [ ] T11 / [ ] T12 / [ ] T13 / [ ] T14 / [ ] T15
