# ch15: AgentTeam Checklist

> 所有条目可勾选、可观测。验收方式写在条目后面括号中。验收：已通过验证的项均勾选。

## 1. 实现完整性

- [ ] 枚举 `BackendType` 在 `mewcode/teams/models.py:10-13` 含三档常量 `TMUX="tmux" / ITERM2="iterm2" / IN_PROCESS="in-process"`
- [ ] dataclass `TeammateInfo` 在 `mewcode/teams/models.py:16-31` 7 字段含 `name / agent_id / agent_type / model / worktree_path / backend_type / is_active`，`is_active: bool | None = None` 三值语义
- [ ] dataclass `AgentTeam` 在 `mewcode/teams/models.py:40-102` 含 `members: list[TeammateInfo]`、`get_member` 同时按 name 和 agent_id 双向查找、`set_member_active` / `all_idle` / `active_members` / `save` / `load` 全方法
- [ ] `resolve_team_dir` / `unique_team_name` 在 `mewcode/teams/models.py:105-117`，落到 `~/.mewcode/teams/<slug>/`，同名冲突自动加 `-2/-3/...` 后缀
- [ ] `MailboxMessage` dataclass 在 `mewcode/teams/mailbox.py:11-27` 8 字段，`message_type` 注释三档 `text | shutdown_request | shutdown_response`
- [ ] `Mailbox` 在 `mewcode/teams/mailbox.py:30-102` 实现单文件单消息模型 `<base>/<agent_id>/<timestamp>_<id>.json`，`write / read / consume / broadcast / cleanup / cleanup_all` 六个方法齐全
- [ ] `create_message` 在 `mewcode/teams/mailbox.py:105-122` 自动填 `uuid.uuid4().hex[:12]` 和 `time.time()`
- [ ] `BackendDetectionError` + `detect_backend` 在 `mewcode/teams/backend_detect.py:9-51` 实现优先级链，失败抛错而非静默回退
- [ ] `SharedTask` + `SharedTaskStore` 在 `mewcode/teams/shared_task.py:9-` 实现 JSON 文件 `{"next_id", "tasks": [...]}` 存储和 `create / get / list_tasks / update / init_empty` 五方法
- [ ] `AgentNameRegistry` 单例在 `mewcode/teams/registry.py:6-40` 线程安全 double-checked locking，`resolve` 同时支持 name 和 agent_id 反查
- [ ] `InProcessTeammateHandle` + `spawn_inprocess_teammate` 在 `mewcode/teams/spawn_inprocess.py:14-56` 用 `asyncio.create_task` 起协程；handle.done / result / cancel 三属性
- [ ] `build_cli_command` 在 `mewcode/teams/spawn_tmux.py:32-56` 输出 `MEWCODE_TEAM_NAME=X MEWCODE_TEAMMATE_NAME=Y MEWCODE_MAILBOX_DIR=Z mewcode -p --work-dir <wt> '<prompt>'`，prompt 内单引号转义为 `'\''`
- [ ] `spawn_tmux_teammate` 在 `mewcode/teams/spawn_tmux.py:59-108` 三级 fallback（split-window → new-window → new-session）
- [ ] `kill_pane` / `send_keys_to_pane` 在 `mewcode/teams/spawn_tmux.py:111-122` best-effort 静默失败
- [ ] `spawn_iterm2_teammate` 在 `mewcode/teams/spawn_iterm2.py:31-58` 复用 `build_cli_command`，通过 `it2 split-pane` 创建 pane
- [ ] `save_transcript / load_transcript` 在 `mewcode/teams/transcript.py:67-92` 序列化 `ConversationManager.history` 含 tool_uses / tool_results 块到 `<team_dir>/transcripts/<agent_id>.json`
- [ ] `TeamManager` 在 `mewcode/teams/manager.py:31-201` 7 内部字典 + 13 个公开方法齐全；`__init__` 接受 `worktree_manager` 和 `trace_manager`；`_detected_backend` 第一次后缓存
- [ ] `delete_team` 在 `mewcode/teams/manager.py:163-201` 先校验 `is_active is not False` 必须 idle，否则抛 `TeamError`
- [ ] `COORDINATOR_MODE_ALLOWED_TOOLS` 在 `mewcode/agents/tool_filter.py:66-79` 含 12 项 `{Agent, SendMessage, TaskCreate, TaskGet, TaskList, TaskUpdate, TeamCreate, TeamDelete, ReadFile, Glob, Grep, Bash}`（写工具 `WriteFile / EditFile` 被排除）
- [ ] `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` 在 `mewcode/agents/tool_filter.py:58-64` 是 `ASYNC_AGENT_ALLOWED_TOOLS | TEAMMATE_COORDINATION_TOOLS | {CronCreate, CronDelete, CronList}` 联合
- [ ] `build_teammate_tools` 在 `mewcode/agents/tool_filter.py:129-184` 按 backend 类型分流：in-process 严格白名单、pane 模式只剔除 `TeamCreate` 和 `TeamDelete`
- [ ] `apply_coordinator_filter` 在 `mewcode/agents/tool_filter.py:187-193` 把 registry 筛到 `COORDINATOR_MODE_ALLOWED_TOOLS`
- [ ] `get_coordinator_system_prompt` 在 `mewcode/teams/coordinator.py:39-` 输出含 `Research / Synthesis / Implementation / Verification` 四阶段、`<task-notification>` XML 格式、`based on your findings` anti-pattern
- [ ] `SendMessageTool` 在 `mewcode/tools/send_message.py:27-123` 实现 `to / message / summary / message_type / metadata` 五参数；`to == "*"` 走 broadcast；`text` 类型必须有 `summary`
- [ ] `TeamCreateTool` 在 `mewcode/tools/team_create.py:19-85` 实现 `team_name + description`；Coordinator Mode 激活时备份 `_full_registry`
- [ ] `TeamDeleteTool` 在 `mewcode/tools/team_delete.py:18-53` 实现 `team_name`；Coordinator Mode 还原 `_full_registry`
- [ ] `AgentTool._execute_as_teammate` 在 `mewcode/tools/agent_tool.py:246-414` 处理 `team_name != None` 分支，含 worktree 创建 / build_teammate_tools / register_member / spawn 分发

## 2. 接入完整性（必查，杜绝死代码）

- [ ] `grep -n "TeamManager" mewcode/app.py` 在 `mewcode/app.py:731-735` 找到导入和 `self.team_manager = TeamManager(worktree_manager, trace_manager)` 创建
- [ ] `grep -n "TeamCreateTool\|TeamDeleteTool" mewcode/app.py` 在 `mewcode/app.py:749-762` 找到两个工具注册点
- [ ] `agent_tool = AgentTool(..., team_manager=self.team_manager)` 注入点在 `mewcode/app.py:737-746`
- [ ] `self.agent._team_manager = self.team_manager` 注入点在 `mewcode/app.py:801`
- [ ] `self.team_manager.on_teammate_completed(task.agent.agent_id)` 在 `mewcode/app.py:1287-1288` 后台任务完成回调
- [ ] shutdown 清理在 `mewcode/app.py:1592-1598` 遍历所有团队强制 set_member_active(False) 后 delete_team
- [ ] `mewcode/__main__.py:57-58` 把 `config.teammate_mode` / `config.enable_coordinator_mode` 透传 `MewCodeApp`
- [ ] `Agent.__init__` 在 `mewcode/agent.py:324-326` 声明 `self.coordinator_mode / self.team_name / self._team_manager` 三字段
- [ ] `Agent._consume_mailbox` 在 `mewcode/agent.py:718-733` 实现；`mewcode/agent.py:433 + :957` 在主循环开头钩入
- [ ] `Agent.run_to_completion` 在 `mewcode/agent.py:471 + :937` 把 `coordinator_mode=self.coordinator_mode` 传给 `build_system_prompt`
- [ ] `AgentTool.execute` 入口分支在 `mewcode/tools/agent_tool.py:86-87` 看到 `p.team_name` 非空时优先走 `_execute_as_teammate`
- [ ] `AgentTool.__init__` 接受 `team_manager` 参数在 `mewcode/tools/agent_tool.py:72`，写入 `self._team_manager` 在 `:81`

## 3. 编译与测试

- [ ] `ruff check mewcode/teams mewcode/tools/team_create.py mewcode/tools/team_delete.py mewcode/tools/send_message.py` 无错误
- [ ] `pytest tests/test_teams.py -v` 通过（覆盖至少 30 个用例：TestModels 7 个 / TestSharedTaskStore 6 个 / TestMailbox 5 个 / TestAgentNameRegistry 4 个 / TestBackendDetect 6 个 / TestToolFilter 3 个 / TestCoordinatorMode 11 个 / TestConfigExtensions 3 个 / TestTranscript 2 个 / TestAgentCoordinatorIntegration 3 个）
- [ ] `pytest tests/test_subagent.py -v` 全部通过（确保 AgentTool 改造未破坏 ch13）
- [ ] `pytest tests/test_agent.py -v` 全部通过（确保 Agent.__init__ 新字段未破坏现有用例）
- [ ] 测试运行不在用户主目录残留 `~/.mewcode/teams/` 目录（fixture 用 `patch("mewcode.teams.models.Path.home", return_value=Path(tmp_dir))` 重定向）

## 4. 端到端验证

- [ ] 注册路径：`MewCodeApp.__init__` 在 `mewcode/app.py:730-762` 创建 `TeamManager` 并把 `TeamCreate / TeamDelete / SendMessage` 三件套放入 registry；用户向 Lead 说 "create a team to refactor X" → LLM 调 `TeamCreate(team_name="refactor-X")` → `detect_backend()` 选模式 → Output 返回 `Team refactor-X created successfully. Backend: ... Config: ~/.mewcode/teams/refactor-x/config.json`
- [ ] Spawn 路径：Lead 继续说 "spawn alice to do data layer" → LLM 调 `Agent(team_name="refactor-X", name="alice", prompt="...")` → `AgentTool.execute` 在 `mewcode/tools/agent_tool.py:86-87` 识别 `team_name` 分支调 `_execute_as_teammate` → `worktree_manager.create(f"team-refactor-X/alice")` → `build_teammate_tools` → `spawn_inprocess_teammate` / `spawn_tmux_teammate` / `spawn_iterm2_teammate` 按 backend 分发 → 队员开始干活
- [ ] 通信路径：队员 alice 通过 `SendMessage(to="bob", message="...", summary="...")` 给 bob 写 mailbox → `AgentNameRegistry.resolve("bob")` 拿到 target_id → `mailbox.write(target_id, msg)` → `_wake_pane(target_id)`（pane 后端需要）→ bob 下一轮 `_consume_mailbox` 收到消息作为 user message
- [ ] Lead 感知路径：每个队员后台 task 完成时 `app.py:1287-1288` 调 `team_manager.on_teammate_completed(agent_id)` → 找到所在团队后 `set_member_idle(team_name, name)` → 翻 `is_active=False` + 写一条 `Teammate '<name>' is now idle (run_to_completion finished).` 到 Lead 邮箱 → Lead 下一轮 `_consume_mailbox` 注入对话
- [ ] Coordinator Mode 路径：启用 `enable_coordinator_mode=True` 且 `MEWCODE_COORDINATOR_MODE=1` → `TeamCreateTool.execute` 把 `parent_agent.coordinator_mode = True / _full_registry 备份 / registry = apply_coordinator_filter(registry)` → Lead 每轮工具集只剩 12 项白名单；调 `WriteFile` / `EditFile` 会找不到工具被拒绝；`TeamDelete` 清空团队后恢复 `_full_registry`
- [ ] Tmux 后端：`TMUX` env 非空时 `detect_backend` 返 `BackendType.TMUX` → `spawn_tmux_teammate` 用 `build_cli_command` 拼出 `MEWCODE_TEAM_NAME=refactor-X MEWCODE_TEAMMATE_NAME=alice MEWCODE_MAILBOX_DIR=... mewcode -p --work-dir /tmp/wt 'prompt'` → tmux send-keys 启动子进程 → 子进程加载同一份 mailbox 目录开始 _consume_mailbox 轮询
- [ ] iTerm2 后端：`TERM_PROGRAM=iTerm.app` 且 `shutil.which("it2")` 非空且不在 tmux 时 `detect_backend` 返 `BackendType.ITERM2` → `spawn_iterm2_teammate` 用 `it2 split-pane --command "/bin/zsh -c '<cmd>'"` 创建 pane
- [ ] 关闭路径：`TeamDelete(team_name="refactor-X")` → `team_manager.delete_team` → 校验全员 idle → 遍历每个 member 清 name registry / cancel handle / kill pane / git worktree remove / trace_manager.remove → cleanup mailbox + 删团队目录 → 弹出 `_teams / _task_stores / _mailboxes` 三个缓存 → 如 Lead 在 Coordinator Mode 则恢复 `_full_registry`

## 5. 文档

- [ ] `docs/python/ch15/spec.md` 已写
- [ ] `docs/python/ch15/tasks.md` 已写，16 个 T 全部勾完
- [ ] `docs/python/ch15/checklist.md` 已写并逐项验收
- [ ] commit 信息标注 `ch15` 与三件套关闭状态（待用户确认后由人或 CI 触发）
