# ch11: Skill 系统 Spec（Python 版）

> 本版本按课程「第 11 章 Skill 系统」全量实现 Python 版本。Skill 把可复用 prompt 升级为 Markdown 能力包，配合 progressive disclosure 与执行模式，让模型在工具变多时仍能精准触发。

## 1. 背景

MewCode 用户会反复输入一组类似的 prompt（commit message 规范、代码审查清单、跑测试的项目类型识别）。当前所有 prompt 要么写死在源码 Slash Command（`/review`）里，要么用户每次手敲，三个痛点：(1) 不能复用与分发，(2) 工具一多模型选错的概率指数级上升，(3) 没有任务级的工具白名单与上下文隔离。Skill 把可复用 SOP 装进可编辑的 Markdown 文件，配渐进式披露与执行模式，同时解决上述三个问题。

## 2. 目标

把 `SKILL.md` 升级为「带 frontmatter + 资源 + 专属工具」的能力包。启动时只把 `name + description` 注入对话给 Agent 看；Agent 通过 `LoadSkill` 工具按需把完整 SOP 钉到环境上下文，相关专属工具注册进当前会话。`inline` 模式 SOP 在主对话内执行，`fork` 模式独立子 Agent 隔离执行后把结果回流。`/<skill-name>` 显式触发与意图识别自动触发共用同一套执行器。

## 3. 功能需求

### 解析与加载

- F1: `SkillDef`（`mewcode/skills/parser.py:24`）字段：`name / description / prompt_body / allowed_tools / mode / model / context / source_path / is_directory`；`mode` 取 `inline | fork`（默认 `inline`），`context` 取 `full | recent | none`（默认 `full`，仅 fork 模式生效）
- F2: 单文件 `*.md`（YAML frontmatter + body）与目录型（`<skill>/SKILL.md` + `tool.json` + `references/*.py`）两种磁盘布局；`SkillLoader._scan_directory` 区分两类
- F3: 三级搜索路径加载（`mewcode/skills/loader.py:23`），优先级 `项目 .mewcode/skills/` > `~/.mewcode/skills/` > 内置（`importlib.resources`）；首次出现的 name 占位，后续同名跳过；解析失败单条 `warning` 日志并跳过
- F4: 启动期 `SkillLoader.load_all` 解析所有 frontmatter+body 进内存；`SkillLoader.get(name)` 每次重读源文件实现热重载，失败回退缓存（`mewcode/skills/loader.py:96`）

### 执行

- F5: `substitute_arguments(prompt_body, args)`（`mewcode/skills/parser.py:99`）把 `$ARGUMENTS` 替换为参数；没有占位符则原样返回
- F6: inline 执行：`SkillExecutor.execute_inline`（`mewcode/skills/executor.py:54`）渲染 body 后调用 `Agent.activate_skill(name, body)` 钉到 env context，主循环每轮迭代重建 environment 时 SOP 都注入；同时按 `allowed_tools` 过滤当前会话工具集
- F7: fork 执行：`SkillExecutor.execute_fork`（`mewcode/skills/executor.py:58`）创建独立 `ConversationManager`，按 `context` 字段决定历史携带（`full` = 主对话拼接摘要 / `recent` = 最近 5 条 / `none` = 完全隔离），临时 Agent 跑到 `LoopComplete` 后把累计文本回流
- F8: 工具白名单：`filter_tool_registry`（`mewcode/skills/executor.py:25`）按 `allowed_tools` 重建一个新的 `ToolRegistry`；白名单中出现不存在的工具立刻 `raise SkillDependencyError`
- F9: 系统工具豁免：`Tool.is_system_tool`（`mewcode/tools/base.py:28`）标记的工具在 `filter_tool_registry` 时自动透传，确保 `LoadSkill` 在 skill 执行期仍可用以支持嵌套调用

### LoadSkill 工具与 Skill Catalog 注入

- F10: `LoadSkill`（`mewcode/tools/load_skill.py:21`）read-only 系统工具，输入 `{name: str}`；调用 `SkillLoader.get` 取 skill → `Agent.activate_skill` 钉 SOP → 目录型 skill 调 `register_skill_tools` 注册专属工具 → 返回简短确认（不返回完整 SOP，避免 tool_result 占用空间）
- F11: 启动期 `app.py:673` 构建「Available Skills」段（只 `- <name>: <description>` 列表 + LoadSkill 调用指引），通过 `Agent.set_skill_catalog` 注入 environment context（`mewcode/prompts.py:293`）

### 命令集成

- F12: 每个 skill 由 `register_skill_commands`（`mewcode/commands/handlers/skill_register.py:18`）注册为 `/<name>` 短命令，描述末尾标注 `[skill]`；mode 字段决定运行时分支：inline 调 `execute_inline` 后再发送一次 user message 触发 loop，fork 则后台 `asyncio.create_task(_run_fork)` 把结果作为 system message 插入
- F13: `/skill list | info <name> | reload` 管理子命令（`mewcode/commands/handlers/skill.py:11`）：list 列出已加载 skill 与来源；info 显示完整 frontmatter 与文件路径；reload 重新扫描三级目录并重新注册命令
- F14: ch10 留下的 `/review` 由 review skill 接管；旧硬编码 handler 仍可保留但优先级被 skill 覆盖

### 目录型 Skill

- F15: 目录布局 `<skill>/SKILL.md` + `<skill>/tool.json` + `<skill>/references/*.py`；`tool.json` 声明该 skill 专属新增的工具 schema（function calling 兼容），LoadSkill 时 `register_skill_tools`（`mewcode/skills/directory.py:104`）把 schema 实例化为 `SkillCustomTool` 注册到 registry，工具实现由 `importlib.util.spec_from_file_location` 动态加载 `references/<tool_name>.py` 内的 `execute` 函数
- F16: `tool.json` 与 `allowed_tools` 职责分离：`tool.json` 负责「向 registry 注册新工具」，`allowed_tools` 负责「skill 执行期间可见工具白名单」；同名工具已存在则跳过注册

### 热更新与清理

- F17: `SkillLoader.get(name)` 每次调用都 `parse_skill_file(source_path)` 重读，文件修改即时生效；解析失败回退 `_cache` 中的旧版本并记 warning（`mewcode/skills/loader.py:103`）
- F18: `/clear` 命令在清对话历史时调 `Agent.clear_active_skills()`（`mewcode/commands/handlers/clear.py:19`）把激活 skill 列表清空

### 内置 Skill

- F19: 四个内置 skill 通过 `importlib.resources` 从 `mewcode/skills/builtins/` 加载：`commit`（inline）、`review`（fork, context: none）、`test`（inline）、`backend-interview`（fork, context: none，目录型自带 `parse_resume` 工具）
- F20: 加载顺序保证磁盘版本可覆盖内置：项目 → 用户 → 内置；`SkillLoader.get_source_label`（`mewcode/skills/loader.py:117`）按路径前缀返回 `project | user | builtin`

## 4. 非功能需求

- N1: 单个 skill 文件解析失败不能阻断其他 skill 加载，错误走 `logging.warning`
- N2: `LoadSkill` 工具调用不弹权限提示（read-only 类别 + `is_system_tool=True`）
- N3: fork 模式必须隔离 `ConversationManager`，主对话状态不被子 Agent 修改
- N4: 工具过滤通过 `filter_tool_registry` 返回新 `ToolRegistry` 实例实现，过滤动态生效不要求重启 Agent
- N5: 内置 skill 与磁盘上同名 skill 冲突时，磁盘版本优先（用户可覆盖内置）
- N6: 目录型 skill 工具实现走 `importlib` 动态加载 `.py` 文件而非 entry point，避免安装步骤

## 5. 设计概要

### 核心数据结构

- `SkillDef`（`mewcode/skills/parser.py:23`）：dataclass，含 `mode / model / context` 三个执行字段 + `source_path / is_directory` 元信息
- `SkillLoader`（`mewcode/skills/loader.py:15`）：name → `SkillDef`；持有 `_skills` 与 `_cache` 两份字典，热更新失败回退缓存
- `SkillExecutor`（`mewcode/skills/executor.py:43`）：`execute_inline(skill, args) -> None` 与 `execute_fork(skill, args) -> str`
- `SkillCustomTool`（`mewcode/skills/directory.py:64`）：动态 Tool 子类，`params_model` 用 `_DynamicParams(extra="allow")`，包裹 `references/*.py` 里的 `execute` 函数
- `LoadSkill`（`mewcode/tools/load_skill.py:21`）：实现 `Tool` 抽象类，`is_system_tool = True`；持有 `SkillLoader` 与 `Agent` 引用
- Agent 新增字段与方法：`active_skills: dict[str, str]`、`_skill_catalog: str`、`activate_skill(name, body)`、`clear_active_skills()`、`set_skill_catalog(catalog)`（`mewcode/agent.py:317-364`）

### 主流程

1. 启动：`MewCodeApp.__init__` → 实例化 `LoadSkill` 并 register → 构造 `Agent` → `SkillLoader(work_dir).load_all()` → `load_skill_tool.set_loader/set_agent` → 构造 `SkillExecutor` → 把 catalog 字符串写入 `agent.set_skill_catalog` → `register_skill_commands` 把每个 skill 注册成 `/<name>`
2. system prompt 注入：`build_environment_context`（`mewcode/prompts.py:277`）每轮迭代重建 environment block，把 `agent._skill_catalog` 与 `agent.active_skills` 字典分别拼为 catalog 段和「## Active Skills」段
3. 主 Agent 循环每轮 `_build_system_message` 调 `build_environment_context(work_dir, active_skills, skill_catalog, agent_catalog)`（`mewcode/agent.py:400`），实现 SOP 钉到 env 的能力
4. 显式调用 `/commit`：`register_skill_commands` 注册的 handler → `executor.execute_inline(skill, args)` → `agent.activate_skill("commit", rendered_body)` → 再 `ctx.ui.send_user_message(trigger)` 触发 Agent loop
5. 意图识别：Agent 调 `LoadSkill({name: "commit"})` → `loader.get` → `agent.activate_skill` + 目录型调 `register_skill_tools` → 返回 `"Skill 'commit' activated. SOP pinned to environment context."`
6. fork 调用 `/review`：handler 走 `asyncio.create_task(_run_fork)` → `executor.execute_fork` 新 conversation + 临时 Agent + 收集 `StreamText` 到 `LoopComplete` → 把 finalText 作为 system message 插入主对话
7. `/clear`：handler → reset conversation → `agent.clear_active_skills()` → 后续轮 environment 不再注入旧 SOP

### 调用链

- 启动：`mewcode.app.MewCodeApp.__init__` → `SkillLoader.load_all` → `register_skill_commands`（`mewcode/app.py:687`）
- inline 显式：用户 `/commit` → command handler → `executor.execute_inline` → `agent.activate_skill` → `ctx.ui.send_user_message` → Agent loop（每轮 env 注入 SOP）
- fork 显式：用户 `/review` → handler → `asyncio.create_task(execute_fork)` → `system message`
- 意图触发：Agent 在某轮调用 `LoadSkill` → `loader.get` → `agent.activate_skill` + register dir tools → 下一轮 SOP 钉在 env 里
- 清理：用户 `/clear` → `handle_clear` → conversation reset + `agent.clear_active_skills`

### 与其他模块的交互

- 上行依赖：`mewcode/app.py`（注入 system prompt、注册命令、注入 `SkillLoader/SkillExecutor` 到 `CommandContext.config`）、`Agent`（`active_skills` 字段 + env 注入）、`ConversationManager`（fork 用独立实例）、`ToolRegistry`（动态注册目录型工具）
- 下行：`SkillExecutor` 通过 `from mewcode.agent import Agent` 局部 import 避免循环依赖；`SkillCustomTool` 通过 `importlib.util` 加载用户脚本，不依赖 entry point

## 6. Out of Scope

- 远程安装 Skill（`InstallSkill` 工具）：Python 版本暂不实现，用户需手动 clone 到 `.mewcode/skills/` 下
- 嵌套深度限制：Skill A → LoadSkill(B) → LoadSkill(C) 不做主动限制，依赖 Agent `max_iterations` 自然封顶
- fork 嵌套跨 Agent 边界的父子链路记录：留给后续 SubAgent 章节
- 目录型 skill 工具的 sandbox：`SkillCustomTool` 执行用户 `.py` 不做沙箱，与本机 Python 同权限运行
- 用户级 `~/.mewcode/skills/` 与项目级冲突时的合并策略：高优先级目录里出现的 name 直接覆盖，不做字段级 merge

## 7. 完成定义

见 [checklist.md](checklist.md)，所有条目勾上即完成。
