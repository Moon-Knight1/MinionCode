# ch12: Hook 系统 Tasks

## T1: 定义生命周期事件常量

- 影响文件: `mewcode/hooks/events.py`
- 依赖任务: 无
- 完成标准: `LifecycleEvent` StrEnum 包含 15 个事件值（session_start / session_end / turn_start / turn_end / pre_tool_use / post_tool_use / pre_send / post_receive / startup / shutdown / error / compact / permission_request / file_change / command_execute）；可直接和字符串比较。
- 实际产出: `mewcode/hooks/events.py:6-30`

## T2: 数据模型 —— Action / Hook / HookContext / ActionResult / ToolRejectedError

- 影响文件: `mewcode/hooks/models.py`
- 依赖任务: T1
- 完成标准:
  - `Action`（dataclass）字段齐：type / command / message / url / method / body / headers / prompt / timeout（默认 30）
  - `Hook`（dataclass）字段齐：id / event / action / condition / reject / once / async_exec / executed；`should_run` 检查 once + executed；`mark_executed` 翻 True
  - `HookContext`（dataclass）实现 `get_field("tool"/"event"/"args.<key>")` 与 `expand` 模板替换（$EVENT / $TOOL_NAME / $FILE_PATH / $MESSAGE / $ERROR / $TOOL_ARGS.<key>）
  - `ActionResult`（dataclass）含 output / success
  - `ToolRejectedError(Exception)` 带 tool / reason / hook_id 三字段
- 实际产出: `mewcode/hooks/models.py:9-85`

## T3: Condition DSL —— leaf / 复合 / 解析

- 影响文件: `mewcode/hooks/conditions.py`
- 依赖任务: T2
- 完成标准:
  - 支持四种 leaf 操作符：`==` / `!=` / `=~`（正则）/ `~=`（glob，走 `fnmatch.fnmatch`）
  - `=~` 包裹 `/.../` 时自动去除斜杠；`re.error` 时返回 False
  - 支持 `&&` / `||` 复合，但同一表达式混用时 `parse_condition` 抛 `ConditionParseError("Cannot mix '&&' and '||'")`
  - 空表达式或纯空白返回 None
  - 字符串值带双引号时自动去除
- 实际产出: `mewcode/hooks/conditions.py:12-96`（`Condition` / `ConditionGroup` / `parse_condition` / `_parse_single`）

## T4: HookEngine 核心 —— find_matching_hooks / run_hooks / once / async_exec

- 影响文件: `mewcode/hooks/engine.py`
- 依赖任务: T2, T3
- 完成标准:
  - `HookEngine.__init__` 初始化 `hooks` / `_prompt_messages` / `_notifications` 三个状态
  - `find_matching_hooks(event, ctx)` 按事件名 + `should_run` + condition 三层过滤
  - `run_hooks(event, ctx)` 顺序触发：`async_exec=True` 走 `asyncio.ensure_future(self._run_single(...))` 不 await；其余 await
  - `_run_single` 把 `prompt` 类型成功结果写入 `_prompt_messages`，所有结果写 `_notifications`，异常被 catch
  - `get_prompt_messages()` 一次性返回并清空 `_prompt_messages`
  - `drain_notifications()` 一次性返回并清空 `_notifications`
- 实际产出: `mewcode/hooks/engine.py:21-110`

## T5: pre_tool_use 阻断专用入口

- 影响文件: `mewcode/hooks/engine.py`
- 依赖任务: T4
- 完成标准: `run_pre_tool_hooks(ctx) -> ToolRejectedError | None`：顺序执行命中 hook，遇到 `hook.reject=True` 立即返回 `ToolRejectedError(tool=ctx.tool_name, reason=result.output, hook_id=hook.id)`；不允许 reject 时返回 None；执行异常被捕获记 log 不抛出。
- 实际产出: `mewcode/hooks/engine.py:80-103`

## T6: 四种动作执行器（command / prompt / http / agent）

- 影响文件: `mewcode/hooks/executors.py`
- 依赖任务: T2
- 完成标准:
  - `execute_command`：`asyncio.create_subprocess_shell` + stderr 合并到 stdout，命令字符串先 `ctx.expand` 替换变量；`asyncio.wait_for(..., timeout=action.timeout)` 超时时 `proc.kill()` + `await proc.wait()` 并返回 `success=False, output="Command timed out after Xs: <cmd>"`
  - `execute_prompt`：对 `action.message` 跑 `ctx.expand` 后包成 `ActionResult(output=..., success=True)`
  - `execute_http`：默认 POST，`urlopen(req, timeout=30)` 走 `run_in_executor` 异步化；body 非空时自动加 `Content-Type: application/json`；响应体截断 500 字符
  - `execute_agent`：stub，返回 `ActionResult(output="agent executor not yet implemented", success=True)`
  - `execute_action` 通过 `_EXECUTOR_MAP` 派发；未知 type 返回 `success=False`
- 实际产出: `mewcode/hooks/executors.py:13-97`

## T7: 加载与校验 `load_hooks(raw_hooks)`

- 影响文件: `mewcode/hooks/loader.py`
- 依赖任务: T1, T2, T3
- 完成标准:
  - `load_hooks(None)` / `load_hooks([])` 返回 `[]`
  - event 不在 `LifecycleEvent` 白名单内抛 `HookConfigError("invalid event ...")`
  - action.type 不在 `{"command","prompt","http","agent"}` 内抛 `HookConfigError("invalid action type ...")`
  - 按 `_REQUIRED_FIELDS` 校验每种类型必填字段（command→command、prompt→message、http→url、agent→prompt）
  - `reject=True` 且 event != "pre_tool_use" 抛错
  - `async=True` 且 event == "pre_tool_use" 抛错
  - timeout 非正整数抛错
  - hook id 缺失时按 `f"{event}_{i}"` 自动生成
  - condition 字符串经 `parse_condition` 解析失败时包成 `HookConfigError`
  - 错误消息含 hook id（无则用 `f"hook #{i+1}"`）
- 实际产出: `mewcode/hooks/loader.py:7-115`

## T8: 包出口与公共 API

- 影响文件: `mewcode/hooks/__init__.py`
- 依赖任务: T1-T7
- 完成标准: `mewcode.hooks` 包对外暴露 `Action / ActionResult / Condition / ConditionGroup / ConditionParseError / Hook / HookConfigError / HookContext / HookEngine / LifecycleEvent / ToolRejectedError / load_hooks / parse_condition`；测试 `from mewcode.hooks import HookEngine` 能跑通。
- 实际产出: `mewcode/hooks/__init__.py:1-27`

## T9: 单元测试覆盖

- 影响文件: `tests/test_hooks.py`
- 依赖任务: T1-T8
- 完成标准: 覆盖以下场景（每个一个 `pytest` 类）：
  - `TestLifecycleEvent`：15 个事件数量与字符串比较
  - `TestHookContext`：get_field 四种字段、expand 全变量替换、未定义变量保留
  - `TestParseCondition`：单条件 / `&&` / `||` / 混用错误 / 空 / 正则 / 无操作符
  - `TestConditionEvaluate`：四种 leaf 操作符
  - `TestConditionGroupEvaluate`：and 全通过 / and 部分失败 / or 任一通过 / or 全失败 / 空 group
  - `TestCommandExecutor`：正常执行 / 变量替换 / 超时
  - `TestPromptExecutor`：返回 message
  - `TestHttpExecutor`：mock urlopen 验证 status
  - `TestAgentExecutor`：stub 返回不抛错
  - `TestExecuteAction`：派发 + 未知 type
  - `TestLoadHooks`：完整配置 / 自动 id / 空输入 / 各类非法配置错误
  - `TestHookEngine`：find_matching_hooks / condition 过滤 / once 过滤 / reject / 非 reject / prompt 消息收集 / 错误不抛 / async 不阻塞
  - `TestAgentHookIntegration`：mock LLM 触发 `rm -rf` 被 reject 后 Agent 收到 `Hook rejected: ...` 错误结果
- 实际产出: `tests/test_hooks.py:1-510`（13 个测试类）

## T10: 接入主流程 —— config 绑定

- 影响文件: `mewcode/config.py`
- 依赖任务: T1
- 完成标准: `AppConfig` dataclass 新增 `raw_hooks: list[dict]` 字段（保留原始 dict，由 loader 二次解析）；`load_config` 在 `validate_config_structure` 后填入 `raw_hooks=validated["hooks"]`。
- 实际产出: `mewcode/config.py:93`（字段定义）、`mewcode/config.py:152`（赋值）

## T11: 接入主流程 —— Agent + App 装配 + Agent loop 触发

- 影响文件: `mewcode/__main__.py`、`mewcode/app.py`、`mewcode/agent.py`
- 依赖任务: T2-T8, T10
- 完成标准:
  - 入口：`mewcode/__main__.py:40-45` 调 `load_hooks(config.raw_hooks)`，`HookConfigError` 时打 stderr + `sys.exit(1)`，否则 `HookEngine(hooks)` 传给 `MewCodeApp`
  - App 装配：`mewcode/app.py:515 / 526` 接收 `hook_engine` 参数并持有；`mewcode/app.py:658` 把 `hook_engine` 透传给 `Agent`
  - 生命周期：`mewcode/app.py:803-808`（startup）/`mewcode/app.py:1581-1587`（shutdown）派发对应事件
  - Agent 字段：`mewcode/agent.py:296 / 314` 接收 `hook_engine` 参数；`mewcode/agent.py:371-382` 实现 `_build_hook_context`；`mewcode/agent.py:384-394` 实现 `_drain_hook_events`
  - Agent loop 触发点：
    - session_start：`mewcode/agent.py:407-410`
    - turn_start：`mewcode/agent.py:427-430`
    - pre_send：`mewcode/agent.py:460-463` + `get_prompt_messages` 注入下一轮
    - post_receive：`mewcode/agent.py:509-512`
    - pre_tool_use：`mewcode/agent.py:625-636`（reject 时把 `Hook rejected: {reason}` 包成 `ToolResult` 并 yield `ToolResultEvent(is_error=True)`）
    - post_tool_use：`mewcode/agent.py:674-682`
    - turn_end + session_end：`mewcode/agent.py:569-573`
- 实际产出: 同上

## T12: 端到端验证

- 影响文件: 无
- 依赖任务: T11
- 完成标准: 在 `config.yaml` 配 `hooks: [{event: pre_tool_use, if: 'tool == "Bash" && args.command =~ /rm\s+-rf/', action: {type: prompt, message: "blocked"}, reject: true}]`，启动 `python -m mewcode` 让 LLM 调 Bash + `rm -rf` 看到工具结果是 `Hook rejected: blocked`；HTTP hook / async hook / once hook 路径由 `tests/test_hooks.py` 中对应测试覆盖。
- 实际产出: 由 `TestAgentHookIntegration.test_pre_tool_use_reject_skips_tool` + `TestHookEngine` 系列覆盖；配置文件手测见 checklist §4。

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
- [ ] T10
- [ ] T11
- [ ] T12
