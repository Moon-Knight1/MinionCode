# ch14: Worktree Checklist

> 所有条目可勾选、可观测。验收方式写在条目后面括号中。验收：已通过验证的项均勾选。

## 1. 实现完整性

- [ ] 常量 `MAX_SLUG_LENGTH = 64` 在 `mewcode/worktree/slug.py:5` 定义
- [ ] 函数 `validate_slug` 在 `mewcode/worktree/slug.py:9-24` 含空名、长度、空段、`.` / `..`、非法段五类错误分类
- [ ] 函数 `flatten_slug` 在 `mewcode/worktree/slug.py:27-28` 把 `/` 替换成 `+`；分支名由调用方拼 `f"worktree-{flat_slug}"`
- [ ] dataclass `Worktree` 在 `mewcode/worktree/models.py:7-14` 含 6 字段（`name / path / branch / based_on / head_commit / created`）
- [ ] dataclass `WorktreeSession` 在 `mewcode/worktree/models.py:17-25` 含 7 字段，`session_id` / `hook_based` 有默认值
- [ ] dataclass `Changes` 在 `mewcode/worktree/changes.py:25-28`，`CleanupResult` 在 `mewcode/worktree/changes.py:59-63`
- [ ] 函数 `count_worktree_changes` 在 `mewcode/worktree/changes.py:31-51`，git 子进程异常时把对应计数置 1（**fail-closed**）
- [ ] 函数 `has_worktree_changes` 在 `mewcode/worktree/changes.py:54-56`，`has_unpushed_commits` 在 `:66-74` git 失败默认返 True
- [ ] 字符串 `WORKTREE_NOTICE_TEMPLATE` 在 `mewcode/worktree/integration.py:9-20` 含 `{parent_cwd}` / `{wt_path}` 占位 + "re-read files before editing" 关键句
- [ ] 函数 `generate_worktree_name` 在 `mewcode/worktree/integration.py:23-24` 用 `secrets.token_hex(4)` 出 `agent-` 开头 8 hex 名字
- [ ] 函数 `save_worktree_session` 在 `mewcode/worktree/session.py:18-36`，`session is None` 时写 `"{}"`（清空）
- [ ] 函数 `load_worktree_session` 在 `mewcode/worktree/session.py:39-58` 容忍文件缺失、JSON 损坏、空 dict、缺字段全部返 `None`
- [ ] 常量 `LOCAL_CONFIG_FILES` 在 `mewcode/worktree/setup.py:12-15` 含 `settings.local.json` + `.env`
- [ ] 函数 `perform_post_creation_setup` 在 `mewcode/worktree/setup.py:18-29` 依序调四项 A/B/C/D
- [ ] 函数 `_copy_ignored_files` 在 `mewcode/worktree/setup.py:85-131` 单文件失败 `continue` 不中断
- [ ] 类 `WorktreeManager` 在 `mewcode/worktree/manager.py:35-328` 持有 `_lock=asyncio.Lock() / active / current_session`
- [ ] 静态方法 `WorktreeManager.read_worktree_head_sha` 在 `mewcode/worktree/manager.py:84-128` 完整链路（`.git → gitdir → commondir → HEAD → loose/packed-refs`），失败返 `None`
- [ ] 方法 `WorktreeManager._run_git` 在 `mewcode/worktree/manager.py:68-78` 强制 `env=GIT_ENV + stdin=DEVNULL + timeout=60`
- [ ] 方法 `WorktreeManager.create` 在 `mewcode/worktree/manager.py:134-186` 实现"快速恢复 → 创建路径"二选一，使用 `-B` 大写参数
- [ ] 方法 `WorktreeManager.exit` 在 `mewcode/worktree/manager.py:218-243` 在 `action="remove" and not discard_changes` 时跑变更保护
- [ ] 方法 `WorktreeManager._remove_worktree` 在 `mewcode/worktree/manager.py:249-260` 含 `await asyncio.sleep(0.1)` 等 git lockfile 释放
- [ ] 方法 `WorktreeManager.auto_cleanup` 在 `mewcode/worktree/manager.py:266-275` 脏返 `kept=True` + path/branch，干净返 `kept=False`
- [ ] 方法 `WorktreeManager.restore_session` 在 `mewcode/worktree/manager.py:291-310` 在 `read_worktree_head_sha is None` 时反向 `save_worktree_session(None)` 清脏
- [ ] 变量 `EPHEMERAL_PATTERNS` 在 `mewcode/worktree/cleanup.py:16-22` 含五个正则（agent-a / wf_ / wf- / bridge- / job-）
- [ ] 函数 `cleanup_stale_worktrees` 在 `mewcode/worktree/cleanup.py:29-81` 三层过滤顺序固定（L1 命名 → L2 时态 → L3 git 状态）
- [ ] 函数 `start_stale_cleanup_task` 在 `mewcode/worktree/cleanup.py:84-96` 死循环 + 异常 warning 不抛
- [ ] 类 `EnterWorktreeTool` 在 `mewcode/tools/enter_worktree.py:26-65`，`should_defer = True` + `params_model = EnterWorktreeParams`
- [ ] 类 `ExitWorktreeTool` 在 `mewcode/tools/exit_worktree.py:28-110`，`should_defer = True`
- [ ] `ExitWorktreeTool.execute` 在 `mewcode/tools/exit_worktree.py:63-84` 单复数 file/files、commit/commits 正确处理

## 2. 接入完整性（必查，杜绝死代码）

- [ ] `grep -rn "EnterWorktreeTool" --include="*.py" mewcode/` 在 `mewcode/app.py:711-713` 找到 import + 注册
- [ ] `grep -rn "ExitWorktreeTool" --include="*.py" mewcode/` 在 `mewcode/app.py:712-714` 找到 import + 注册
- [ ] `grep -rn "WorktreeManager" --include="*.py" mewcode/` 至少命中 `mewcode/app.py:694`、`mewcode/tools/agent_tool.py`、`mewcode/teams/manager.py`、`mewcode/commands/handlers/worktree.py`
- [ ] `grep -rn "restore_session" --include="*.py" mewcode/` 在 `mewcode/app.py:704` 找到启动恢复调用
- [ ] `grep -rn "start_stale_cleanup_task" --include="*.py" mewcode/` 在 `mewcode/app.py:716-722` 找到 `asyncio.create_task` 包裹
- [ ] `grep -rn "build_worktree_notice" --include="*.py" mewcode/` 在 `mewcode/tools/agent_tool.py:544` 找到 prompt 拼接调用
- [ ] `grep -rn "generate_worktree_name" --include="*.py" mewcode/` 在 `mewcode/tools/agent_tool.py:535` 找到调用
- [ ] `grep -rn "auto_cleanup" --include="*.py" mewcode/` 在 `mewcode/tools/agent_tool.py:604` 找到子 Agent 完成后清理调用
- [ ] `grep -rn "count_worktree_changes" --include="*.py" mewcode/` 在 `mewcode/tools/exit_worktree.py:64` 和 `mewcode/worktree/manager.py:229` 找到调用
- [ ] `grep -rn "has_worktree_changes" --include="*.py" mewcode/` 在 `mewcode/worktree/cleanup.py:59` 和 `mewcode/worktree/manager.py:271` 找到调用
- [ ] `grep -rn "create_worktree_command" --include="*.py" mewcode/` 在 `mewcode/app.py:708` 找到 `/worktree` 命令注册
- [ ] `grep -rn "_execute_with_worktree" --include="*.py" mewcode/` 在 `mewcode/tools/agent_tool.py:96` 和 `:491` 找到分流入口

## 3. 编译与测试

- [ ] `ruff check mewcode/worktree mewcode/tools/enter_worktree.py mewcode/tools/exit_worktree.py mewcode/commands/handlers/worktree.py` 无报错
- [ ] `pytest tests/test_worktree.py -v` 通过（含 `TestValidateSlug` / `TestFlattenSlug` / `TestSessionPersistence` / `TestIntegrationHelpers` / `TestWorktreeManager` / `TestChangeDetection` / `TestReadWorktreeHeadSha` 等组）
- [ ] `python -c "from mewcode.worktree import WorktreeManager, validate_slug, flatten_slug; print('ok')"` 无 import 错误
- [ ] `python -m mypy mewcode/worktree` 或 `pyright mewcode/worktree` 无新增 type 错误

## 4. 端到端验证

- [ ] **路径 A — 工具直接驱动**：用户对主 Agent 说"用 EnterWorktree 工具创建一个名叫 demo 的工作树" → LLM 调 `EnterWorktree({name: "demo"})` → 返回 `Created worktree at .../.mewcode/worktrees/demo on branch worktree-demo`；让 Agent 在 worktree 里创建 `hello.txt` 并 `git commit`；让 Agent 调 `ExitWorktree({action: "remove"})` → 因有未推送 commit 被变更保护拒绝，错误文本包含具体 `1 commit` 或 `N commits`；`ExitWorktree({action: "remove", discard_changes: true})` 强删成功；`ls .mewcode/worktrees/` 看到 `demo/` 已消失。
- [ ] **路径 B — 子 Agent 自动隔离**：用户让主 Agent 在主目录建 `witness.txt`（内容 "original content from main agent"）→ 调 `Agent({subagent_type: "<声明 isolation worktree 的类型>", description: "...", prompt: "把 witness.txt 改成 \"modified by isolated worker\"，然后 git 提交"})`；验证 `cat witness.txt` 主目录内容仍是 "original ..."；`cat .mewcode/worktrees/agent-*/witness.txt` 是修改后版本；若子 Agent 有 commit → 结果末尾出现 `[Worktree preserved at ..., branch worktree-agent-...]`；若无修改 → worktree 自动清理（`.mewcode/worktrees/` 下 `agent-*` 目录消失）。
- [ ] **持久化与 crash 恢复**：TUI 里 `EnterWorktree({name: "crashtest"})` 创建 worktree → `Ctrl+C` 杀进程 → `cat .mewcode/worktree_session.json` 文件仍在并含 crashtest 会话；重启 MewCode → 启动期间 `restore_session` 把 session 写回；下一次工具调用时 `get_current_session()` 非 None，且 `agent.work_dir` 已切到 worktree 路径。
- [ ] **变更保护单复数**：在 worktree 里建 1 个未提交修改 → `ExitWorktree({action: "remove"})` 返回 `"1 uncommitted file"`；建 2+ 个修改 → 返回 `"N uncommitted files"`（注意单复数）；同样验证 commit 数 `"1 commit"` / `"N commits"`。
- [ ] **后台清理保守不删**：手动在 `.mewcode/worktrees/agent-aabcdef1/` 下建一个有未推送 commit 的目录（mtime 设为过期前）→ 等 cleanup loop 跑一轮（或手动 `await cleanup_stale_worktrees(manager, 1)` 测试）→ 该目录仍保留（L3 fail-closed 的 `has_unpushed_commits` 拦住）。
- [ ] **会话级 enter 时清理 FileCache**：在主仓里读一个文件触发 FileCache 命中 → `EnterWorktree` → 验证 `file_cache` 被清空（`len(file_cache) == 0`），保证后续读 worktree 不复用主仓的缓存。
- [ ] **`/worktree` 本地命令**：`/worktree create demo` 创建并进入 → `/worktree status` 显示当前 session → `/worktree list` 列出含 demo → `/worktree exit --remove --discard` 强删。

## 5. 文档

- [ ] `docs/python/ch14/spec.md` 已按 ch12/ch13 风格写完（F1-F17 + N1-N8，无 file:line 代码标注）
- [ ] `docs/python/ch14/tasks.md` 已写，16 个 T 全部勾完（T1-T16）
- [ ] `docs/python/ch14/checklist.md` 已写并逐项验收
- [ ] commit 信息标注 `ch14`，新增代码的调用链已在 PR 描述或 commit message 里说明
