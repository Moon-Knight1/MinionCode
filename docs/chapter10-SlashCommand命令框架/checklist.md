# ch10: Slash Command Checklist

## 1. 实现完整性

- [ ] `CommandType` 枚举 `LOCAL / LOCAL_UI / PROMPT` 在 `mewcode/commands/registry.py:9-13`
- [ ] `UIController` Protocol 暴露 `add_system_message / send_user_message / set_plan_mode / get_token_count / refresh_status` 在 `mewcode/commands/registry.py:15-20`
- [ ] `CommandContext` dataclass 含 args/agent/conversation/session/session_manager/memory_manager/ui/config 八字段在 `mewcode/commands/registry.py:23-32`
- [ ] `Command` dataclass 含 name/description/type/handler/aliases/usage/arg_prompt/hidden 在 `mewcode/commands/registry.py:38-47`
- [ ] `CommandRegistry.__init__` 含 `_commands / _alias_map / _lock = asyncio.Lock()` 在 `mewcode/commands/registry.py:50-54`
- [ ] `CommandRegistry.register / register_sync / find / list_commands` 在 `mewcode/commands/registry.py:56-94`，alias 冲突抛 `ValueError("conflicts with...")`
- [ ] `parse_command` 在 `mewcode/commands/parser.py:6-16`：非 `/` 前缀返回 `("", "", False)`；`/` 返回 `("", "", True)`；name 小写；空白拆 `(name, args)`
- [ ] `complete` 在 `mewcode/commands/parser.py:19-29`：剥前导 `/`，遍历非 hidden 命令的 name + aliases，返回字典序 `["/xxx", ...]`
- [ ] 10 个内置 handler 全部存在：`mewcode/commands/handlers/{help,clear,compact,plan,do,session,memory,permission,status,skill}.py`，每个文件导出 `handle_<name>` 与 `<NAME>_COMMAND`
- [ ] `register_all_commands` 在 `mewcode/commands/handlers/__init__.py:29-31` 注册 10 个常量，无 alias 冲突
- [ ] 带依赖的命令工厂：`create_tasks_command` / `create_trace_command` / `create_worktree_command` / `register_skill_commands` 分别在 `handlers/tasks.py:98-106`、`handlers/trace.py:23-69`、`handlers/worktree.py:13-45`、`handlers/skill_register.py:18-95`
- [ ] `REVIEW_PROMPT` 文案在 `mewcode/commands/handlers/review.py:7-12` 含「逻辑错误 / 安全问题 / 性能问题 / 代码风格」四条
- [ ] `CompletionPopup` Textual 组件在 `mewcode/commands/completion.py:9-57`，含 `show / hide / Selected` 消息

## 2. 接入完整性

- [ ] `grep -rn "register_all_commands" mewcode/ --include="*.py"` 命中 `mewcode/app.py:46` 和 `mewcode/app.py:555` 的非测试调用
- [ ] `grep -rn "self.command_registry" mewcode/app.py | wc -l` 输出 ≥ 8（注册、补全、分发、_build_command_context、skill_register 都引用）
- [ ] `grep -rn "parse_command" mewcode/app.py` 命中 `mewcode/app.py:901`
- [ ] `grep -rn "from mewcode.commands.completion" mewcode/app.py` 命中 `mewcode/app.py:45`
- [ ] `MewCodeApp.command_registry` 初始化在 `mewcode/app.py:554-555`
- [ ] `register_skill_commands` 调用在 `mewcode/app.py:687-689`
- [ ] `create_worktree_command` / `create_tasks_command` / `create_trace_command` 注册在 `mewcode/app.py:708-709 / 790-791 / 793-795`
- [ ] `CompletionPopup` 挂载在 `mewcode/app.py:597`（`yield CompletionPopup()`）
- [ ] Tab 补全入口 `on_chat_input_tab_complete` 在 `mewcode/app.py:951-961`，调用 `complete(self.command_registry, event.text)`
- [ ] 命令分发 `_dispatch_command` 在 `mewcode/app.py:900-934`，三种 `CommandType` 由 handler 内部决定行为（handler 通过 `ctx.ui` 触发不同副作用）
- [ ] `_build_command_context` 在 `mewcode/app.py:870-888`，config dict 包含 `registry / set_session / set_conversation / clear_chat / render_restored / skill_loader / skill_executor`
- [ ] 入口路径：用户在 TUI 输入 `/<name>` → `on_chat_input_submitted` (`app.py:945`) → `_dispatch_command` (`app.py:900`) → `parse_command` (`app.py:901`) → `registry.find` (`app.py:921`) → `cmd.handler(ctx)` (`app.py:932`)

## 3. 编译与测试

- [ ] `cd /Users/codemelo/mewcode && ruff check mewcode/commands/ tests/test_commands.py` 无错误
- [ ] `cd /Users/codemelo/mewcode && pytest tests/test_commands.py -v` 全部测试通过（`TestParseCommand`、`TestCommandRegistry`、`TestComplete`、`TestHelpHandler`、`TestPlanDoHandlers`、`TestSkillHandler`、`TestStatusHandler`、`TestSessionHandler`、`TestMemoryHandler`、`TestRegisterAllCommands` 等 10 个测试类）
- [ ] `pytest tests/test_commands.py::TestParseCommand -v` 8 个测试用例全过（normal / with_args / case_insensitive / only_slash / not_a_command / empty_input / whitespace_input / leading_spaces / multiple_args）
- [ ] `pytest tests/test_commands.py::TestCommandRegistry -v` 验证 alias 冲突、name 冲突、跨字段冲突、async 注册四类异常路径
- [ ] `pytest tests/test_commands.py::TestRegisterAllCommands::test_all_10_commands_registered -v` 验证 `{help, compact, clear, plan, do, session, memory, permission, status, skill}` 全部到位

## 4. 端到端验证

- [ ] 在 TUI 中输入 `/status` 后看到 `MewCode 状态` 输出（含 `模式 / 会话 / Token / 工具 / 记忆 / 工作目录 / 版本` 字段）—— `mewcode/commands/handlers/status.py:11-43` 的 handler 输出
- [ ] 在 TUI 中输入 `/help` 看到 `可用命令：` 列表 —— `mewcode/commands/handlers/help.py:31-39` 的 handler
- [ ] 在 TUI 中输入 `/h`（别名）效果等同 `/help` —— `mewcode/commands/handlers/help.py:43`（aliases=["h", "?"]）
- [ ] 在 TUI 中输入 `/` 后按 Tab 弹出补全菜单 —— `app.py:951-961` 触发 `popup.show(matches)`
- [ ] 在 TUI 中输入 `/skill list` 看到已加载的 Skill 列表 —— `mewcode/commands/handlers/skill.py:31-41`
- [ ] 在 TUI 中输入 `/review fix race condition`，TUI 把 `REVIEW_PROMPT + "\n\n额外关注：fix race condition"` 当 user message 发给 LLM —— `mewcode/commands/handlers/review.py:15-19`
- [ ] 在 TUI 中输入 `/plan 设计登录模块`，UI 进入 Plan 模式且立刻把「设计登录模块」当 user message 发出 —— `mewcode/commands/handlers/plan.py:6-10`
- [ ] 留存证据：未提供截图（手动 TUI 验证不在课程验收流程要求范围内）

## 5. 文档

- [ ] `docs/python/ch10/spec.md` 存在
- [ ] `docs/python/ch10/tasks.md` 存在
- [ ] `docs/python/ch10/checklist.md` 存在
- [ ] `git log --oneline origin/python -- mewcode/commands/` 至少有一条引入 Slash Command 框架的 commit
