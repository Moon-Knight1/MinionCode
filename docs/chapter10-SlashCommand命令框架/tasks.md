# ch10: Slash Command Tasks

## T1: 定义命令类型、Context、UI 协议与 Registry

- 影响文件: `mewcode/commands/registry.py`
- 依赖任务: 无
- 完成标准: `CommandType / UIController / CommandContext / Command / CommandRegistry` 在 `registry.py` 实现；`register / register_sync / find / list_commands` 行为正确，alias 冲突抛 `ValueError`。
- 实际产出: `registry.py:9-13`（`CommandType`）、`registry.py:15-20`（`UIController`）、`registry.py:23-32`（`CommandContext`）、`registry.py:38-47`（`Command`）、`registry.py:50-94`（`CommandRegistry`）

## T2: 实现 parse_command 与 complete

- 影响文件: `mewcode/commands/parser.py`
- 依赖任务: T1
- 完成标准: `parse_command("/foo bar")` 返回 `("foo", "bar", True)`；`parse_command("nothing")` 返回 `("", "", False)`；`parse_command("/")` 返回 `("", "", True)`；name 强制小写。`complete(registry, "/h")` 返回所有 name/alias 命中 `h` 前缀的 `"/" + name` 字典序列表。
- 实际产出: `parser.py:6-16`（`parse_command`）、`parser.py:19-29`（`complete`）

## T3: 实现 10 个内置 handler

- 影响文件: `mewcode/commands/handlers/help.py`、`clear.py`、`compact.py`、`plan.py`、`do.py`、`session.py`、`memory.py`、`permission.py`、`status.py`、`skill.py`
- 依赖任务: T1
- 完成标准: 每个 handler 暴露 `async def handle_<name>(ctx)` 与 `<NAME>_COMMAND` 顶层常量；handler 类型与 spec.F4 一致；`/help` 支持 `args = ""` 列出全部、`args` 非空时打印单命令详情。
- 实际产出:
  - `help.py:12-39`（handler）、`help.py:41-48`（HELP_COMMAND，aliases `["h", "?"]`）
  - `clear.py:7-23`（handler）、`clear.py:26-32`（LOCAL_UI）
  - `compact.py:6-23`（handler）、`compact.py:26-33`（aliases `["c"]`）
  - `plan.py:6-10`（handler）、`plan.py:13-19`（LOCAL_UI，aliases `["p"]`）
  - `do.py:6-8`（handler）、`do.py:11-16`（LOCAL_UI）
  - `session.py:8-110`（handler，含 list/resume/new/delete 四个子命令）
  - `memory.py:6-39`（handler，list/clear/edit）
  - `permission.py:11-110`（handler，mode/rules/add/reset）
  - `status.py:11-43`（handler）、`status.py:45-52`（aliases `["s"]`）
  - `skill.py:11-29`（handler，list/info/reload 三个子命令）、`skill.py:84-92`（aliases `["skills"]`）

## T4: register_all_commands 聚合入口

- 影响文件: `mewcode/commands/handlers/__init__.py`
- 依赖任务: T3
- 完成标准: `ALL_COMMANDS` 列出 10 个常量；`register_all_commands(registry)` 调用 `register_sync` 逐个注册，无 alias 冲突；模块顶层只 import 不副作用。
- 实际产出: `handlers/__init__.py:15-26`（ALL_COMMANDS）、`handlers/__init__.py:29-31`（register_all_commands）

## T5: 带依赖的命令工厂（review / tasks / trace / worktree / skill_register）

- 影响文件: `mewcode/commands/handlers/review.py`、`tasks.py`、`trace.py`、`worktree.py`、`skill_register.py`
- 依赖任务: T1
- 完成标准: 这些命令依赖 `WorktreeManager / TaskManager / TraceManager / SkillExecutor`，必须用工厂函数（`create_*_command` / `register_skill_commands`）在 app 启动后注入；review 是无依赖 PROMPT 命令但保留在工厂层，由调用方决定何时注册。
- 实际产出:
  - `review.py:7-12`（REVIEW_PROMPT）、`review.py:15-19`（handler）、`review.py:22-28`（PROMPT）
  - `tasks.py:65-95`（create_tasks_handler）、`tasks.py:98-106`（create_tasks_command）
  - `trace.py:23-69`（create_trace_command）
  - `worktree.py:13-45`（create_worktree_command）、`worktree.py:48-167`（子命令实现）
  - `skill_register.py:18-95`（register_skill_commands，含 fork/inline 双路径）

## T6: 输入框 Tab 补全 UI 组件

- 影响文件: `mewcode/commands/completion.py`
- 依赖任务: T2
- 完成标准: `CompletionPopup` 继承 `textual.containers.Vertical`，dock 在底部；`show(items) / hide()` / `is_visible` 与 `Selected` 消息齐备；点选后发 `Selected(value)` 并自动隐藏。
- 实际产出: `completion.py:9-57`（CompletionPopup）

## T7: 单元测试

- 影响文件: `tests/test_commands.py`
- 依赖任务: T1-T5
- 完成标准: 覆盖 `parse_command` 所有边界（空串、空白、纯 `/`、`/HELP` 大小写、多 args）；`CommandRegistry` 注册 / 查找 / alias / 冲突 / hidden / 异步注册；`complete` 前缀 / alias / hidden 排除 / 无命中；`register_all_commands` 数量 10、aliases 通；`handle_help / handle_plan / handle_do / handle_skill / handle_status / handle_session / handle_memory` 主路径与异常路径。
- 实际产出: `tests/test_commands.py:80-128`（parse）、`128-184`（registry）、`192-227`（complete）、`235-285`（help）、`288-313`（plan/do）、`316-348`（skill）、`352-372`（status）、`376-405`（session）、`408-441`（memory）、`447-477`（register_all）

## T8: 接入主流程 —— MewCodeApp 注册 / 补全 / 分发

- 影响文件: `mewcode/app.py`
- 依赖任务: T1-T6
- 完成标准: `MewCodeApp` 持有 `self.command_registry: CommandRegistry`，构造时填默认命令；provider 就绪后追加 worktree / tasks / trace 命令与 skill 命令；输入 `/` 后 `on_chat_input_tab_complete` 触发 `complete`；`on_chat_input_submitted` 走 `_dispatch_command` → `parse_command` → `find` → `cmd.handler(ctx)`。
- 实际产出:
  - 注册: `app.py:554-555`（`self.command_registry = CommandRegistry()` / `register_all_commands`）
  - skill→command: `app.py:687-689`（`register_skill_commands`）
  - worktree 命令: `app.py:708-709`（`create_worktree_command` → `register_sync`）
  - tasks 命令: `app.py:790-791`（`create_tasks_command`）
  - trace 命令: `app.py:793-795`（`create_trace_command`）
  - context 构造器: `app.py:870-888`（`_build_command_context`）
  - 分发: `app.py:900-934`（`_dispatch_command`）
  - 补全入口: `app.py:951-961`（`on_chat_input_tab_complete`）
  - 选中回填: `app.py:970-1014`（`on_completion_popup_selected`）
  - 弹窗组件挂载: `app.py:597`（`yield CompletionPopup()`）

## T9: 端到端验证

- 影响文件: 无
- 依赖任务: T8
- 完成标准: TUI 启动后输入 `/status` 看到 `MewCode 状态`、`模式 / 会话 / Token / 工具 / 记忆 / 工作目录 / 版本` 全部字段；输入 `/` 弹补全菜单；输入 `/review` 把 `REVIEW_PROMPT` 发回 Agent；`pytest tests/test_commands.py` 与 `ruff check mewcode/commands/` 全绿。
- 实际产出: `tests/test_commands.py` 覆盖了 parse / registry / complete / 7 个 handler 与 register_all 的端到端逻辑；TUI 流程通过手动验证（见 checklist 4）。

## 进度

- [ ] T1
- [ ] T2
- [ ] T3
- [ ] T4
- [ ] T5
- [ ] T6
- [ ] T7
- [ ] T8
- [ ] T9
