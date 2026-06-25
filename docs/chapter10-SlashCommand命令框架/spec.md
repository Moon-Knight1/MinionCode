# ch10: Slash Command Spec

## 1. 背景

TUI 需要一种快捷方式让用户在不打扰主对话的前提下触发本地操作（清屏、切换 Plan/Do 模式、查看 token 状态、操作 session 与记忆），以及调用既定 prompt 模板。直接把这些诉求丢给 LLM 既浪费 token，也无法即时改变 UI 状态（清屏、Plan 模式开关、permission 切换都不是 LLM 能完成的）。Slash Command 把所有以 `/` 开头的输入收编成命名空间，统一注册、补全、解析、分发；缺少这层框架，要么 if/elif 散落，要么所有动作走 Agent Loop，体验和成本都不可接受。

## 2. 目标

交付进程内的 `CommandRegistry`，让 TUI 在用户敲 `/<name> [args]` 时按统一签名调起 handler。内置 help / clear / compact / plan / do / session / memory / permission / status / skill / review / tasks / trace / worktree 等核心命令；每个命令声明 `type ∈ {LOCAL, LOCAL_UI, PROMPT}`，框架据此决定走系统消息回显、UI 状态改写，还是把 prompt 当成 user message 投回 Agent。Skill 系统在 provider 就绪后把每个 skill 注册为 `PROMPT` 类型命令；TUI 在输入框响应 `/` 时实时弹补全菜单。

## 3. 功能需求

- F1: `CommandRegistry` 暴露 `register / register_sync / find / list_commands` 四个方法，并维护 alias → canonical name 投射；冲突时抛 `ValueError`。
- F2: 三类命令类型由 `CommandType` 枚举：`LOCAL`（handler 显示系统消息）、`LOCAL_UI`（handler 改写 UI 状态，例如清屏 / 切 Plan）、`PROMPT`（handler 调用 `ui.send_user_message`，转给 Agent 当用户消息发出）。
- F3: `parse_command(text)` 把 `/foo bar baz` 拆为 `(name, args, is_command)`：非 `/` 前缀返回 `("", "", False)`；只有 `/` 返回 `("", "", True)`；name 自动小写化。
- F4: `register_all_commands(registry)` 一次性注册 10 个内置命令（help / compact / clear / plan / do / session / memory / permission / status / skill）；其余命令（review / tasks / trace / worktree）由 app 在依赖就绪后通过工厂函数注册。
- F5: `complete(registry, prefix)` 接受 `/abc` 形式前缀，遍历所有非 hidden 命令的 name 与 aliases，按字典序返回所有匹配 `"/" + name` 字符串列表。
- F6: `CommandContext` 是 handler 唯一入参，必须承载 `args / agent / conversation / session / session_manager / memory_manager / ui / config`；`config` 是 dict，托管 registry、skill_loader、skill_executor 等需要回写的闭包钩子。
- F7: `UIController` 协议固定 `add_system_message / send_user_message / set_plan_mode / get_token_count / refresh_status` 五个方法，handler 只通过它跟 UI 交互。
- F8: TUI 监听 `/` 前缀输入：`Tab` 触发 `complete` → `CompletionPopup.show`；`Enter` 走 `_dispatch_command` → `parse_command` → `registry.find` → `cmd.handler(ctx)`。

## 4. 非功能需求

- N1: `CommandRegistry.register` 默认禁止冲突，重复 name / alias 抛 `ValueError`；后续 skill 热重载需要先从 `_commands / _alias_map` 主动剔除旧条目。
- N2: `parse_command` 不能抛异常：空串、`/`、连续空格、前导空格、纯空白都要返回稳定 3 元组。
- N3: `CommandRegistry` 在异步路径使用 `asyncio.Lock` 保护并发注册（为 skill 热重载预留）。
- N4: handler 是 `async` 函数，签名只依赖 `CommandContext`；`mewcode.commands` 包不 import TUI 内部模块，反向依赖通过 `config` dict 注入闭包。
- N5: handler 抛异常时由调用方（`_dispatch_command`）捕获并以系统消息形式反馈，单条命令失败不能把 TUI 拉崩。

## 5. 设计概要

- 核心数据结构：
  - `mewcode.commands.registry.CommandType`：枚举 `LOCAL / LOCAL_UI / PROMPT`
  - `mewcode.commands.registry.Command`：name / description / type / handler / aliases / usage / arg_prompt / hidden
  - `mewcode.commands.registry.CommandRegistry`：`_commands` dict + `_alias_map` dict + `asyncio.Lock`
  - `mewcode.commands.registry.CommandContext`：args / agent / conversation / session / session_manager / memory_manager / ui / config
  - `mewcode.commands.registry.UIController`：`Protocol`，规定 5 个 UI 方法
- 主流程：
  1. `MewCodeApp.__init__` → `self.command_registry = CommandRegistry()` → `register_all_commands(self.command_registry)` 装 10 个内置命令
  2. provider 就绪 → 注册 worktree / tasks / trace 等带依赖的命令 → `register_skill_commands` 把 skill catalog 注册为 `PROMPT` 命令
  3. 用户敲 `/` → `on_chat_input_tab_complete` → `complete(registry, prefix)` → 单命中 inline 填回，多命中弹 `CompletionPopup`
  4. 用户回车 → `on_chat_input_submitted` → `_dispatch_command(text)` → `parse_command` → `registry.find` → 构造 `CommandContext` → `await cmd.handler(ctx)`
  5. handler 按 type 行为分化：`LOCAL` 调 `ui.add_system_message`；`LOCAL_UI` 触发 `set_plan_mode` / `clear_chat` / `set_session` 等 UI 副作用；`PROMPT` 调 `ui.send_user_message` 把构造好的 prompt 发回 Agent
- 调用链（模块层级）：
  - `/status` → `MewCodeApp._dispatch_command` → `parse_command` → `registry.find("status")` → `handle_status(ctx)` → `ctx.ui.add_system_message`
  - `/review fix race condition` → 同样路径 → `handle_review` 拼出 `REVIEW_PROMPT + 额外关注` → `ctx.ui.send_user_message` → 进入 Agent Loop
  - `/<skill_name>` → handler 由 `register_skill_commands.make_handler` 闭包构造 → 调 `SkillExecutor.execute_inline / execute_fork`
- 与其他模块的交互：
  - 上行依赖：`MewCodeApp`（注册、补全、分发）、`SkillLoader`（每个 skill 注册成命令）、`WorktreeManager` / `TaskManager` / `TraceManager`（工厂函数注入依赖）
  - 下行：纯接口包，仅 import `mewcode.conversation` / `mewcode.permissions` / `mewcode.memory.session` 等数据模型；不 import 任何 UI / agent 实现

## 6. Out of Scope

- 命令级权限过滤（命令是否能用由调用方决定，框架不裁剪）
- 文件式 markdown 命令加载（Python 版尚未实现 `LoadDir` / 三层目录合并，由后续章节追加）
- 命令链式调用 / 管道（`/a | /b` 不支持）
- 命令的 fuzzy match（`complete` 只做前缀匹配）
- markdown 命令的文件 watcher / 热更新

## 7. 完成定义

见 [checklist.md](checklist.md)，所有条目勾上即完成。
