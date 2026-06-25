# ch09: 记忆系统 Spec

## 1. 背景

只活在单轮对话里的 Agent 每次启动都是一张白纸：用户偏好得反复说、项目背景得反复讲、上次的会话内容拿不回来。MewCode 用三种正交的「记忆」机制组合解决这三个问题：项目级指令文件（`MEWCODE.md`，支持 `@include` 模块化）、会话存档（jsonl 落盘 + meta 索引、可 `/session resume` 恢复）、自动记忆（每 N 轮 loop 后 fire-and-forget 跑一次提取，把对话中提到的偏好/反馈/项目/参考四类信息写进 `.mewcode/memories.md`，下次启动随系统提示注入回上下文）。本章把这三套机制全部落到 MewCode 的 Python 实现。

## 2. 目标

交付三类相互独立的持久化机制，给 TUI 启动与运行期共同使用：项目指令加载（按三层优先级拼装 `MEWCODE.md` 并展开 `@include`）、会话存档（消息追加写 jsonl、`/session resume` 列表与恢复、tool-use/tool-result 链路完整性校验、断会话时长提示）、自动记忆（项目级 + 用户级双路 memories.md、每 5 轮 loop 触发 LLM extractor 自动改写、`/memory list/clear/edit` 命令暴露）。最终 TUI 启动时三套机制都开箱可用，注入路径由 `ConversationManager.inject_long_term_memory` 统一收口。

## 3. 功能需求

### 项目指令

- F1: `load_instructions(project_root)` 按三层优先级（`<project_root>/MEWCODE.md` → `<project_root>/.mewcode/MEWCODE.md` → `~/.mewcode/MEWCODE.md`）依次读取，用 `\n---\n` 拼接为一段；缺失文件静默跳过。
- F2: `process_includes(content, base_dir, project_root, depth)` 递归展开 `@include <path>` 行；`MAX_INCLUDE_DEPTH=5` 兜底；越界路径（解析后不在 `project_root` 内）替换为 `<!-- @include blocked: path outside project -->`；文件不存在替换为 `<!-- @include skipped: file not found -->`。
- F3: 拼装后的整段通过 `Agent.instructions_content` 字段透传，由 `ConversationManager.inject_long_term_memory` 以 `## 项目指令\n...` 形式插入对话首部。

### 会话存档

- F4: `SessionRecord` dataclass 持 `type / content / timestamp / tool_use_id / is_error`；`RecordType` 枚举 5 种（`system_prompt / user / assistant / tool_result / compression`）；`to_jsonl / from_jsonl` 一对方法做序列化，反序列化失败返回 `None`。
- F5: `SessionRecord.from_message(message)` 把单条 `Message` 拆成一到多条 jsonl 记录（含 tool_use 内联到 assistant content blocks、tool_results 各自一条 `tool_result` 记录）。
- F6: `Session.append(message)` 把 `from_message` 拆出的记录逐条写 jsonl + `\n` 并 `flush`；同步更新 `meta.message_count / last_active`；首条 user 消息截断到 `TITLE_MAX_LENGTH=50` 写入 `meta.title`；`Session.close()` 安全 `flush + close`。
- F7: `SessionManager(work_dir)` lazy 创建 `<work_dir>/.mewcode/sessions/`；`create()` 用 `session_<YYYYMMDD_HHMMSS>_<4 字符 suffix>` 命名；`list()` 扫所有 `*.meta` 反序列化并按 `last_active` 倒序；`resume(id)` 读取 jsonl + meta、校验链路完整性、重建 `[Message]`、追加打开 jsonl 续写；`delete(id)` / `cleanup(max_age_days=30)` 维护清理。
- F8: `records_to_messages(records)` 把 jsonl 记录序列还原成 `[Message]`：连续 tool_result 合并到下一条 user 消息的 `tool_results`、`assistant` content 为 list 时拆出 text + tool_uses、`system_prompt` 跳过、`compression` 渲染成 `[摘要]\n...` 的 user 消息。
- F9: `validate_message_chain(records)` 扫描 tool_use ↔ tool_result 配对状态，返回链路完整的最大前缀长度；resume 时用该长度截断防止把缺少 tool_result 的 tool_use 灌回去触发 API 400。
- F10: `build_time_gap_message(last_active)` 在距上次活跃 ≥ `TIME_GAP_THRESHOLD=24h` 时返回一条系统提示 `Message`（≥48 小时表达为「N 天」，否则「N 小时」），追加到恢复后的对话尾部提示用户「代码可能有变更」。

### 自动记忆

- F11: `MemoryManager(project_root)` 构造时计算两个固定路径：`~/.mewcode/memories.md`（用户级，跨项目）和 `<project_root>/.mewcode/memories.md`（项目级），由 `user_path / project_path` property 暴露。
- F12: `load()` 拼装两层 memories.md 内容（`\n\n` 分隔），返回空字符串时调用方跳过注入。
- F13: `clear()` 把两个 memories.md 截断为空（不删除文件本身，保持后续直接 `write_text` 可用）。
- F14: `get_display_text()` 为 `/memory list` 渲染层级标注（`[用户级] <path>\n<content>` / `[项目级] ...`）；两路皆空返回 `"当前没有任何自动记忆。"`。
- F15: `MEMORY_EXTRACTION_PROMPT` 文本固化四类分类（用户偏好 / 纠正反馈 / 项目知识 / 参考资料），要求 LLM 输出完整 memories.md、空分类下不写占位、相同含义条目不重复添加。
- F16: `extract(client, conversation, protocol)` 从 `conversation.history[self._last_extraction_msg_count:]` 取增量对话、拼装 prompt、跑一次非流式 `client.stream` 收集 text、`_write_memories` 解析输出按 header 关键字 (`_USER_LEVEL_HEADERS={"用户偏好","纠正反馈"} / _PROJECT_LEVEL_HEADERS={"项目知识","参考资料"}`) 分流写入两路 memories.md；异常静默 return。
- F17: `_is_placeholder(line)` 把 `"" / "..." / "…" / "无" / "暂无" / "N/A"` 等占位行过滤掉，防止 LLM 把空模板回写为「真记忆」。

### Agent 钩子 + 注入

- F18: `Agent.__init__` 接受 `instructions_content / memory_manager` 两个字段；`Agent.run` 入口先 `inject_environment` 再 `inject_long_term_memory(instructions, memory_manager.load() if memory_manager else "")`；auto-compact 重置历史后同样的二次注入。
- F19: 自动提取触发：`Agent` 内置 `MEMORY_EXTRACTION_INTERVAL=5` 与 `_loop_count`；每次 `len(response.tool_calls)==0` 分支（loop 收尾）递增 `_loop_count`；当 `_loop_count % MEMORY_EXTRACTION_INTERVAL == 0` 且 `memory_manager` 存在时 `asyncio.ensure_future(self._extract_memories(conversation))` fire-and-forget。
- F20: `_extract_memories(conversation)` 用 `self._extracting` 互斥防重入；try-except-finally 包裹 `memory_manager.extract`，异常仅 `log.debug` 不传播。

### 对话注入

- F21: `ConversationManager.inject_long_term_memory(instructions, memories)` 幂等：若 `ltm_injected` 已为 True 直接 return；否则按 `env_injected` 计算 base pos，再依次插入 `## 项目指令\n...` 和 `## 自动记忆\n...` 两条 user message，最后追加一条 assistant 占位 `"好的，我已了解项目背景和记忆。"` 把视觉边界封死；只有当 `instructions` 或 `memories` 至少一个非空时才置 `ltm_injected=True`。
- F22: `ConversationManager.replace_history(new_messages)` 在 resume / 切换会话时重置 `env_injected=False / ltm_injected=False`，下次 Agent.run 才能重新注入。

### TUI / 命令

- F23: `/memory` 命令 (`handle_memory`) 暴露子命令 `list / clear / edit`：list 打印 `get_display_text`、clear 调 `MemoryManager.clear` 并提示「所有自动记忆已清空。」、edit 打印两个 memories.md 路径供用户外部编辑、空参数等价于 list、未知子命令给出 usage。
- F24: `/session` 命令 (`handle_session`) 暴露 `list / resume / new / delete`：list 取 `SessionManager.list()` 前 10 条按 `last_active` 渲染；resume 无参数时打印前 15 条候选并把 ID 列表暂存到 `ctx.config["_resume_candidates"]`，再次执行时支持「序号」简写；resume 成功调 `ctx.config["set_session"](result.session)`、构造新 `ConversationManager` 灌入 `result.messages`、追加 `build_time_gap_message` 输出、`ctx.config["set_conversation"]` 切换、`ctx.config["render_restored"]` 渲染、`ctx.agent._loop_count=0` 复位 extractor cursor。
- F25: `App` (TUI 入口) 启动时调 `load_instructions(work_dir)` / `MemoryManager(work_dir)` / `SessionManager(work_dir).cleanup() / create()`，把结果存到 `self._instructions_content / self.memory_manager / self.session / self.session_manager`，再传入 `Agent` 构造函数；用户每条消息提交后 `self.session.append`，助手每条消息收尾后 `self.session.append` + 异步 `_update_session_summary`。

## 4. 非功能需求

- N1: 加载失败一律静默：找不到文件、`@include` 文件缺失、jsonl 行解析失败、meta 文件损坏都返回空或 None，不中断 TUI 启动。
- N2: `@include` 必须循环安全：`MAX_INCLUDE_DEPTH=5` 兜底 + 路径越界拦截。
- N3: jsonl 单行可能很长（tool_result 含大段输出），写入时按行 `json.dumps(ensure_ascii=False)` + `\n` + `flush`；读取时按行 `strip` 后逐条 `from_jsonl`，空行/失败行跳过。
- N4: memories.md 是「LLM 覆盖式重写」语义，每次 extract 全量覆盖；占位行过滤 (`_is_placeholder`) 是最后的语义防线。
- N5: 自动提取必须 fire-and-forget：用 `asyncio.ensure_future` 不 `await`；`_extracting` 互斥防止上一轮还没跑完又起一轮。
- N6: `inject_long_term_memory` 必须幂等：单次 Agent.run 反复进入（compact 触发的二次注入除外）只插一次。
- N7: 公开符号都被外部模块调用，无死代码：`from mewcode.memory import ...` 在 `mewcode/memory/__init__.py` 集中 re-export，`tests/test_memory.py` 全部命中。

## 5. 设计概要

### 核心数据结构

- `mewcode.memory.auto_memory.MemoryManager`：双路径 + `_last_extraction_msg_count` cursor
- `mewcode.memory.session.RecordType`：5 个值的字符串 Enum
- `mewcode.memory.session.SessionRecord`：dataclass，jsonl 行结构
- `mewcode.memory.session.SessionMeta`：dataclass，会话元数据（id / title / summary / message_count / total_tokens / created_at / last_active），独立 `.meta` JSON 落盘
- `mewcode.memory.session.Session`：活跃会话句柄，持 jsonl file handle 与 meta
- `mewcode.memory.session.ResumeResult`：dataclass，含恢复后的 `session + messages + last_active`
- `mewcode.memory.session.SessionManager`：工厂 + 目录管理
- `mewcode.memory.instructions.MAX_INCLUDE_DEPTH / INCLUDE_PREFIX`：模块常量
- `mewcode.conversation.ConversationManager`：`env_injected / ltm_injected` 两个标志位 + `inject_long_term_memory / replace_history`

### 主流程

- **项目指令**：App 启动 → `load_instructions(work_dir)` → 三层文件读 → `process_includes` 递归展开 → 返回单字符串 → `Agent(instructions_content=...)` → `Agent.run` 入口 `conversation.inject_long_term_memory(self.instructions_content, ...)`。
- **会话存档**：App 启动 → `SessionManager.cleanup() / create()` → `self.session = Session(...)`；用户消息 → `self.session.append(Message(role="user", ...))`；助手消息收尾 → `self.session.append(Message(role="assistant", ...))`；TUI 退出 → `self.session.close()`；`/session resume` → `SessionManager.resume(id)` → `validate_message_chain` 截断 → `records_to_messages` 重建 → `ConversationManager.replace_history` → 注入 time gap message。
- **自动记忆**：Agent loop 收尾 (`len(tool_calls)==0`) → `_loop_count += 1` → 模 5 触发 → `asyncio.ensure_future(self._extract_memories(conversation))` → `MemoryManager.extract` → `client.stream` 跑 extractor prompt → `_write_memories` 按 header 关键字分流写入 user/project memories.md → 下次 `Agent.run` 入口 `memory_manager.load()` 读回 → `inject_long_term_memory` 注入 `## 自动记忆` block。
- **`/memory` 命令**：用户输入 `/memory list` → `handle_memory` 派发 → `MemoryManager.get_display_text` → `ctx.ui.add_system_message`；`/memory clear` → `MemoryManager.clear` → 提示文字。
- **断会话提示**：`/session resume <id>` → 取 `meta.last_active` → `build_time_gap_message` 判断 ≥24h → 追加一条 user message 到恢复对话末尾。

### 调用链（模块层级）

- 启动：`mewcode.app.App` → `mewcode.memory.{load_instructions, MemoryManager, SessionManager}` → `mewcode.agent.Agent(instructions_content, memory_manager)`
- 运行：`mewcode.agent.Agent.run` → `mewcode.conversation.ConversationManager.inject_long_term_memory` + `mewcode.memory.auto_memory.MemoryManager.{load, extract}`
- 命令：`mewcode.commands.handlers.memory.handle_memory` ↔ `MemoryManager.{get_display_text, clear}`；`mewcode.commands.handlers.session.handle_session` ↔ `SessionManager.{list, resume, create, delete}` + `build_time_gap_message`

### 与其他模块的交互

- `mewcode/conversation.py`：提供 `inject_long_term_memory / replace_history` 注入面板；`ltm_injected` 是幂等开关。
- `mewcode/agent.py`：吸纳 `instructions_content / memory_manager` 字段；`MEMORY_EXTRACTION_INTERVAL / _loop_count / _extracting / _extract_memories` 串起提取闭环。
- `mewcode/app.py`：唯一安装点，承担三套机制的 lazy 构造与 session lifecycle 管理。
- `mewcode/commands/handlers/`：通过 `CommandContext.{memory_manager, session_manager, session, agent, config, ui}` 拿到所有句柄，无需直接访问 App。
- `mewcode/context.py`：`ensure_session_dir / auto_compact` 与 memory 的关系是「auto-compact 触发后再次注入 LTM」，靠 `ltm_injected=False`（由 `replace_history` 重置）协调。

### 新增文件 / 函数清单

新增（全部位于 `/Users/codemelo/mewcode/mewcode/memory/`）：

- `__init__.py`：re-export `MemoryManager / load_instructions / process_includes / Session / SessionManager / SessionMeta / SessionRecord / ResumeResult / build_time_gap_message / generate_session_summary / validate_message_chain`
- `auto_memory.py`：`MemoryManager / MEMORY_EXTRACTION_PROMPT / _USER_LEVEL_HEADERS / _PROJECT_LEVEL_HEADERS / USER_MEMORIES_RELPATH / PROJECT_MEMORIES_RELPATH`
- `instructions.py`：`load_instructions / process_includes / MAX_INCLUDE_DEPTH / INCLUDE_PREFIX`
- `session.py`：`RecordType / SessionRecord / records_to_messages / validate_message_chain / SessionMeta / Session / ResumeResult / generate_session_summary / build_time_gap_message / _generate_session_id / SessionManager` + 模块常量

修改：

- `mewcode/agent.py`：`MEMORY_EXTRACTION_INTERVAL` 常量 + `Agent.__init__` 接 `instructions_content / memory_manager` + `Agent.run` 入口注入 + loop 收尾分支触发 `_extract_memories` + auto-compact 后重新注入。
- `mewcode/conversation.py`：`inject_long_term_memory / replace_history / ltm_injected`。
- `mewcode/commands/handlers/memory.py`：`handle_memory / MEMORY_COMMAND`。
- `mewcode/commands/handlers/session.py`：`handle_session`（含 list / resume / new / delete）。
- `mewcode/app.py`：启动期构造 + session lifecycle 管理 + 命令上下文字段填充。

## 6. Out of Scope

- 会话过期清理的策略可配置化：硬编码 `DEFAULT_MAX_AGE_DAYS=30`，暂不开放。
- 团队 / 远程 memory 同步：只做本地双路径个人模式。
- Memory 老化提示：本章 Python 实现暂不带 freshness / drift caveat。
- 相关性召回（selector LLM 选 ≤5 条 memory 注入）：本章不实现，所有 memories 一并注入。
- 自动提取的「写时互斥跳过」（Agent 自己刚改过 memories.md 就跳过本轮 extractor）：靠 LLM 提示词 + 占位过滤兜底，不在调度层做。
- 会话 jsonl 的 schema 版本化与迁移。
- MEWCODE.md 大文件警告与裁剪。
- `MEWCODE_REMOTE_MEMORY_DIR` 等环境变量覆盖。

## 7. 完成定义

见 [checklist.md](checklist.md)，所有条目勾上即完成。
