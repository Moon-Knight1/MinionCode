# ch11: Skill 系统 Checklist（Python 版）

> 所有条目必须可勾选、可观测。验收方式写在每项后面的括号里。操作目录在仓库根 `/Users/codemelo/mewcode`。

## 1. 实现完整性

### 1.1 解析与加载

- [ ] `mewcode/skills/parser.py:23` `SkillDef` 含字段 `name / description / prompt_body / allowed_tools / mode / model / context / source_path / is_directory`（`grep -n "class SkillDef" mewcode/skills/parser.py` 命中）
- [ ] `mewcode/skills/parser.py:36` `parse_frontmatter(raw) -> (dict, str)` 处理 `---\n...\n---` 格式
- [ ] `mewcode/skills/parser.py:57` `_validate_meta` 校验 `name` 正则 + `mode in {inline, fork}` + `context in {full, recent, none}`
- [ ] `mewcode/skills/parser.py:99` `substitute_arguments(prompt_body, args)` 实现 `$ARGUMENTS` 替换
- [ ] `mewcode/skills/loader.py:15` `SkillLoader(work_dir)` 实现三级搜索（`grep -n "PROJECT_SKILLS_DIR\|USER_SKILLS_DIR\|_load_builtins" mewcode/skills/loader.py` 命中 ≥3 处）
- [ ] `mewcode/skills/loader.py:96` `get(name)` 每次重读源文件实现热重载，失败回退 `_cache` 并 `log.warning`
- [ ] `mewcode/skills/loader.py:117` `get_source_label(name)` 按 `_project_dir / _user_dir` 前缀返回 `project | user | builtin`

### 1.2 内置 skill

- [ ] `mewcode/skills/builtins/commit/SKILL.md` 存在，frontmatter `mode: inline / allowedTools: [Bash, ReadFile, Grep]`
- [ ] `mewcode/skills/builtins/review/SKILL.md` 存在，frontmatter `mode: fork / context: none / allowedTools: [Bash, ReadFile, Grep, Glob]`，body 含「Critical」「Warning」「Info」分级
- [ ] `mewcode/skills/builtins/test/SKILL.md` 存在，frontmatter `mode: inline / allowedTools: [Bash, ReadFile, Grep, Glob]`，body 含 `pyproject.toml` 与 `go.mod` 检测
- [ ] `mewcode/skills/builtins/backend-interview/SKILL.md` 存在 + `tool.json` 声明 `parse_resume` + `references/parse_resume.py` 含 `async def execute(file_path: str = "", **kwargs)`
- [ ] `mewcode/skills/loader.py:65` `_load_builtins` 使用 `importlib.resources.files("mewcode.skills.builtins")` 遍历

### 1.3 Executor

- [ ] `mewcode/skills/executor.py:43` 含 `class SkillExecutor` 与 `execute_inline / execute_fork` 两个方法
- [ ] inline 调用链：`substitute_arguments` → `agent.activate_skill(name, body)`（`grep -n "activate_skill" mewcode/skills/executor.py` 命中）
- [ ] fork 调用链：新 `ConversationManager` → `_build_fork_context(context)` 按 `full / recent / none` 三档装填 → `filter_tool_registry` 过滤工具 → 临时 Agent run → 收集 `StreamText` 到 `LoopComplete`
- [ ] `mewcode/skills/executor.py:25` `filter_tool_registry(registry, allowed)` 缺工具 `raise SkillDependencyError`，系统工具自动透传

### 1.4 Agent 集成

- [ ] `mewcode/agent.py:317` Agent 含 `self.active_skills: dict[str, str] = {}`
- [ ] `mewcode/agent.py:357` `activate_skill(name, prompt_body)` 实现
- [ ] `mewcode/agent.py:360` `clear_active_skills()` 实现
- [ ] `mewcode/agent.py:363` `set_skill_catalog(catalog)` 实现
- [ ] `mewcode/agent.py:400` 主循环每轮调用 `build_environment_context(work_dir, active_skills, skill_catalog, agent_catalog)`
- [ ] `mewcode/prompts.py:277` `build_environment_context` 把 `active_skills` 拼成 `## Active Skills` 段；`skill_catalog` 拼到 environment block

### 1.5 LoadSkill 工具与系统工具豁免

- [ ] `mewcode/tools/load_skill.py:21` 含 `class LoadSkill(Tool)`，`name = "LoadSkill"`、`category = "read"`、`is_system_tool = True`
- [ ] `mewcode/tools/load_skill.py:39` `set_loader / set_agent` 注入方法
- [ ] `mewcode/tools/load_skill.py:46` `execute` 调 `loader.get → agent.activate_skill → register_skill_tools`（目录型）→ 返回 `"Skill '<name>' activated. SOP pinned to environment context."`
- [ ] `mewcode/tools/base.py:28` `Tool.is_system_tool: bool = False` 类属性
- [ ] `mewcode/skills/executor.py:14` `SYSTEM_TOOL_NAMES = frozenset({"LoadSkill"})` 常量
- [ ] `filter_tool_registry` 应用 `allowed_tools` 时不剔除 `is_system_tool=True` 的工具（`grep -n "is_system_tool" mewcode/skills/executor.py` 命中）

### 1.6 目录型 skill

- [ ] `mewcode/skills/directory.py:17` `parse_tool_json(path)` 支持 list 与单 dict 两种格式
- [ ] `mewcode/skills/directory.py:34` `load_tool_implementation` 用 `importlib.util.spec_from_file_location` 动态加载 `references/<name>.py` 内的 `execute` 函数
- [ ] `mewcode/skills/directory.py:64` `SkillCustomTool` 继承 `Tool`，`params_model = _DynamicParams`（`extra="allow"`）
- [ ] `mewcode/skills/directory.py:104` `register_skill_tools(skill_dir, registry) -> int` 遍历 tool.json，注册成功 +1，重名跳过
- [ ] `backend-interview` 的 `parse_resume` 工具能通过 `register_skill_tools` 注册到 registry（见 `tests/test_skills.py` `test_register_skill_tools`）

### 1.7 命令集成

- [ ] 每个 skill 自动注册为 `/<name>` 命令，描述末尾含 `[skill]`（`grep -n "\\[skill\\]" mewcode/commands/handlers/skill_register.py` 命中）
- [ ] `mewcode/commands/handlers/skill_register.py:18` `register_skill_commands(registry, loader, executor)` 实现；模块级 `_REGISTERED_SKILL_NAMES` 跟踪重复注册
- [ ] inline skill 命令 handler 调 `executor.execute_inline` 后再 `ctx.ui.send_user_message(trigger)`
- [ ] fork skill 命令 handler 走 `asyncio.create_task(_run_fork)`，结果作为 `add_system_message` 插入
- [ ] `mewcode/commands/handlers/skill.py:11` `/skill list | info <name> | reload` 子命令分发
- [ ] `mewcode/commands/handlers/clear.py:19` `handle_clear` 调用 `ctx.agent.clear_active_skills()`

## 2. 接入完整性（杜绝死代码）

- [ ] `grep -rn "SkillLoader" mewcode/app.py` 命中 ≥2 处（import + 实例化）
- [ ] `grep -rn "activate_skill" mewcode/` 命中 Agent 方法定义 + Executor + LoadSkillTool 三处调用
- [ ] `grep -rn "clear_active_skills" mewcode/` 命中 `/clear` handler 调用 + Agent 方法定义
- [ ] `grep -rn "LoadSkill\|\"LoadSkill\"" mewcode/` 命中 tool 定义 + app 注册 + 至少 1 个测试
- [ ] `grep -rn "SkillExecutor\|register_skill_commands" mewcode/` 命中 app.py 注册 + handler 模块
- [ ] `grep -rn "execute_inline\|execute_fork" mewcode/skills/` 命中 Executor 定义 + handler 调用
- [ ] `grep -rn "loader.get\|SkillLoader.get" mewcode/tools/load_skill.py` 命中 1 处
- [ ] `grep -rn "is_system_tool" mewcode/` 命中 base.py 定义 + executor filter 检查 + LoadSkill 实现
- [ ] `mewcode/app.py:556` 存在 `self.skill_loader` / `self.skill_executor` / `self._load_skill_tool` 字段
- [ ] `mewcode/app.py:885` `CommandContext.config` 字典塞入 `"skill_loader"` 与 `"skill_executor"` key

## 3. 编译与测试

- [ ] `cd /Users/codemelo/mewcode && ruff check mewcode/skills mewcode/tools/load_skill.py` 无 error
- [ ] `cd /Users/codemelo/mewcode && pytest tests/test_skills.py -q` 全部通过
- [ ] `cd /Users/codemelo/mewcode && pytest tests/test_agent.py -q` 全部通过
- [ ] `cd /Users/codemelo/mewcode && python -c "from mewcode.skills.loader import SkillLoader; l = SkillLoader('/tmp'); print(list(l.load_all().keys()))"` 输出含 `commit / review / test / backend-interview`
- [ ] `cd /Users/codemelo/mewcode && python -c "from mewcode.tools.load_skill import LoadSkill; t = LoadSkill(); print(t.name, t.category, t.is_system_tool)"` 输出 `LoadSkill read True`

## 4. 端到端验证（手动操作 TUI）

> 启动命令：`cd /Users/codemelo/mewcode && python -m mewcode`

- [ ] 启动后输 `/help`，看到 `/commit [skill]` / `/review [skill]` / `/test [skill]` / `/backend-interview [skill]` / `/skill` 都列出
- [ ] 输 `/skill list`，输出含 4 个 skill 名称 + 来源（builtin / project / user）
- [ ] 输 `/skill info commit`，输出含完整 frontmatter（mode / context / model / AllowedTools / Source / Path）
- [ ] 改一处真实文件（如修个空格），输 `/commit`，看到 Agent 真的走 git status → diff → 生成 commit message → git add → git commit；`git log -1` 看到新 commit
- [ ] 输 `/review`，看到 fork 路径执行：主对话不污染；末尾以 `[review skill result]` 开头收到摘要含 Critical/Warning/Info 分级
- [ ] 自然语言 `"帮我准备一下后端面试"`，Agent tool_use 里出现 `LoadSkill({name: "backend-interview"})` 并且 environment 段里出现该 skill 的 SOP
- [ ] 输 `/clear`，立即输任意消息，environment 段里**不再出现** `## Active Skills`
- [ ] 修改 `.mewcode/skills/<name>.md` 一行（如自建一个 `custom.md`），**不重启** TUI，再 `/custom`，看到新行进入 prompt（热重载验证）
- [ ] 创建 `.mewcode/skills/bad.md` 故意写错 frontmatter，启动日志出现 `Skipping ... skill 'bad': ...` warning，其他 skill 仍正常加载
- [ ] LoadSkill 工具调用时**不**弹权限提示（`category=read` + `is_system_tool=True`）

## 5. 文档

- [ ] `docs/python/ch11/spec.md` 更新到课程全量版（不是验收版）
- [ ] `docs/python/ch11/tasks.md` 13 个任务全部勾上
- [ ] `docs/python/ch11/checklist.md` 全部条目勾上
- [ ] commit 信息：`feat(ch11): full skill system per course design (python) [spec/tasks/checklist closed]`
