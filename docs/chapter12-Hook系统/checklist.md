# ch12: Hook 系统 Checklist

## 1. 实现完整性

- [ ] 15 个生命周期事件常量在 `mewcode/hooks/events.py:6-30`：session_start / session_end / turn_start / turn_end / pre_tool_use / post_tool_use / pre_send / post_receive / startup / shutdown / error / compact / permission_request / file_change / command_execute；`len(LifecycleEvent) == 15`
- [ ] 4 个动作类型在 `mewcode/hooks/loader.py:8` 的 `_VALID_ACTION_TYPES` 集合：command / prompt / http / agent
- [ ] 数据结构 `Action / ActionResult / Hook / HookContext / ToolRejectedError` 在 `mewcode/hooks/models.py:9-85`，字段齐全
- [ ] `Condition / ConditionGroup / parse_condition / _parse_single` 在 `mewcode/hooks/conditions.py:12-96`；leaf 操作符 `==/!=/=~/~=` 全部实现
- [ ] `_OPERATORS = ("==", "!=", "=~", "~=")` 常量在 `mewcode/hooks/conditions.py:54`
- [ ] 同一表达式混用 `&&` 和 `||` 时 `parse_condition` 抛 `ConditionParseError("Cannot mix '&&' and '||' in a single condition expression")`（`mewcode/hooks/conditions.py:79-83`）
- [ ] `HookEngine / HookNotification` 在 `mewcode/hooks/engine.py:14-110`；`__init__` 初始化 `hooks` / `_prompt_messages` / `_notifications` 三个状态
- [ ] `find_matching_hooks` 三层过滤（event / should_run / condition）在 `mewcode/hooks/engine.py:31-41`
- [ ] `run_hooks` 中 `async_exec=True` 走 `asyncio.ensure_future(self._run_single(hook, ctx))` 不 await（`mewcode/hooks/engine.py:43-50`）
- [ ] `_run_single` 把 `prompt` 类型成功结果写入 `_prompt_messages`，所有结果写 `_notifications`，异常被 catch（`mewcode/hooks/engine.py:52-78`）
- [ ] `run_pre_tool_hooks` 遇到 `hook.reject=True` 即返回 `ToolRejectedError`（`mewcode/hooks/engine.py:96-102`）
- [ ] `get_prompt_messages()` 一次性取出并清空（`mewcode/hooks/engine.py:105-108`）
- [ ] `drain_notifications()` 一次性取出并清空（`mewcode/hooks/engine.py:110-113`）
- [ ] `execute_command` 在 `mewcode/hooks/executors.py:13-35`：`asyncio.create_subprocess_shell` + `stderr=STDOUT` 合并，超时时 `proc.kill()` + `await proc.wait()`，output 含 "timed out"
- [ ] `execute_prompt` 在 `mewcode/hooks/executors.py:38-40` 仅做模板替换
- [ ] `execute_http` 在 `mewcode/hooks/executors.py:43-72`：默认 POST，`urlopen(req, timeout=30)`，body 非空时自动加 `Content-Type: application/json`，响应体截断 500 字符
- [ ] `execute_http` 通过 `loop.run_in_executor(None, _do_request)` 把同步 urlopen 放到默认线程池
- [ ] `execute_agent` 在 `mewcode/hooks/executors.py:75-81` 是 stub，返回 "agent executor not yet implemented"
- [ ] `_EXECUTOR_MAP` + `execute_action` 派发在 `mewcode/hooks/executors.py:84-97`
- [ ] `load_hooks` 实现完整校验链路在 `mewcode/hooks/loader.py:25-115`：event 白名单、action.type 白名单、必填字段、reject/async 与 event 的约束、timeout 正整数、自动 id、condition 解析失败包错
- [ ] `Hook.should_run()` 在 `mewcode/hooks/models.py:43-46` 检查 once + executed，配合 `mark_executed()` 实现单次触发

## 2. 接入完整性

- [ ] `grep -rn "HookEngine(" /Users/codemelo/mewcode --include="*.py"` 命中 `mewcode/__main__.py:45` 一个非测试调用方
- [ ] `grep -rn "run_pre_tool_hooks\|run_hooks(" /Users/codemelo/mewcode --include="*.py" | grep -v test` 至少命中 `mewcode/agent.py` 7 处（session_start / turn_start / pre_send / post_receive / pre_tool_use / post_tool_use / turn_end+session_end）+ `mewcode/app.py` 2 处（startup / shutdown）
- [ ] Config 绑定：`mewcode/config.py:93` 含 `raw_hooks: list[dict] = field(default_factory=list)` 字段；`mewcode/config.py:152` 在 `load_config` 中填 `raw_hooks=validated["hooks"]`
- [ ] Agent 字段：`mewcode/agent.py:296` 构造参数含 `hook_engine: HookEngine | None = None`；`mewcode/agent.py:314` 赋值 `self.hook_engine = hook_engine`
- [ ] App 装配：`mewcode/app.py:515` 构造参数含 `hook_engine`；`mewcode/app.py:526` 赋值；`mewcode/app.py:658` 透传给 Agent
- [ ] 入口路径：`config.yaml` `hooks` → `config.raw_hooks` → `__main__.py:load_hooks` → `HookEngine(hooks)` → `MewCodeApp(hook_engine=...)` → `Agent(hook_engine=...)` → `agent.run` 内 emit
- [ ] 工具调用前 `pre_tool_use` 在 `mewcode/agent.py:625-636`，reject 时打包成 `ToolResult(output=f"Hook rejected: {rejection.reason}", is_error=True)` 并 yield `ToolResultEvent(is_error=True)`，`continue` 跳过实际执行
- [ ] 工具调用后 `post_tool_use` 在 `mewcode/agent.py:674-682`
- [ ] startup 事件在 `mewcode/app.py:803-808` 通过 `asyncio.ensure_future` 派发；shutdown 事件在 `mewcode/app.py:1581-1587` await 派发
- [ ] `_build_hook_context` 在 `mewcode/agent.py:371-382` 统一构造 `HookContext`
- [ ] `_drain_hook_events` 在 `mewcode/agent.py:384-394` 把 `HookNotification` 转成 `HookEvent` 流给 TUI 展示
- [ ] `pre_send` 钩子注入：`mewcode/agent.py:466-468` 调 `get_prompt_messages()` 把 prompt 类型 hook 输出注入下一轮 LLM 请求
- [ ] 非法 hook 配置启动时打 stderr 而非 crash：`mewcode/__main__.py:40-43` 捕获 `HookConfigError` 并 `sys.exit(1)`

## 3. 编译与测试

- [ ] `cd /Users/codemelo/mewcode && ruff check mewcode/hooks/ tests/test_hooks.py` 通过
- [ ] `cd /Users/codemelo/mewcode && pytest tests/test_hooks.py -v` 全部测试通过：覆盖 `TestLifecycleEvent` / `TestHookContext` / `TestParseCondition` / `TestConditionEvaluate` / `TestConditionGroupEvaluate` / `TestCommandExecutor` / `TestPromptExecutor` / `TestHttpExecutor` / `TestAgentExecutor` / `TestExecuteAction` / `TestLoadHooks` / `TestHookEngine` / `TestAgentHookIntegration` 共 13 个测试类
- [ ] `cd /Users/codemelo/mewcode && python -c "from mewcode.hooks import HookEngine, load_hooks, LifecycleEvent; print(len(LifecycleEvent))"` 输出 `15`
- [ ] `cd /Users/codemelo/mewcode && python -c "from mewcode.agent import Agent; from mewcode.app import MewCodeApp"` 无 ImportError，确认 hooks 包被 agent/app 正确 import

## 4. 端到端验证

- [ ] 在 `config.yaml` 配 pre_tool_use reject hook（例如 `tool == "Bash" && args.command =~ /rm\s+-rf/`），`python -m mewcode` 启动 TUI 让 LLM 触发匹配的 Bash 命令，工具结果是 `Hook rejected: <message>`（路径 `mewcode/agent.py:625-636`）
- [ ] HTTP hook 由 `tests/test_hooks.py` 中 `TestHttpExecutor.test_mock_request` 用 `unittest.mock.patch` mock `urlopen` 验证 status 200 + 响应体
- [ ] async hook 由 `tests/test_hooks.py` 中 `TestHookEngine.test_async_hook_does_not_block` 验证 `sleep 5` 不阻塞主协程返回
- [ ] once hook 由 `tests/test_hooks.py` 中 `TestHookEngine.test_once_filter` 验证 `mark_executed()` 后 `find_matching_hooks` 返回空
- [ ] reject 端到端由 `tests/test_hooks.py` 中 `TestAgentHookIntegration.test_pre_tool_use_reject_skips_tool` 验证 mock LLM 调 `rm -rf /` 时 Agent 拿到的 `ToolResultEvent.is_error == True` 且 output 含 `Hook rejected`
- [ ] command 超时端到端：`tests/test_hooks.py` 中 `TestCommandExecutor.test_timeout` 验证 `sleep 10` + `timeout=1` 时 `success == False` 且 output 含 "timed out"
- [ ] 加载校验端到端：在临时 `config.yaml` 配一个非法 hook（如 `event: pre_tool_use, action: {type: command}` 缺 command），`python -m mewcode` 启动时 stderr 看到 `Hook config error: hook #1: action type 'command' requires 'command' field` 形式的错误并退出码 1（`mewcode/__main__.py:40-43`）
- [ ] 留存证据：未提供 TUI 截图（手动验证不在课程验收流程要求范围内）

## 5. 文档

- [ ] `docs/python/ch12/spec.md` 存在
- [ ] `docs/python/ch12/tasks.md` 存在
- [ ] `docs/python/ch12/checklist.md` 存在
- [ ] commit 已落地到 Python 分支 hooks 子系统；建议下一次三件套关闭 commit 使用形如 `docs(ch12-python): close spec/tasks/checklist for hooks system` 的消息
