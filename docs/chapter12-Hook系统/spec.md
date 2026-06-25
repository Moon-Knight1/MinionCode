# ch12: Hook 系统 Spec

## 1. 背景

Agent 主流程在工具调用前后、session 起止、turn 起止、消息收发等关键节点都有「副作用钩子」的需求：工具调用前阻断危险命令、调用后异步推日志到外部系统、用户提交前注入额外提示词、新 session 启动时拉取项目上下文。把这些写死在 Agent 循环里既不优雅又难配置。Hook 系统把这层做成可声明（yaml）+ 条件匹配 + 多种动作类型的引擎，并保留 once / async（async_exec）/ reject 三种执行控制。Python 实现使用 asyncio 协程作为执行单位，配合 `asyncio.create_subprocess_shell` 跑外部命令，`urllib.request` + `run_in_executor` 跑 HTTP。

## 2. 目标

交付 `mewcode.hooks.HookEngine`，从 `config.yaml` 的 `hooks` 数组加载并经 `load_hooks` 校验后注入引擎；提供两类入口：普通事件用 `run_hooks(event, ctx)` 跑全部命中钩子；`pre_tool_use` 用 `run_pre_tool_hooks(ctx)` 允许阻断工具调用并返回 `ToolRejectedError`。Condition 支持 leaf 操作符（`==` / `!=` / `=~` / `~=`）+ 复合（`&&` / `||`，但同一表达式不允许混用）+ 变量覆盖 tool / event / `args.<key>`。`Agent.run` 在 session 入口、turn 入口、每次工具调用前后、消息收发前后接入对应入口；`MewCodeApp` 负责从配置初始化引擎并挂到 Agent，并在 mount / unmount 时触发 startup / shutdown 事件。

## 3. 功能需求

- F1: 提供 15 个生命周期事件常量（`LifecycleEvent` StrEnum）：session_start / session_end / turn_start / turn_end / pre_tool_use / post_tool_use / pre_send / post_receive / startup / shutdown / error / compact / permission_request / file_change / command_execute。
- F2: 提供 4 个动作类型（在 loader 的 `_VALID_ACTION_TYPES` 集合里）：command / prompt / http / agent；`agent` 当前为 stub，返回 "agent executor not yet implemented"。
- F3: Condition DSL：
  - leaf 操作符：`==`（等于）/ `!=`（不等于）/ `=~`（正则，包裹 `/.../` 时自动去除斜杠）/ `~=`（glob，走 `fnmatch.fnmatch`）
  - 复合：`&&` / `||`，但一行表达式只能用一种，混用抛 `ConditionParseError("Cannot mix '&&' and '||'")`
  - 变量解析：`tool` / `event` / `args.<key>`，由 `HookContext.get_field` 实现
  - 模板展开：动作字段中支持 `$EVENT / $TOOL_NAME / $FILE_PATH / $MESSAGE / $ERROR / $TOOL_ARGS.<key>`，由 `HookContext.expand` 替换
- F4: `HookEngine.run_hooks(event, ctx)` 按事件名 + condition 过滤后逐个执行；`async_exec=True` 的 hook 通过 `asyncio.ensure_future` 后台跑，不阻塞主协程；`prompt` 类型成功结果写入 `_prompt_messages` 队列，由 `get_prompt_messages()` 一次性取出后清空。
- F5: `HookEngine.run_pre_tool_hooks(ctx)` 专门跑 `pre_tool_use` 事件：任何 `hook.reject=True` 命中即返回 `ToolRejectedError(tool, reason, hook_id)`；执行异常被捕获并写 log，不影响主流程。
- F6: 动作执行器：
  - command：`asyncio.create_subprocess_shell` 拉子进程，stderr 合并到 stdout，命令字符串先经 `ctx.expand` 替换变量
  - prompt：直接对 `action.message` 做变量替换后返回，`success=True`
  - http：默认 POST，`urllib.request.Request` + `urlopen` 走 `run_in_executor` 异步化；带 body 时自动添加 `Content-Type: application/json`，响应体截断到 500 字节
  - agent：stub，仅记录日志并返回成功占位字符串
- F7: `once` 控制：`Hook.executed` 在首次触发后置 True，`Hook.should_run()` 在 `once=True` 且 `executed=True` 时返回 False，`find_matching_hooks` 据此跳过。
- F8: `async_exec` 控制：`run_hooks` 中以 `asyncio.ensure_future(self._run_single(hook, ctx))` 派发，不 await；`run_pre_tool_hooks` 不支持 async（loader 校验阻止）。
- F9: 加载期校验 `load_hooks(raw_hooks) -> list[Hook]`：
  - event 必须在 `LifecycleEvent` 白名单内
  - action.type 必须在 `_VALID_ACTION_TYPES = {"command","prompt","http","agent"}` 里
  - 按 `_REQUIRED_FIELDS` 强制每种类型的必填字段：command→command、prompt→message、http→url、agent→prompt
  - `reject=True` 只允许配在 `pre_tool_use` 事件上
  - `async=True` 不允许配在 `pre_tool_use` 事件上
  - `action.timeout` 必须是正整数（>0）
  - hook id 缺失时按 `f"{event}_{i}"` 自动生成
  - condition 字符串经 `parse_condition` 解析失败时包成 `HookConfigError`
  - 任意非法配置抛 `HookConfigError`，错误消息带 `f"hook '{id}'"` 或 `f"hook #{index+1}"` 定位
- F10: command 动作超时执行：`execute_command` 使用 `asyncio.wait_for(proc.communicate(), timeout=action.timeout)`，超时时 `proc.kill()` + `await proc.wait()` 清理子进程，返回 `ActionResult(output="Command timed out after Xs: <cmd>", success=False)`；`action.timeout` 默认值在 `Action` dataclass 中为 30 秒。

## 4. 非功能需求

- N1: hook 执行不能让 Agent 主协程崩溃：`_run_single` / `run_pre_tool_hooks` 内层 `try/except Exception`，捕获后记录 warning log 并写入 `_notifications`；condition 正则编译失败按「不命中」处理（`re.error` 返回 False）。
- N2: 并发安全：`HookEngine` 设计运行在单 event loop 上，所有状态修改在协程内顺序进行，无需显式锁；`async_exec` 派生的协程通过 `asyncio.ensure_future` 注册到 loop，由 loop 调度。
- N3: HTTP hook 必须有超时与响应体大小限制：`urlopen(req, timeout=30)`，响应体截断 500 字符；通过 `run_in_executor` 把同步 `urlopen` 放到默认线程池，避免阻塞 event loop。
- N4: `run_pre_tool_hooks` reject 时 `ToolRejectedError.reason` 必须取自 action 输出，Agent loop 包装为 `"Hook rejected: {reason}"` 作为工具结果。
- N5: `load_hooks` 出错时必须能定位到具体 hook：错误消息含 hook id（无则用 `f"hook #{i+1}"`）+ 出错字段名，例如 `hook 'auto-format': action type 'command' requires 'command' field`。
- N6: command 超时不能泄漏子进程：`asyncio.TimeoutError` 分支必须先 `proc.kill()` 再 `await proc.wait()`，确认子进程退出后才返回，避免僵尸进程。

## 5. 设计概要

- 核心数据结构：
  - `LifecycleEvent`（StrEnum，15 个值）：所有生命周期事件常量
  - `Action`（dataclass）：type / command / message / url / method / body / headers / prompt / timeout，单结构承载四种动作类型
  - `Condition`（dataclass）：field / operator / value，叶子条件
  - `ConditionGroup`（dataclass）：conditions 列表 + logic（"and"/"or"），复合条件
  - `Hook`（dataclass）：id / event / action / condition / reject / once / async_exec / executed，配合 `should_run()` / `mark_executed()` 方法
  - `HookContext`（dataclass）：event_name / tool_name / tool_args / file_path / message / error，配合 `get_field()` / `expand()` 方法
  - `ActionResult`（dataclass）：output / success
  - `HookNotification`（dataclass）：hook_id / event / output / success，drain 队列单元
  - `HookEngine`：hooks 列表 + `_prompt_messages` 队列 + `_notifications` 队列
  - `ToolRejectedError`（Exception）：tool / reason / hook_id
  - `HookConfigError`（Exception）：loader 报错
  - `ConditionParseError`（Exception）：condition 解析报错
- 主流程：
  1. `mewcode/__main__.py:main` 启动 → `load_config` 读 `config.yaml` 拿到 `raw_hooks`（dict 列表）
  2. `load_hooks(raw_hooks)` 校验 + 解析成 `list[Hook]`；`HookConfigError` 时打 stderr 并 `sys.exit(1)`
  3. `HookEngine(hooks)` 构造 → 传入 `MewCodeApp(..., hook_engine=...)` → `Agent(..., hook_engine=...)`
  4. App `on_mount` 派发 `startup` 事件；`on_unmount` 派发 `shutdown` 事件
  5. `Agent.run` 入口派发 `session_start` → 每轮 `turn_start` → stream 前 `pre_send` → stream 后 `post_receive` → 退出循环时派发 `turn_end` + `session_end`
  6. 工具调用前调 `run_pre_tool_hooks(ctx)`：返回 `ToolRejectedError` 时打包成 `ToolResult(output=f"Hook rejected: {reason}", is_error=True)`，跳过实际 tool 执行
  7. 工具调用后调 `run_hooks("post_tool_use", ctx)`
  8. 每轮 hook 执行完后调 `_drain_hook_events()`，把 `HookNotification` 转成 `HookEvent` 事件流 yield 给 TUI 展示
- 调用链（模块层级）：
  - 启动：`mewcode/__main__.py` → `load_hooks` → `HookEngine` → `MewCodeApp` → `Agent`
  - 触发：`Agent.run` → `_build_hook_context` → `hook_engine.run_hooks` / `run_pre_tool_hooks` → `execute_action` → `_EXECUTOR_MAP[type]`
- 与其他模块的交互：
  - 上行依赖：`agent.py`（loop 触发）、`app.py`（startup/shutdown）、`config.py`（raw_hooks 字段）、`__main__.py`（装配入口）
  - 下行：仅依赖 Python 标准库（asyncio / urllib / fnmatch / re）

## 6. Out of Scope

- `agent` action type 真正实现：当前是 stub，建议在 ch13 SubAgent 稳定后再补上（调 `Agent.run` 单轮）
- 已声明但未在主流程触发的事件（error / compact / permission_request / file_change / command_execute）：等业务场景出现再在对应模块 emit
- Condition DSL 的括号 / 短路求值 / 混合 `&&` 和 `||`：当前实现明确拒绝混用，需要复杂逻辑时建议拆成多个 hook
- Hook 配置的热更新：必须重启进程才生效
- HTTP hook 的认证 / 重试 / mTLS / 大响应流式处理：当前仅支持简单 POST + JSON

## 7. 完成定义

见 [checklist.md](checklist.md)，所有条目勾上即完成。
