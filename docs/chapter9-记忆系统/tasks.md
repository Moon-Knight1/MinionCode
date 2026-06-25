# ch09: 记忆系统 Tasks

> 任务粒度: 每个任务可在一次会话内完成，可独立交付。所有 T 任务完成后逐项勾上，每条任务记录实际落地的文件与行号。

## T1: 项目指令 `@include` 递归展开

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/instructions.py:9-46`（`process_includes`）
- 依赖任务: 无
- 完成标准:
  - `MAX_INCLUDE_DEPTH=5` 与 `INCLUDE_PREFIX="@include "` 常量到位
  - 逐行扫描 content，命中前缀的行剥出相对路径，按 `(base_dir / rel_path).resolve()` 解析
  - 解析后用 `abs_path.relative_to(resolved_root)` 判断是否在 project_root 内，越界落 `<!-- @include blocked: path outside project -->`
  - 文件不存在 / 非 file 落 `<!-- @include skipped: file not found -->`
  - 命中的文件递归 `process_includes(..., depth+1)` 后拼回
  - `depth >= MAX_INCLUDE_DEPTH` 直接 return 原 content
  - 测试 `TestProcessIncludes` 5 个用例（无 include / 基本 include / 递归 include / depth 限制 / path 越界 / 文件不存在）全部命中

## T2: 项目指令三层加载

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/instructions.py:48-66`（`load_instructions`）
- 依赖任务: T1
- 完成标准:
  - 三层优先级顺序：`<root>/MEWCODE.md` → `<root>/.mewcode/MEWCODE.md` → `~/.mewcode/MEWCODE.md`
  - 每层文件存在且 is_file 时读取并对其内容跑 `process_includes(content, path.parent, root)`
  - 多段用 `\n---\n` join
  - 无任何文件存在时返回 `""`
  - 测试 `TestLoadInstructions`（`test_single_layer / test_multi_layer_priority / test_no_files_returns_empty`）通过

## T3: SessionRecord 序列化

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/session.py:30-119`（`RecordType / SessionRecord`）
- 依赖任务: 无
- 完成标准:
  - `RecordType(str, Enum)` 5 个值：`system_prompt / user / assistant / tool_result / compression`
  - `SessionRecord` dataclass 字段：`type / content / timestamp / tool_use_id / is_error`
  - `to_jsonl()`：序列化 `{type, content, timestamp}`，可选 `tool_use_id`，仅 tool_result 写 `is_error`，`ensure_ascii=False`
  - `from_jsonl(line)`：异常返回 None；未知 RecordType 也返回 None
  - `from_message(message)`：tool_results 拆多条 tool_result 记录；assistant + tool_uses 内联到 content blocks (`[{type:text}, {type:tool_use,id,name,input}]`)；plain user / assistant 走单条普通记录
  - 测试 `TestSessionRecord` 5 个用例（user roundtrip / assistant with tool_uses / tool_results multiple records / malformed jsonl / plain assistant）通过

## T4: 记录 ↔ 消息互转 + 链路校验

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/session.py:122-222`（`records_to_messages / validate_message_chain`）
- 依赖任务: T3
- 完成标准:
  - `records_to_messages`：维护 `pending_tool_results` 队列，遇到非 tool_result 记录前先把队列冲到一条 user message 的 `tool_results`；system_prompt 跳过；compression 渲染为 `[摘要]\n<content>` 的 user message；assistant content list 时拆 text + tool_uses
  - `validate_message_chain`：维护 `pending_tool_uses set`，assistant content list 里 tool_use block 的 id 进集合，tool_result 出集合；集合为空时记录前缀长度，最后返回最大完整前缀
  - 测试 `TestRecordsToMessages`（3 个）+ `TestValidateMessageChain`（3 个）全部通过

## T5: SessionMeta 落盘 + Session 句柄

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/session.py:225-307`（`SessionMeta / Session / ResumeResult`）
- 依赖任务: T3
- 完成标准:
  - `SessionMeta` dataclass 7 字段（id / title / summary / message_count / total_tokens / created_at / last_active），`created_at / last_active` 默认 `datetime.now(timezone.utc)`
  - `SessionMeta.save(path)`：JSON 落盘，含 `isoformat` 时间字段
  - `SessionMeta.load(path)`：异常返回 None
  - `Session.__init__(session_id, file, meta, sessions_dir)` 持文件句柄
  - `Session.append(message)`：调 `SessionRecord.from_message` 拆条逐条 `to_jsonl + "\n"` 写入并 `flush`；`meta.message_count += 1`；`meta.last_active = now`；首次遇到 user content 时截 `TITLE_MAX_LENGTH=50` 写入 `meta.title`；每次 append 后 `meta.save` 覆盖 `.meta` 文件
  - `Session.close()`：判空 + `flush + close`
  - `ResumeResult` dataclass：`session / messages / last_active`
  - 测试 `TestSession`（2 个：append 写 jsonl + title 设置）通过

## T6: SessionManager 生命周期

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/session.py:384-482`（`_generate_session_id / SessionManager`）
- 依赖任务: T4, T5
- 完成标准:
  - `_generate_session_id()`：`session_<YYYYMMDD_HHMMSS>_<4 字符 a-z0-9>` 格式
  - `SessionManager.__init__(work_dir)`：构造 `<work_dir>/.mewcode/sessions/` 并 `mkdir(parents=True, exist_ok=True)`
  - `create()`：新 ID + 写 `.meta` + 打开 jsonl `mode="a"` + 返回 `Session`
  - `list()`：扫 `*.meta`、`SessionMeta.load` 反序列化、按 `last_active` 倒序
  - `resume(id)`：jsonl 缺失返回 None；逐行 `from_jsonl` 跳空跳错；`validate_message_chain` 截断；`records_to_messages` 重建；重新打开 jsonl `mode="a"` 续写
  - `delete(id)`：删 jsonl + .meta，任一存在即返回 True
  - `cleanup(max_age_days=30)`：迭代 `.meta`、`last_active < cutoff` 调 `delete` 并计数
  - 测试 `TestSessionManager`（create_and_list / delete / cleanup / generates_valid_id）+ `TestSessionResume`（restores_messages / nonexistent / truncates_incomplete_chain）通过

## T7: 断会话时长提示

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/session.py:358-380`（`build_time_gap_message`）+ `TIME_GAP_THRESHOLD` 常量
- 依赖任务: 无
- 完成标准:
  - `TIME_GAP_THRESHOLD=timedelta(hours=24)` 常量
  - 距 `last_active < 24h` 返回 None
  - `gap.total_seconds() // 3600 >= 48` 表达为「N 天」，否则「N 小时」
  - 返回的 Message 包含 `代码可能有变更，建议在操作前重新读取相关文件。`
  - 测试 `TestTimeGapMessage`（no gap returns none / gap returns message）通过

## T8: 会话摘要生成（可选）

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/session.py:316-355`（`generate_session_summary`）+ `SESSION_SUMMARY_PROMPT`
- 依赖任务: 无
- 完成标准:
  - `SESSION_SUMMARY_PROMPT` 文本到位（要求一句话总结、不调用工具）
  - `generate_session_summary(client, conversation, protocol)`：取 `history[-10:]`；构造单独 `ConversationManager` 拼装 prompt + 最近消息 + 收尾问句；跑 `client.stream` 收 `TextDelta`；异常返回 `""`
  - 不做单独单元测试（集成在 App 的异步 summary 更新中）

## T9: MemoryManager 双路径基础

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/auto_memory.py:8-71`（常量 + `__init__ / user_path / project_path / load`）
- 依赖任务: 无
- 完成标准:
  - 常量：`USER_MEMORIES_RELPATH = ".mewcode/memories.md"` / `PROJECT_MEMORIES_RELPATH = ".mewcode/memories.md"`
  - `__init__`：算 `_user_path = Path.home() / USER_MEMORIES_RELPATH` 与 `_project_path = Path(project_root) / PROJECT_MEMORIES_RELPATH`，`_last_extraction_msg_count = 0`
  - `user_path / project_path` property 暴露
  - `load()`：两个路径若存在且非空，`strip` 后用 `\n\n` join；都空返回 `""`
  - 测试 `TestMemoryManager.test_load_empty / test_load_merges_user_and_project` 通过

## T10: MEMORY_EXTRACTION_PROMPT + extract LLM 跑提取

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/auto_memory.py:11-37, 72-127`（`MEMORY_EXTRACTION_PROMPT / extract`）
- 依赖任务: T9
- 完成标准:
  - prompt 含 4 类分类标题（用户偏好 / 纠正反馈 / 项目知识 / 参考资料）、`不要重复添加` / `不要写任何条目，不要写占位符` / `不要调用任何工具` 等关键 marker
  - `extract(client, conversation, protocol)`：
    - 从 `conversation.history[self._last_extraction_msg_count:]` 取增量
    - 把 user/assistant 文本拼成 `"用户: ..."` / `"助手: ..."` 行
    - 拼装 prompt 含 `## 当前 memories.md\n<当前内容 or (空)>` + `## 最近对话\n...`
    - 构造独立 `ConversationManager`、`history = [Message(role="user", content=prompt)]`
    - 跑 `client.stream(extract_conv, system="你是一个记忆提取助手。")`，收集 `TextDelta.text`
    - 异常静默 `return`
    - 成功后更新 `self._last_extraction_msg_count = len(conversation.history)`，把 collected 转给 `_write_memories`
  - 测试 `TestMemoryExtraction.test_extraction_prompt_contains_categories` 通过

## T11: `_write_memories` 分流 + 占位过滤

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/auto_memory.py:39-40, 128-190`（`_USER_LEVEL_HEADERS / _PROJECT_LEVEL_HEADERS / _write_memories / _is_placeholder / _assign_section`）
- 依赖任务: T10
- 完成标准:
  - `_USER_LEVEL_HEADERS = {"用户偏好", "纠正反馈"}` / `_PROJECT_LEVEL_HEADERS = {"项目知识", "参考资料"}`
  - `_is_placeholder(line)`：剥 `- ` 与空白后命中 `{"", "...", "…", "无", "暂无", "N/A"}` 返回 True
  - `_assign_section(header, lines, user_sections, project_sections)`：先过滤出 `- ` 开头且非占位的 real_lines；构造 `header + "\n" + join(real_lines)`；按 header 含的关键字归入 user 或 project
  - `_write_memories(content)`：按 `### ` 切段、每段调 `_assign_section`；user/project 各自非空时 `mkdir(parents, exist_ok)` + `write_text(strip + "\n", utf-8)`
  - 测试 `TestMemoryManager.test_write_memories_splits_correctly` 通过

## T12: clear + get_display_text

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/auto_memory.py:191-213`（`clear / get_display_text`）
- 依赖任务: T9
- 完成标准:
  - `clear()`：两路径若存在 `write_text("")` 截断（不删文件）
  - `get_display_text()`：两路径分别按 `[用户级] <path>\n<content>` / `[项目级] ...` 渲染、`\n\n` 拼接；都空返回 `"当前没有任何自动记忆。"`
  - 测试 `TestMemoryManager.test_clear / test_get_display_text_empty` 通过

## T13: `mewcode/memory/__init__.py` 统一 re-export

- 影响文件: `/Users/codemelo/mewcode/mewcode/memory/__init__.py:1-26`
- 依赖任务: T1-T12
- 完成标准: 通过 `from mewcode.memory.auto_memory import MemoryManager` / `from mewcode.memory.instructions import load_instructions, process_includes` / `from mewcode.memory.session import (ResumeResult, Session, SessionManager, SessionMeta, SessionRecord, build_time_gap_message, generate_session_summary, validate_message_chain)` 集中暴露，`__all__` 列表与导入一一对应

## T14: ConversationManager 注入面板

- 影响文件: `/Users/codemelo/mewcode/mewcode/conversation.py:41, 75-113`（`ltm_injected` 字段 + `inject_environment / inject_long_term_memory / replace_history`）
- 依赖任务: 无
- 完成标准:
  - `ltm_injected` 默认 False 的 init=False 字段
  - `inject_long_term_memory(instructions, memories)`：已注入直接 return；按 `env_injected` 计算 base pos；instructions 非空时 insert `## 项目指令\n<content>`；memories 非空时 insert `## 自动记忆\n<content>`；二者至少一个非空时插入收尾 assistant `"好的，我已了解项目背景和记忆。"` 并置 `ltm_injected=True`；都空时不动 + `ltm_injected` 保持 False
  - `replace_history(new_messages)`：重置 `env_injected=False / ltm_injected=False`
  - 测试 `TestConversationInjection` 6 个用例（带 env / 幂等 / instructions only / memories only / 全空 / replace 重置）全部通过

## T15: Agent 钩入 + 自动提取

- 影响文件: `/Users/codemelo/mewcode/mewcode/agent.py:49, 295, 313-315, 404-405, 453-454, 564-568, 870-883, 902-904, 924-926`（`MEMORY_EXTRACTION_INTERVAL / __init__ / run 入口注入 / loop 收尾触发 / _extract_memories / manual_compact 二次注入 / run_to_completion 注入`）
- 依赖任务: T9, T14
- 完成标准:
  - 模块常量 `MEMORY_EXTRACTION_INTERVAL = 5`
  - `Agent.__init__` 接 `instructions_content: str = ""` / `memory_manager: MemoryManager | None = None`；自存 `self._loop_count = 0` / `self._extracting = False`
  - `Agent.run` 入口：`inject_environment` 之后立即 `memory_content = self.memory_manager.load() if self.memory_manager else ""` + `conversation.inject_long_term_memory(self.instructions_content, memory_content)`
  - loop 收尾分支 (`len(response.tool_calls)==0` → `add_assistant_message`)：`self._loop_count += 1`；若 `self._loop_count % MEMORY_EXTRACTION_INTERVAL == 0 and self.memory_manager`：`asyncio.ensure_future(self._extract_memories(conversation))`
  - `_extract_memories(conversation)` async 方法：`self._extracting` 互斥；try-except 包 `memory_manager.extract`，except `log.debug` 不传播；finally 复位 `self._extracting=False`
  - `manual_compact` / `run_to_completion` 路径在 compact 之后或新建 conversation 时同样调 `inject_long_term_memory`（与现有 `inject_environment` 配对）
  - 测试：`tests/test_memory.py` 不直接 cover Agent 触发，靠集成验证；新增 `test_loop_count_triggers_extract` 若有需要

## T16: `/memory` 命令

- 影响文件: `/Users/codemelo/mewcode/mewcode/commands/handlers/memory.py:1-46`（`handle_memory / MEMORY_COMMAND`）
- 依赖任务: T9, T12
- 完成标准:
  - `handle_memory(ctx)` 空子命令 → `get_display_text`；`list` → 同上；`clear` → `MemoryManager.clear` + 提示「所有自动记忆已清空。」；`edit` → 打印 user_path / project_path 两行路径；未知子命令 → `usage`
  - `MEMORY_COMMAND = Command(name="memory", description="记忆管理", usage="/memory [list | clear | edit]", type=CommandType.LOCAL, handler=handle_memory)`
  - `ctx.memory_manager is None` 时打印「记忆管理器未初始化」并 return

## T17: `/session` 命令

- 影响文件: `/Users/codemelo/mewcode/mewcode/commands/handlers/session.py`（`handle_session`）
- 依赖任务: T5, T6, T7
- 完成标准:
  - 空子命令打印当前 session meta 摘要（id / title / 消息 / token / 最后活跃）
  - `list` 取 `SessionManager.list()[:10]` 渲染
  - `resume` 无 ID 时列前 15 候选 + 把 ID 列表暂存到 `ctx.config["_resume_candidates"]`，再次调用支持「数字序号」简写
  - resume 成功：关旧 session、`ctx.config["set_session"]` 切换、构造新 `ConversationManager` 灌入 `result.messages`、追加 `build_time_gap_message` 返回值（若非 None）、`ctx.config["set_conversation"]` 切换、`ctx.agent._loop_count = 0`、`ctx.config["render_restored"]` 重绘 UI
  - `new` 新建 session + 清空 conversation + 复位 `_loop_count` + `clear_chat`
  - `delete <id>` 拒绝缺 ID 的调用

## T18: 接入主流程（App 启动 + 命令上下文）

- 影响文件: `/Users/codemelo/mewcode/mewcode/app.py:51-58, 550-558, 633-660, 873-881, 1063-1070, 1167-1175, 1228-1245, 1474-1495, 1564-1575, 1605-1615`
- 依赖任务: T1-T17
- 完成标准:
  - App 顶部 `from mewcode.memory import (MemoryManager, Session, SessionManager, ..., generate_session_summary, load_instructions)`
  - `__init__` 持字段：`self.session_manager / self.session / self.memory_manager / self._instructions_content`
  - 启动期：`self._instructions_content = load_instructions(work_dir)` / `self.memory_manager = MemoryManager(work_dir)` / `self.session_manager = SessionManager(work_dir)` / `self.session_manager.cleanup()` / `self.session = self.session_manager.create()`
  - `Agent(...)` 构造点透传 `instructions_content=self._instructions_content` 与 `memory_manager=self.memory_manager`
  - 用户消息提交时 `self.session.append(Message(role="user", content=text))`
  - 助手消息收尾时 `self.session.append(msg)` + `meta.total_tokens` 累计 + 异步调 `_update_session_summary`
  - `CommandContext` 填 `session / session_manager / memory_manager / agent / config["set_session"] / config["set_conversation"] / config["render_restored"] / config["clear_chat"]`
  - 退出路径 `self.session.close()`

## T19: 端到端验证

- 影响文件: 无（仅运行验证）
- 依赖任务: T18
- 完成标准:
  - `ruff check mewcode/memory/ mewcode/agent.py mewcode/conversation.py mewcode/commands/handlers/memory.py mewcode/commands/handlers/session.py` 无错误
  - `pytest tests/test_memory.py -v` 全部通过（约 30+ 用例）
  - 项目根放一个 `MEWCODE.md`，TUI 启动后 system prompt（或 conversation 首部）能看到「## 项目指令」block
  - `MEWCODE.md` 含 `@include ./sub/details.md` 时能展开并加入注入
  - TUI 发几条消息后退出，`.mewcode/sessions/` 出现 `session_*.jsonl` + `session_*.meta`；重启后 `/session list` 列出该会话；`/session resume <id>` 把对话恢复
  - 距上次活跃 ≥24h 的 resume 后，对话末尾出现一条「[系统提示] 距离上次会话已过去 N 小时/天。代码可能有变更，建议在操作前重新读取相关文件。」
  - 跟 Agent 对话 5 轮后（每轮 `loop_count` 模 5 等于 0），`.mewcode/memories.md` 或 `~/.mewcode/memories.md` 出现新行；下次启动后 `## 自动记忆` block 包含该内容；`/memory list` 列出；`/memory clear` 后回到「当前没有任何自动记忆。」

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
- [ ] T13
- [ ] T14
- [ ] T15
- [ ] T16
- [ ] T17
- [ ] T18
- [ ] T19
