# ch09: 记忆系统 Checklist

> 所有条目必须可勾选、可观测。验收方式写在每项后面的括号里。

## 1. 实现完整性

### 项目指令（mewcode/memory/instructions.py）

- [ ] 模块常量 `MAX_INCLUDE_DEPTH = 5` 在 `/Users/codemelo/mewcode/mewcode/memory/instructions.py:5` 定义。
- [ ] 模块常量 `INCLUDE_PREFIX = "@include "` 在 `instructions.py:6` 定义。
- [ ] 函数 `process_includes(content, base_dir, project_root, depth=0)` 在 `instructions.py:9-46` 实现：逐行扫描，命中前缀的行剥相对路径并 `(base_dir / rel_path).resolve()`，越界落 `<!-- @include blocked: path outside project -->`，文件不存在落 `<!-- @include skipped: file not found -->`，命中文件递归 `process_includes(..., depth+1)`；`depth >= MAX_INCLUDE_DEPTH` 直接 return 原 content。
- [ ] 函数 `load_instructions(project_root)` 在 `instructions.py:48-66` 实现：三层优先级（`<root>/MEWCODE.md` → `<root>/.mewcode/MEWCODE.md` → `~/.mewcode/MEWCODE.md`），每层跑 `process_includes`，多段用 `\n---\n` 拼接，无文件返回 `""`。
- [ ] 边界处理：越界 include 不抛异常（`abs_path.relative_to(resolved_root)` 在 `ValueError` 分支落注释行），测试 `TestProcessIncludes.test_path_outside_project_blocked` 验证。

### 会话存档（mewcode/memory/session.py）

- [ ] 模块常量 `SESSIONS_DIR = ".mewcode/sessions"` 在 `session.py:14` 定义。
- [ ] 模块常量 `TIME_GAP_THRESHOLD = timedelta(hours=24)` 在 `session.py:15` 定义。
- [ ] 模块常量 `DEFAULT_MAX_AGE_DAYS = 30` 在 `session.py:16` 定义。
- [ ] 模块常量 `TITLE_MAX_LENGTH = 50` 在 `session.py:17` 定义。
- [ ] 枚举 `RecordType(str, Enum)` 在 `session.py:30-35` 定义 5 个值（`system_prompt / user / assistant / tool_result / compression`）。
- [ ] 类 `SessionRecord` dataclass 在 `session.py:38-45` 定义：`type / content / timestamp / tool_use_id / is_error`。
- [ ] 方法 `SessionRecord.to_jsonl()` 在 `session.py:46-56` 实现：`ensure_ascii=False`、可选 `tool_use_id`、仅 tool_result 写 `is_error`。
- [ ] 方法 `SessionRecord.from_jsonl(line)` 在 `session.py:58-70` 实现：`json.JSONDecodeError / KeyError / ValueError` 三类异常都返回 None。
- [ ] 方法 `SessionRecord.from_message(message)` 在 `session.py:72-119` 实现：tool_results 拆多条；assistant + tool_uses 把 text 与 tool_use blocks 内联到 content list；plain user / assistant 走单条。
- [ ] 函数 `records_to_messages(records)` 在 `session.py:122-196` 实现：`pending_tool_results` 队列冲洗到 user message；system_prompt 跳过；compression 渲染 `[摘要]\n<content>`；assistant content list 拆 text 与 tool_uses。
- [ ] 函数 `validate_message_chain(records)` 在 `session.py:199-222` 实现：维护 `pending_tool_uses set`，集合为空时记录前缀长度，最后返回最大完整前缀。
- [ ] 类 `SessionMeta` dataclass 在 `session.py:225-233` 定义 7 字段；`save / load` 在 `session.py:235-266` 实现。
- [ ] 类 `Session` 在 `session.py:271-302` 定义：`append` 内含 `from_message + write + flush + meta` 同步更新逻辑；首条 user 消息截 `TITLE_MAX_LENGTH=50` 写 `meta.title`。
- [ ] 类 `ResumeResult` dataclass 在 `session.py:309-313` 定义（`session / messages / last_active`）。
- [ ] 函数 `_generate_session_id()` 在 `session.py:384-387` 实现：`session_<YYYYMMDD_HHMMSS>_<4 char a-z0-9>`。
- [ ] 类 `SessionManager` 在 `session.py:390-482` 定义：`__init__ / create / list / resume / delete / cleanup` 全部到位；`resume` 内部跑 `validate_message_chain + records_to_messages` 链路。
- [ ] 函数 `build_time_gap_message(last_active)` 在 `session.py:358-380` 实现：`<24h` 返 None；`>=48h` 表达「N 天」否则「N 小时」；含 `代码可能有变更` 文案。

### 自动记忆（mewcode/memory/auto_memory.py）

- [ ] 模块常量 `USER_MEMORIES_RELPATH = ".mewcode/memories.md"` 在 `auto_memory.py:8` 定义。
- [ ] 模块常量 `PROJECT_MEMORIES_RELPATH = ".mewcode/memories.md"` 在 `auto_memory.py:9` 定义。
- [ ] 常量 `MEMORY_EXTRACTION_PROMPT` 在 `auto_memory.py:11-37` 定义：含「用户偏好 / 纠正反馈 / 项目知识 / 参考资料」四类标题、「不要重复添加」、「没有值得记忆的内容，该分类下留空（不要写任何条目，不要写占位符）」、「不要调用任何工具」。
- [ ] 常量 `_USER_LEVEL_HEADERS = {"用户偏好", "纠正反馈"}` 与 `_PROJECT_LEVEL_HEADERS = {"项目知识", "参考资料"}` 在 `auto_memory.py:39-40` 定义。
- [ ] 类 `MemoryManager` 在 `auto_memory.py:43-72` 定义；`user_path / project_path` property 暴露。
- [ ] 方法 `MemoryManager.load()` 在 `auto_memory.py:57-70` 实现：两路径若存在且 strip 非空则收集，`\n\n` 拼接。
- [ ] 方法 `MemoryManager.extract(client, conversation, protocol)` 在 `auto_memory.py:72-126` 实现：取 `history[self._last_extraction_msg_count:]` 增量、构造 prompt、跑 `client.stream` 收 `TextDelta.text`、异常静默 return、成功时推 cursor 并调 `_write_memories`。
- [ ] 方法 `MemoryManager._write_memories(content)` 在 `auto_memory.py:128-161` 实现：按 `### ` 切段、`_assign_section` 分流、写入路径前 `mkdir(parents=True, exist_ok=True)`、`write_text(strip + "\n", utf-8)`。
- [ ] 静态方法 `MemoryManager._is_placeholder(line)` 在 `auto_memory.py:163-166` 实现：剥 `- ` 与空白后命中 `{"", "...", "…", "无", "暂无", "N/A"}` 返回 True。
- [ ] 静态方法 `MemoryManager._assign_section(...)` 在 `auto_memory.py:168-189` 实现：过滤出 `- ` 开头非占位行；按 header 关键字归入 user / project sections。
- [ ] 方法 `MemoryManager.clear()` 在 `auto_memory.py:191-195` 实现：两路径若存在 `write_text("")` 截断（不删文件）。
- [ ] 方法 `MemoryManager.get_display_text()` 在 `auto_memory.py:197-213` 实现：两路径分别按 `[用户级] <path>\n<content>` / `[项目级] ...` 渲染、`\n\n` 拼接；都空返回 `"当前没有任何自动记忆。"`。

### 包级 re-export（mewcode/memory/__init__.py）

- [ ] `/Users/codemelo/mewcode/mewcode/memory/__init__.py` 集中 `from mewcode.memory.auto_memory import MemoryManager` / `from mewcode.memory.instructions import load_instructions, process_includes` / `from mewcode.memory.session import (ResumeResult, Session, SessionManager, SessionMeta, SessionRecord, build_time_gap_message, generate_session_summary, validate_message_chain)`。
- [ ] `__all__` 与导入名一一对应，无遗漏。

### 对话注入（mewcode/conversation.py）

- [ ] 字段 `ltm_injected: bool = field(default=False, init=False)` 在 `conversation.py:41` 定义。
- [ ] 方法 `inject_long_term_memory(instructions, memories)` 在 `conversation.py:80-107` 实现：已注入 return；按 `env_injected` 计算 pos；instructions 非空 insert `## 项目指令\n<content>`；memories 非空 insert `## 自动记忆\n<content>`；二者至少一个非空时追加 assistant `"好的，我已了解项目背景和记忆。"` 并置 `ltm_injected=True`。
- [ ] 方法 `replace_history(new_messages)` 在 `conversation.py:109-112` 实现：重置 `env_injected=False / ltm_injected=False`。

### Agent 钩入（mewcode/agent.py）

- [ ] 模块常量 `MEMORY_EXTRACTION_INTERVAL = 5` 在 `agent.py:49` 定义。
- [ ] `Agent.__init__` 在 `agent.py:295-329` 接收 `instructions_content: str = ""` / `memory_manager: MemoryManager | None = None`；自存 `self._loop_count = 0` / `self._extracting = False`。
- [ ] `Agent.run` 入口在 `agent.py:399-405` 调 `inject_environment` 之后立即调 `conversation.inject_long_term_memory(self.instructions_content, memory_content)`。
- [ ] loop 收尾分支在 `agent.py:564-568` 触发 `asyncio.ensure_future(self._extract_memories(conversation))`（当 `self._loop_count % MEMORY_EXTRACTION_INTERVAL == 0 and self.memory_manager`）。
- [ ] `_extract_memories(conversation)` 在 `agent.py:870-883` 实现：`self._extracting` 互斥；try-except 包 `memory_manager.extract`，except `log.debug` 不传播；finally 复位 `self._extracting=False`。
- [ ] `manual_compact` 在 `agent.py:902-904` 与 `run_to_completion` 在 `agent.py:924-926` 同样调 `inject_long_term_memory` 完成二次注入。

### 命令处理（mewcode/commands/handlers/）

- [ ] `/memory` 命令处理在 `mewcode/commands/handlers/memory.py:6-46` 实现：`memory_manager is None` 时提示「记忆管理器未初始化」；空 / `list` 子命令打印 `get_display_text`；`clear` 调用 `MemoryManager.clear` 并提示「所有自动记忆已清空。」；`edit` 打印 user/project 路径；未知子命令打印 usage。
- [ ] `MEMORY_COMMAND = Command(name="memory", description="记忆管理", usage="/memory [list | clear | edit]", type=CommandType.LOCAL, handler=handle_memory)` 在 `memory.py:40-46` 定义。
- [ ] `/session` 命令处理在 `mewcode/commands/handlers/session.py` 实现：list / resume / new / delete 子命令；resume 后追加 `build_time_gap_message` 返回值；resume 切换 session/conversation 后复位 `ctx.agent._loop_count = 0`。

## 2. 接入完整性（必查，杜绝死代码）

- [ ] `grep -rn "from mewcode.memory" /Users/codemelo/mewcode/mewcode --include="*.py"` 至少 5 处非测试调用方：
  - `mewcode/app.py:51-58` (导入 `MemoryManager / Session / SessionManager / SessionMeta / generate_session_summary / load_instructions`)
  - `mewcode/agent.py:24` (`from mewcode.memory.auto_memory import MemoryManager`)
  - `mewcode/commands/handlers/memory.py` (经 ctx 调用，无直接 import 不强求)
  - `mewcode/commands/handlers/session.py` (`from mewcode.memory.session import build_time_gap_message`)
- [ ] `grep -rn "MemoryManager\b" /Users/codemelo/mewcode/mewcode --include="*.py" | grep -v "__init__.py" | grep -v "_test\|/tests/"` 至少 4 处（`app.py / agent.py / commands/handlers/memory.py / memory/auto_memory.py`）。
- [ ] `grep -rn "SessionManager\b" /Users/codemelo/mewcode/mewcode --include="*.py" | grep -v "__init__.py" | grep -v "/tests/"` 至少 3 处（`app.py / commands/handlers/session.py / memory/session.py`）。
- [ ] `grep -rn "inject_long_term_memory" /Users/codemelo/mewcode/mewcode --include="*.py" | grep -v "_test\|/tests/"` 至少 4 处（`conversation.py` 定义 + `agent.py` 3 处调用：run 入口 / 压缩后 / run_to_completion）。
- [ ] `grep -rn "MEMORY_EXTRACTION_INTERVAL\|_extract_memories\|_loop_count" /Users/codemelo/mewcode/mewcode --include="*.py" | grep -v "/tests/"` 命中 `agent.py` 主逻辑。
- [ ] `grep -rn "load_instructions" /Users/codemelo/mewcode/mewcode --include="*.py" | grep -v "/tests/"` 命中 `app.py:634` 与 `memory/instructions.py / memory/__init__.py`。
- [ ] `grep -rn "build_time_gap_message" /Users/codemelo/mewcode/mewcode --include="*.py" | grep -v "/tests/"` 命中 `commands/handlers/session.py` 与定义点。
- [ ] 用户输入到本模块的路径可一句话描述：
  - 项目指令：App 启动 → `load_instructions(work_dir)` → `Agent(instructions_content=...)` → `Agent.run` 入口 → `conversation.inject_long_term_memory(instructions, memory_content)` 注入 `## 项目指令` block。
  - 自动记忆：Agent loop 收尾 → `_loop_count += 1` → 模 5 → `asyncio.ensure_future(self._extract_memories)` → `MemoryManager.extract` → `client.stream` 跑 prompt → `_write_memories` 分流写两路 memories.md → 下轮 `Agent.run` 重新 `memory_manager.load()` 注入。
  - 会话存档：用户消息 → `App` → `self.session.append(Message)` → `SessionRecord.from_message` 拆条 → 写 jsonl + flush + meta 同步；`/session resume <id>` → `SessionManager.resume` → `validate_message_chain` 截断 → `records_to_messages` 重建 → `replace_history` → 追加 time gap message。
  - 命令链：`/memory list/clear/edit` → `handle_memory(ctx)` → `MemoryManager.{get_display_text, clear}`；`/session list/resume/new/delete` → `handle_session(ctx)` → `SessionManager.{list, resume, create, delete}`。

## 3. 编译与测试

- [ ] `ruff check mewcode/memory/ mewcode/agent.py mewcode/conversation.py mewcode/commands/handlers/memory.py mewcode/commands/handlers/session.py` 无错误。
- [ ] `pytest tests/test_memory.py -v` 全部通过，覆盖：
  - `TestProcessIncludes`（no_includes / basic / recursive / depth_limit / path_outside_blocked / file_not_found）
  - `TestLoadInstructions`（single_layer / multi_layer_priority / no_files_returns_empty）
  - `TestSessionRecord`（user_roundtrip / assistant_with_tool_uses / tool_results_multiple_records / malformed_jsonl / plain_assistant）
  - `TestSession`（append_writes_jsonl_and_updates_meta / title_set_from_first_user_message）
  - `TestSessionManager`（create_and_list / delete / cleanup_removes_old_sessions / create_generates_valid_id）
  - `TestValidateMessageChain`（complete_chain / truncate_at_missing_tool_result / empty_records）
  - `TestRecordsToMessages`（basic_roundtrip / tool_result_grouping / system_prompt_skipped）
  - `TestSessionResume`（restores_messages / nonexistent_returns_none / truncates_incomplete_chain）
  - `TestTimeGapMessage`（no_gap_returns_none / gap_returns_message）
  - `TestSessionMeta`（save_and_load / load_invalid_returns_none）
  - `TestMemoryManager`（load_empty / load_merges_user_and_project / clear / get_display_text_empty / write_memories_splits_correctly）
  - `TestConversationInjection`（inject_long_term_memory / inject_idempotent / inject_instructions_only / inject_memories_only / inject_nothing / replace_history_resets_ltm）
  - `TestMemoryExtraction`（extraction_prompt_contains_categories）
- [ ] `python -c "from mewcode.memory import MemoryManager, load_instructions, process_includes, Session, SessionManager, SessionMeta, SessionRecord, ResumeResult, build_time_gap_message, generate_session_summary, validate_message_chain; print('ok')"` 打印 `ok`。

## 4. 端到端验证

- [ ] 项目根放一份 `MEWCODE.md`，TUI 启动后通过 `--debug` 日志或临时打印 `conversation.history[0:3]` 能看到 `## 项目指令\n...` 段。
- [ ] `MEWCODE.md` 内写 `@include ./sub/style.md`（sub/style.md 在项目内）能展开；写 `@include ../../etc/passwd` 显示为 `<!-- @include blocked: path outside project -->`；写 `@include ./nonexistent.md` 显示为 `<!-- @include skipped: file not found -->`。
- [ ] TUI 收发若干消息后退出（Ctrl-D），`.mewcode/sessions/` 出现一对 `session_*.jsonl` + `session_*.meta`；jsonl 每行为合法 JSON、含 `type/content/timestamp`。
- [ ] 重启 TUI，运行 `/session list` 看到该会话；`/session resume <id>` 后 `conversation.history` 含历史消息，UI 重新渲染过去对话。
- [ ] 把某 session 的 `meta.last_active` 手动改成 25 小时前再 `/session resume <id>`，对话末尾出现「[系统提示] 距离上次会话已过去 25 小时。代码可能有变更，建议在操作前重新读取相关文件。」
- [ ] 把 `last_active` 改成 31 天前，启动时 `self.session_manager.cleanup()` 应当自动删除该会话；`/session list` 不再列出。
- [ ] 跟 Agent 对话 5 轮（每轮包含一次 final assistant 无 tool_call），观察 `~/.mewcode/memories.md` 或项目级 `.mewcode/memories.md` 出现新行；下次启动后 `conversation.history` 的 `## 自动记忆` block 内包含该内容。
- [ ] `/memory list` 显示 `[用户级] <path>\n...` / `[项目级] ...`；`/memory clear` 后再 `/memory list` 显示「当前没有任何自动记忆。」。
- [ ] 主动构造一个含未结束 tool_use 的 session（手动改 jsonl 删掉对应 tool_result 行），`/session resume` 后 `validate_message_chain` 把 tool_use 截断，恢复出的对话不含未匹配的 tool_use（防止 API 400 invalid_request_error）。

## 5. 文档

- [ ] spec.md / tasks.md / checklist.md 三件套齐全且最新（位于 `/Users/codemelo/mewcode/docs/python/ch09/`）。
- [ ] commit 信息标注 `ch09 (python)` 与三件套关闭状态。
