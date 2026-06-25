# ch14: Worktree Tasks

> 任务粒度：每个任务可在一次会话内完成，可独立交付。

## T1: 实现 Slug 校验 + 命名映射

- 影响文件: `mewcode/worktree/slug.py`（`MAX_SLUG_LENGTH` @ 5；`_SEGMENT_RE` @ 6；`validate_slug` @ 9-24；`flatten_slug` @ 27-28）
- 依赖任务: 无
- 完成标准: `validate_slug` 校验长度 ≤ 64、按 `/` 切段、每段匹配 `^[a-zA-Z0-9._-]+$`、显式拒绝空段和 `.` / `..` 段，错误返回带原因字符串，合法返回 `None`；`flatten_slug(s) = s.replace("/", "+")`；分支名拼接由调用方做 `f"worktree-{flat_slug}"`。

- [ ] 完成

## T2: 定义数据模型

- 影响文件: `mewcode/worktree/models.py`（`Worktree` @ 7-14；`WorktreeSession` @ 17-25）
- 依赖任务: 无
- 完成标准: `Worktree` dataclass 含 6 字段（`name / path / branch / based_on / head_commit / created`），`created` 默认 `datetime.now`；`WorktreeSession` dataclass 含 7 字段（`original_cwd / worktree_path / worktree_name / original_branch / original_head_commit / session_id="" / hook_based=False`），后两字段有默认值。

- [ ] 完成

## T3: 实现变更检测 fail-closed

- 影响文件: `mewcode/worktree/changes.py`（`GIT_ENV` @ 9；`_run_git` @ 12-22；`Changes` @ 25-28；`count_worktree_changes` @ 31-51；`has_worktree_changes` @ 54-56；`CleanupResult` @ 59-63；`has_unpushed_commits` @ 66-74）
- 依赖任务: 无
- 完成标准: `_run_git` 强制 `env={**os.environ, **GIT_ENV}` + `timeout=30`；`count_worktree_changes` 跑 `git status --porcelain` + `git rev-list --count <head>..HEAD`，任一 `SubprocessError / OSError / ValueError` 把对应字段置 1（**fail-closed**）；`has_worktree_changes` 任一计数 > 0 返 True；`has_unpushed_commits` 跑 `git rev-list --max-count=1 HEAD --not --remotes`，git 失败返 True；`CleanupResult` dataclass 含 `kept / path / branch`。

- [ ] 完成

## T4: 实现 SubAgent worktree 上下文 notice

- 影响文件: `mewcode/worktree/integration.py`（`WORKTREE_NOTICE_TEMPLATE` @ 9-20；`generate_worktree_name` @ 23-24；`build_worktree_notice` @ 27-31）
- 依赖任务: 无
- 完成标准: `WORKTREE_NOTICE_TEMPLATE` 多行字符串，含 `[WORKTREE CONTEXT]` / `[/WORKTREE CONTEXT]` 标记、`{wt_path}` 和 `{parent_cwd}` 占位、关键句 "running in an isolated Git Worktree"、"translate them to your local worktree path"、"re-read files before editing"；`generate_worktree_name()` 返回 `f"agent-{secrets.token_hex(4)}"`（8 hex 字符，匹配 cleanup `^agent-a[0-9a-f]{7}$` 不严格但实际产出 `agent-` 开头 8 hex）；`build_worktree_notice(parent_cwd, wt_path)` 用 `.format()` 注入两个占位。

- [ ] 完成

## T5: 实现会话持久化

- 影响文件: `mewcode/worktree/session.py`（`SESSION_FILENAME` @ 11；`_session_path` @ 14-15；`save_worktree_session` @ 18-36；`load_worktree_session` @ 39-58）
- 依赖任务: T2
- 完成标准: `SESSION_FILENAME = "worktree_session.json"`；`save_worktree_session(mewcode_dir, session)`：`mkdir(parents=True, exist_ok=True)` → `session is None` 时写 `"{}"` 等价清空 → 否则 dump 7 字段到 JSON；`load_worktree_session`：文件不存在返 `None`，`JSONDecodeError / KeyError` 时 `log.warning` 后返 `None`，dict 为空或缺 `worktree_path` 返 `None`，否则构造 `WorktreeSession`，`session_id` / `hook_based` 用 `data.get(...)` 容忍旧版字段缺失。

- [ ] 完成

## T6: 实现创建后设置四项

- 影响文件: `mewcode/worktree/setup.py`（`LOCAL_CONFIG_FILES` @ 12-15；`perform_post_creation_setup` @ 18-29；`_copy_local_configs` @ 32-41；`_setup_git_hooks` @ 44-67；`_create_symlinks` @ 70-82；`_copy_ignored_files` @ 85-131）
- 依赖任务: 无
- 完成标准: `perform_post_creation_setup` 依序调四项 A/B/C/D；`LOCAL_CONFIG_FILES = ["settings.local.json", ".env"]`；A `_copy_local_configs` 用 `shutil.copy2`，`OSError` 仅 warning 不抛；B `_setup_git_hooks` 优先 `<repo>/.husky` 回退 `<repo>/.git/hooks`，找到目录跑 `git config core.hooksPath`；C `_create_symlinks` 遍历 `directories`，跳已存在和不存在的，`OSError` warning；D `_copy_ignored_files` 读 `.worktreeinclude`（跳空行和 `#`）→ `git ls-files --others --ignored --exclude-standard --directory` → `fnmatch.fnmatch` 筛选 → 单文件失败 `continue` 不中断。

- [ ] 完成

## T7: 实现 WorktreeManager 主类 + 快速恢复

- 影响文件: `mewcode/worktree/manager.py`（`GIT_ENV` @ 28；`WorktreeError` @ 31-32；`WorktreeManager.__init__` @ 36-54；`add_cache_clear_callback` @ 56-57；`_clear_all_caches` @ 59-66；`_run_git` @ 68-78；`read_worktree_head_sha` @ 84-128；`_get_current_branch` @ 316-321；`_get_head_commit` @ 323-328）
- 依赖任务: T1, T3
- 完成标准: `WorktreeManager` 持有 `repo_root / file_cache / symlink_directories / worktree_dir / _mewcode_dir / _lock=asyncio.Lock() / active: dict / current_session: WorktreeSession | None`；`worktree_dir` 默认 `<repo_root>/.mewcode/worktrees`；`_run_git` 强制 `env={**os.environ, **GIT_ENV}` + `cwd=cwd or repo_root` + `stdin=subprocess.DEVNULL` + `timeout=60`；`read_worktree_head_sha` 静态方法，完整链路（`.git pointer → gitdir → commondir → HEAD → loose ref/packed-refs`），失败返 `None`，目标延迟无 git 子进程。

- [ ] 完成

## T8: 实现 create + enter + exit + _remove_worktree

- 影响文件: `mewcode/worktree/manager.py`（`create` @ 134-186；`enter` @ 192-212；`exit` @ 218-243；`_remove_worktree` @ 249-260；`auto_cleanup` @ 266-275；`list_worktrees / get_current_session` @ 281-285；`restore_session` @ 291-310）
- 依赖任务: T2, T5, T6, T7
- 完成标准: `create` 在 `async with self._lock` 内：`validate_slug` → `active` 字典重名检查 → 快速恢复（`read_worktree_head_sha` 命中直接构造 `Worktree`，**不**跑 setup）→ 未命中 `os.makedirs(worktree_dir, exist_ok=True)` → `git worktree add -B worktree-<flat> <path> <base_branch>` → `perform_post_creation_setup`；`enter`：`_clear_all_caches` → `os.getcwd` + `_get_current_branch` + `_get_head_commit` → 写 `current_session` + `save_worktree_session`；`exit`：`action="remove" and not discard_changes` 时变更保护抛 `WorktreeError` 含具体计数 → 清缓存 + 清单例 + `save_worktree_session(None)` → `action="remove"` 调 `_remove_worktree`；`_remove_worktree`：`git worktree remove --force` → `await asyncio.sleep(0.1)` → `git branch -D worktree-<flat>` → `active.pop`；`auto_cleanup`：脏返 `CleanupResult(kept=True, path, branch)`，干净 `_remove_worktree` 返 `CleanupResult(kept=False)`；`restore_session`：读持久化 → `read_worktree_head_sha` 验证 → 命中写回 `active` + `current_session`，未命中调 `save_worktree_session(None)` 清脏。

- [ ] 完成

## T9: 实现后台过期清理

- 影响文件: `mewcode/worktree/cleanup.py`（`EPHEMERAL_PATTERNS` @ 16-22；`_is_ephemeral` @ 25-26；`cleanup_stale_worktrees` @ 29-81；`start_stale_cleanup_task` @ 84-96）
- 依赖任务: T3, T8
- 完成标准: `EPHEMERAL_PATTERNS` 五条正则：`^agent-a[0-9a-f]{7}$` / `^wf_[0-9a-f]{8}-[0-9a-f]{3}-\d+$` / `^wf-\d+$` / `^bridge-[A-Za-z0-9_]+(-[A-Za-z0-9_]+)*$` / `^job-[a-zA-Z0-9._-]{1,55}-[0-9a-f]{8}$`；`_is_ephemeral` 任一正则 match 返 True；`cleanup_stale_worktrees(manager, cutoff_hours)` 三层过滤 — **L1 命名**：`_is_ephemeral` False 跳；**L2 时态**：`current_session.worktree_name == name` 跳 + `mtime > cutoff` 跳；**L3 git 状态 fail-closed**：`read_worktree_head_sha is None` 跳 + `has_worktree_changes` 跳 + `has_unpushed_commits` 跳；通过的复用 `_remove_worktree` 或直接 `git worktree remove --force` + `sleep(0.1)` + `git branch -D`；返回清理数；`start_stale_cleanup_task(manager, interval, cutoff_hours)`：死循环 `await asyncio.sleep(interval)` → `cleanup_stale_worktrees` → 异常 `log.warning` 不抛。

- [ ] 完成

## T10: 包级 `__init__.py` 导出

- 影响文件: `mewcode/worktree/__init__.py`（导出 14 个公共符号 + `__all__`）
- 依赖任务: T1, T2, T3, T5, T8, T9
- 完成标准: 从 `changes` 导出 `Changes / CleanupResult / count_worktree_changes / has_worktree_changes`；从 `cleanup` 导出 `cleanup_stale_worktrees / start_stale_cleanup_task`；从 `manager` 导出 `WorktreeError / WorktreeManager`；从 `models` 导出 `Worktree / WorktreeSession`；从 `session` 导出 `load_worktree_session / save_worktree_session`；从 `slug` 导出 `flatten_slug / validate_slug`；`__all__` 列出 14 个名字按字母序。

- [ ] 完成

## T11: 实现 EnterWorktreeTool

- 影响文件: `mewcode/tools/enter_worktree.py`（`EnterWorktreeParams` @ 15-23；`EnterWorktreeTool` @ 26-65）
- 依赖任务: T1, T8
- 完成标准: `EnterWorktreeParams` 用 pydantic 定义，仅 `name: Optional[str]` 字段含描述；`EnterWorktreeTool`：`name = "EnterWorktree"` / `category = "command"` / `should_defer = True` / `params_model = EnterWorktreeParams`；`__init__(self, worktree_manager)`；`execute`：`get_current_session() is not None` → 返 `ToolResult(output="Already in a worktree session", is_error=True)` → 否则 `slug = params.name or f"wt-{secrets.token_hex(4)}"` → `validate_slug` 失败返错 → `manager.create(slug)` + `manager.enter(slug)` → 返回 `ToolResult(output=f"Created worktree at {session.worktree_path} on branch {wt.branch}. The session is now working in the worktree. Use ExitWorktree to leave mid-session, or exit the session to be prompted.")`。

- [ ] 完成

## T12: 实现 ExitWorktreeTool

- 影响文件: `mewcode/tools/exit_worktree.py`（`ExitWorktreeParams` @ 14-25；`ExitWorktreeTool` @ 28-110）
- 依赖任务: T3, T8
- 完成标准: `ExitWorktreeParams`：`action: str` 必填 + `discard_changes: Optional[bool] = None`；`ExitWorktreeTool`：`name = "ExitWorktree"` / `should_defer = True`；`execute`：`get_current_session() is None` → 返 "No-op: there is no active EnterWorktree session to exit. This tool only operates on worktrees created by EnterWorktree in the current session — it will not touch worktrees created manually or in a previous session. No filesystem changes were made."（`is_error=True`）；`action not in ("keep", "remove")` 返非法值；`action == "remove" and not discard` 时 `count_worktree_changes` → `uncommitted/new_commits > 0` 拼具体数（**单复数 file/files、commit/commits 正确**）→ `manager.exit(wt_name, action, discard_changes=discard)` → keep 返 "Your work is preserved at ... Session is now back in ..."，remove 返 "Exited and removed worktree at ..."。

- [ ] 完成

## T13: 实现 `/worktree` 本地命令

- 影响文件: `mewcode/commands/handlers/worktree.py`（`create_worktree_command` @ 11-49；`_handle_create` @ 52-85；`_handle_list` @ 88-110 附近；`_handle_enter` / `_handle_exit` / `_handle_status`）
- 依赖任务: T8
- 完成标准: `create_worktree_command(manager)` 返回 `Command(name="worktree", aliases=["wt"], type=CommandType.LOCAL)`；子命令解析 `create / list / enter / exit / status`，未知子命令报 "未知子命令: ..."；`_handle_create` 调 `manager.create + manager.enter` 并同步 `ctx.agent.work_dir`；`_handle_exit` 解析 `--remove` / `--discard` 标志映射到 `action / discard_changes`；`_handle_list` 列出 `manager.list_worktrees` 标当前；`_handle_status` 输出当前 session 路径和原始分支。

- [ ] 完成

## T14: 接入 AgentTool worktree 隔离

- 影响文件: `mewcode/tools/agent_tool.py`（`AgentToolParams` 含 `isolation` @ 27；`__init__` 接 `worktree_manager` @ 71/80；`execute` 解析 isolation @ 89-96；`_execute_with_worktree` @ 491-610）
- 依赖任务: T4, T8, T11
- 完成标准: `AgentToolParams` 含 `isolation: str | None = None` 和 `team_name: str | None = None`；`AgentTool.__init__` 多两个可选参数 `worktree_manager / team_manager`；`execute` 在 `p.team_name` 时走 teammate 分支，否则按 `definition.isolation == "worktree"` 分流 `_execute_with_worktree`；`_execute_with_worktree`：`worktree_manager is None` 报错 → `generate_worktree_name` 出 `agent-<8hex>` → `manager.create(wt_name, "HEAD")` → `notice = build_worktree_notice(parent_cwd, wt.path)` → `task = notice + "\n\n" + p.prompt` → 构造子 Agent `work_dir=wt.path` + `PathSandbox(wt.path)` → `run_to_completion(task)` → `manager.auto_cleanup(wt_name, wt.head_commit)` → `cleanup.kept` 时结果末尾拼 `[Worktree preserved at <path>, branch <branch>]`。

- [ ] 完成

## T15: 接入 app.py 启动装配

- 影响文件: `mewcode/app.py`（imports @ 82-84；worktree setup 段 @ 691-722；teardown @ 1602-1605）
- 依赖任务: T8, T9, T11, T12, T13, T14
- 完成标准:
  1. `WorktreeConfig` 注入 `symlink_directories / stale_cleanup_interval / stale_cutoff_hours`；
  2. `self.worktree_manager = WorktreeManager(repo_root=work_dir, file_cache=self.file_cache, symlink_directories=wt_cfg.symlink_directories)`；
  3. `add_cache_clear_callback` 加 skills 清理钩子；
  4. `restored = self.worktree_manager.restore_session()` 非 None 时 `self.agent.work_dir = restored.worktree_path`；
  5. `create_worktree_command(self.worktree_manager)` + `command_registry.register_sync`；
  6. `registry.register(EnterWorktreeTool(...))` + `registry.register(ExitWorktreeTool(...))`；
  7. `self._stale_cleanup_task = asyncio.create_task(start_stale_cleanup_task(self.worktree_manager, wt_cfg.stale_cleanup_interval, wt_cfg.stale_cutoff_hours))`；
  8. TeamManager 和 AgentTool 共用同一 `worktree_manager` 注入；
  9. teardown 时遍历 `worktree_manager.active.values()` 清理残留。

- [ ] 完成

## T16: 端到端验证

- 影响文件: 无（仅运行）
- 依赖任务: T1-T15
- 完成标准:
  - `ruff check mewcode/worktree mewcode/tools/enter_worktree.py mewcode/tools/exit_worktree.py` 通过；
  - `pytest tests/test_worktree.py -v` 通过（含 `TestValidateSlug` / `TestFlattenSlug` / `TestSessionPersistence` / `TestWorktreeManager` / `TestChangeDetection` / `TestReadWorktreeHeadSha` 等组）；
  - **路径 A — 工具直接驱动**：主 Agent 调 `EnterWorktree({name: "demo"})` 创建 worktree → 在 worktree 里 `WriteFile + Bash("git commit ...")` → `ExitWorktree({action: "remove"})` 被变更保护拒绝并列出具体数 → `ExitWorktree({action: "remove", discard_changes: true})` 强删成功；
  - **路径 B — 子 Agent 自动隔离**：主 Agent 在主目录 `WriteFile witness.txt = "original content from main agent"` → 调 `Agent({subagent_type: "<声明 isolation worktree 的类型>", prompt: "把 witness.txt 改成 ..."})` → 验证主目录 `witness.txt` 内容不变；`.mewcode/worktrees/agent-*/witness.txt` 是修改后版本；若有 commit → 结果末尾出现 `[Worktree preserved at ..., branch worktree-agent-...]`。

- [ ] 完成

## 进度

- [ ] T1 / [ ] T2 / [ ] T3 / [ ] T4 / [ ] T5 / [ ] T6 / [ ] T7 / [ ] T8 / [ ] T9 / [ ] T10 / [ ] T11 / [ ] T12 / [ ] T13 / [ ] T14 / [ ] T15 / [ ] T16
