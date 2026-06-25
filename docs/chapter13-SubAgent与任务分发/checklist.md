# ch13: SubAgent Checklist

> 所有条目可勾选、可观测。验收方式写在条目后面括号中。验收：已通过验证的项均勾选。

## 1. 实现完整性

- [ ] 类 `AgentTool` 在 `mewcode/tools/agent_tool.py:54-83` 存在，构造参数含 `agent_loader / task_manager / trace_manager / parent_agent / enable_fork / provider_config / worktree_manager / team_manager`
- [ ] dataclass `AgentDef` 在 `mewcode/agents/parser.py:23-35` 存在，12 个字段齐全（含 `agent_type / when_to_use / system_prompt / tools / disallowed_tools / model / max_turns / permission_mode / background / isolation / file_path / source`）
- [ ] pydantic 模型 `AgentToolParams` 在 `mewcode/tools/agent_tool.py:21-30` 存在，必填 `prompt / description`，可选 `subagent_type / model / run_in_background / name / isolation / team_name`
- [ ] 类 `TaskManager` / `BackgroundTask` / `ProgressInfo` 在 `mewcode/agents/task_manager.py:34/19/14` 存在，含 `_notify_queue: asyncio.Queue`
- [ ] 类 `TraceManager` / `TraceNode` 在 `mewcode/agents/trace.py:20/8` 存在，三元组 `agent_id / parent_id / trace_id`
- [ ] 三档 builtin 在 `mewcode/agents/builtins/{general-purpose,plan,explore}.md` 存在；`Plan` 的 `disallowedTools` 含 `Agent / EditFile / WriteFile / NotebookEdit` 且 `maxTurns: 15`；`Explore` 的 `model: haiku` + `maxTurns: 30`
- [ ] `resolve_agent_tools` 在 `mewcode/agents/tool_filter.py:83-126` 实现四层过滤
- [ ] `parse_agent_file` 在 `mewcode/agents/parser.py:97-119` 验证 `name` / `description` 必填，`model` / `permissionMode` / `isolation` 取值白名单
- [ ] `build_forked_messages` 在 `mewcode/agents/fork.py:30-79` 嵌套 fork 检查（扫描 `FORK_BOILERPLATE_TAG`）
- [ ] `build_forked_messages` 在 `mewcode/agents/fork.py:55-74` 给悬挂 `tool_uses` 补 `"interrupted"` placeholder `ToolResultBlock`
- [ ] 错误消息 `"Cannot fork from a forked agent."` 在 `mewcode/agents/fork.py:36` 与原始定义的 fork 检查语义一致

## 2. 接入完整性（必查，杜绝死代码）

- [ ] `grep -rn "AgentTool(" mewcode --include="*.py"` 在 `mewcode/app.py:737` 找到注册调用方
- [ ] `self.registry.register(agent_tool)` 调用点在主流程 `mewcode/app.py:747`，所有依赖（`agent_loader / task_manager / trace_manager / parent_agent / enable_fork / provider_config / worktree_manager / team_manager`）齐全注入
- [ ] `AgentLoader(...).load_all()` 调用点在 `mewcode/app.py:725-728`
- [ ] `task_manager.poll_completed` 调用点在 `mewcode/app.py:1275`（通过 `_check_completed_tasks`）
- [ ] `inject_task_notifications` 调用点在 `mewcode/app.py:1279`
- [ ] `task_manager.adopt_running` 调用点在 `mewcode/app.py:1029`（中断触发的后台挂载）
- [ ] `agent.set_agent_catalog` 调用点在 `mewcode/app.py:788`（把 catalog 喂给主 Agent 系统提示）
- [ ] `tasks` / `trace` slash 命令在 `mewcode/app.py:790-794` 注册
- [ ] Schema 暴露：`Agent` 工具通过 `AgentTool.params_model = AgentToolParams` 注册到 registry，TUI 的 `ToolSearch` 可发现它

## 3. 编译与测试

- [ ] `ruff check mewcode tests` 通过（无新增告警）
- [ ] `pytest tests/test_subagent.py -v` 通过（`TestAgentParser` 13 个 + `TestAgentLoader` 9 个 + `TestToolFilter` 7 个 + `TestForkMode` 5 个 + `TestTraceManager` 9 个 + `TestTaskManager` 6 个 + `TestNotification` 3 个 + `TestConfig` 2 个 + `TestPermissionMode` 1 个 + `TestAgentToolParams` 2 个 + `TestAgentExtensions` 2 个 全部 PASS）
- [ ] `pytest tests/ -q` 全套通过

## 4. 端到端验证

- [ ] 注册路径：在 app 启动后 `agent_tool` 放入 registry（`app.py:747`）；用户向主 Agent 发送 "spawn a Plan agent to review X" → LLM 返回 `Agent` 工具调用 → `execute` → 同步路径 `await sub_agent.run_to_completion(prompt)` → 子 Agent 输出文本返回主 Agent
- [ ] Fork 路径：`enable_fork=true` 时用户说 "fork to investigate Y" → LLM 调用 `Agent` 不带 `subagent_type` → `build_forked_messages` → `is_background=True` 走 `task_manager.launch` → 完成时 `<task-notification>` 通过 `poll_completed + inject_task_notifications` 注入下一轮（`app.py:1275-1279`）
- [ ] 后台路径：调用带 `run_in_background=true` 或定义 `background: true` → 立即返回 `Task ID: ...` 文案 → 后台 `asyncio.Task` 完成后 `task_id` 入队
- [ ] 中断挂后台路径：用户中断 → `app.py:1029` 调 `task_manager.adopt_running` → 当前 Agent 转后台 task，状态从 `running` 走完整状态机
- [ ] 证据：单元测试 + grep 调用方 + 主流程文件行号已列出

## 5. 文档

- [ ] `docs/python/ch13/spec.md` 已写
- [ ] `docs/python/ch13/tasks.md` 已写，15 个 T 全部勾完
- [ ] `docs/python/ch13/checklist.md` 已写并逐项验收
- [ ] commit 信息标注 `ch13` 与三件套关闭状态（待用户确认后由人或 CI 触发）

---

## 6. 关键常量与字段（grep 验证）

- [ ] `ALL_AGENT_DISALLOWED_TOOLS` 在 `mewcode/agents/tool_filter.py:12-20` 含七项：`TaskOutput / ExitPlanMode / EnterPlanMode / Agent / AskUserQuestion / TaskStop / Workflow`
- [ ] `ASYNC_AGENT_ALLOWED_TOOLS` 在 `mewcode/agents/tool_filter.py:32-49` 含 16 项：`ReadFile / WebSearch / TodoWrite / Grep / WebFetch / Glob / Bash / EditFile / WriteFile / NotebookEdit / Skill / LoadSkill / SyntheticOutput / ToolSearch / EnterWorktree / ExitWorktree`
- [ ] `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` 在 `mewcode/agents/tool_filter.py:60-66` 含 `ASYNC + TaskCreate / TaskGet / TaskList / TaskUpdate / SendMessage / CronCreate / CronDelete / CronList`
- [ ] `FORK_BOILERPLATE_TAG = "<fork_boilerplate>"` 在 `mewcode/agents/fork.py:7`
- [ ] `MAX_NOTIFICATION_RESULT_LENGTH = 5000` 在 `mewcode/agents/notification.py:12`
- [ ] `VALID_MODELS = {"inherit", "sonnet", "opus", "haiku", ""}` 在 `mewcode/agents/parser.py:11`
- [ ] `VALID_PERMISSION_MODES = {"default", "acceptEdits", "dontAsk", ""}` 在 `mewcode/agents/parser.py:12`
- [ ] `VALID_ISOLATION_MODES = {"", "worktree"}` 在 `mewcode/agents/parser.py:20`
- [ ] `PROJECT_AGENTS_DIR = ".mewcode/agents"` 与 `USER_AGENTS_DIR = "~/.mewcode/agents"` 在 `mewcode/agents/loader.py:11-12`
- [ ] `PERMISSION_MODE_MAP` 在 `mewcode/tools/agent_tool.py:33-37` 把 `default / acceptEdits / dontAsk` 映射到 `PermissionMode` 枚举
- [ ] `TEAMMATE_ADDENDUM` 在 `mewcode/tools/agent_tool.py:40-51` 包含 `"You are running as an agent in a team"` 提示

## 7. 测试用例点名（pytest）

- [ ] `TestAgentParser::test_parse_valid_agent` PASS
- [ ] `TestAgentParser::test_parse_missing_name` / `test_parse_missing_description` PASS
- [ ] `TestAgentParser::test_parse_invalid_model` / `test_parse_invalid_permission_mode` PASS
- [ ] `TestAgentLoader::test_load_builtins`：`Explore / Plan / general-purpose` 三档全在
- [ ] `TestAgentLoader::test_verification_disabled_by_default` / `test_verification_enabled` PASS
- [ ] `TestAgentLoader::test_project_overrides_builtin` PASS（项目级覆盖 builtin）
- [ ] `TestAgentLoader::test_hot_reload` PASS
- [ ] `TestToolFilter::test_global_disallowed` / `test_disallowed_tools_in_definition` / `test_tools_whitelist` / `test_background_whitelist` / `test_combined_whitelist_and_blacklist` / `test_custom_agent_extra_restrictions` / `test_builtin_no_custom_restrictions` 全部 PASS
- [ ] `TestForkMode::test_basic_fork` / `test_fork_preserves_history` / `test_fork_wraps_pending_tool_use` / `test_no_double_fork` / `test_fork_is_deep_copy` PASS
- [ ] `TestTraceManager::test_create_node` / `test_get_tree` / `test_get_total_tokens` PASS
- [ ] `TestTaskManager::test_launch_and_complete` / `test_poll_completed` / `test_cancel` / `test_failed_task` / `test_list_tasks` PASS
- [ ] `TestNotification::test_format_notification` / `test_truncate_long_result` / `test_inject_notifications` PASS
- [ ] `TestAgentToolParams::test_required_fields` / `test_optional_fields` PASS
