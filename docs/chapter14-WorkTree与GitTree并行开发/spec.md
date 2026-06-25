# ch14: Worktree Spec

## 1. 背景

SubAgent 隔离了消息、权限、工具结果缓存，但所有子 Agent 仍然共享同一个工作目录——两个子 Agent 并发改同一个文件会互相覆盖。Git 分支不解决这个问题：分支只是时间维度的快照，同一时刻整个仓库仍然只有一份 working tree，切换分支会动所有文件的修改时间触发不必要的全量重编。多 Agent 并行要的是空间维度的隔离：同时存在多份独立的 working tree，每份对应不同分支，但共享同一个 `.git`。Git Worktree 提供的就是这个能力。这一章把它接进 MewCode，让主 Agent 和每个子 Agent 都能拥有独立的文件视图。

## 2. 目标

把 worktree 做成两层 API：会话级让 LLM 通过 `EnterWorktree` / `ExitWorktree` 工具自主进出 worktree，Agent 级让 SubAgent 通过 `isolation: "worktree"` 声明自动获得独立 worktree。底层共用一个 `WorktreeManager` 提供创建/快速恢复路径和"创建后设置"管线（本地配置复制 / git hooks 配置 / 大目录软链接 / `.worktreeinclude` 文件复制）。叠加 fail-closed 变更检测（无变更才允许清掉、有变更默认保留）和孤儿 worktree 的后台过期清理 task，保证既不丢用户工作、又不让磁盘堆积。

## 3. 功能需求

- F1: worktree 名称（slug）安全校验：限定字符集 `^[a-zA-Z0-9._-]+$`、长度上限 64、按 `/` 切段、显式拒绝 `.` / `..` 段和空段，校验失败返回带原因的错误字符串；任何 git 命令或路径拼接之前先跑。
- F2: slug 到路径和分支的映射：`flatten_slug` 把 `/` 替换为 `+`，避免嵌套 slug 导致目录或分支命名冲突（Git D/F conflict）；分支统一加 `worktree-` 前缀，方便从 `git branch` 输出里识别 MewCode 创建的。
- F3: 快速恢复路径：worktree 目录已存在时 `read_worktree_head_sha` 纯文件系统读 `.git` 指针 → `gitdir` → `commondir` → `HEAD` → loose ref / packed-refs，跳过 git 子进程；任一步失败返回 `None`，调用方回退到完整创建路径。
- F4: git 子进程统一安全壳：所有 git 调用关闭终端密码提示（`GIT_TERMINAL_PROMPT=0`）、屏蔽 `GIT_ASKPASS=""`、`stdin=subprocess.DEVNULL`，绝不挂起等待用户输入；统一 `timeout=60`，失败返回 `CompletedProcess` 而不是抛异常。
- F5: 创建/恢复主入口 `WorktreeManager.create`：先做 slug 校验和 `active` 字典重名检查，命中已存在目录走快速恢复（不重跑创建后设置），未命中 `os.makedirs(worktree_dir, exist_ok=True)` → `git worktree add -B worktree-<flat> <path> <base_branch>`（大写 `-B` 容忍上次未清的孤儿分支），默认 `base_branch="HEAD"`。
- F6: 创建后设置四项 `perform_post_creation_setup`：依次执行 — A `_copy_local_configs` 复制 `LOCAL_CONFIG_FILES` 里列出的 `settings.local.json` / `.env`（不存在静默跳过）；B `_setup_git_hooks` 优先 `<repo>/.husky` 回退 `<repo>/.git/hooks`，找到目录后在 worktree 里跑 `git config core.hooksPath`；C `_create_symlinks` 从 `WorktreeManager.symlink_directories` 读列表，逐个 `os.symlink(src, dst)`，错误日志吞掉不抛；D `_copy_ignored_files` 读 `<repo>/.worktreeinclude`（跳空行和 `#`）→ `git ls-files --others --ignored --exclude-standard --directory` 列出 gitignored → `fnmatch` 筛选 → 命中的 `shutil.copy2`。
- F7: 会话级 API 三件套：`create`（先快速恢复，未命中走 git add + 创建后设置）、`enter`（清缓存 + 记 `original_cwd` / `original_branch` / `original_head_commit` + 写 `current_session` + 持久化）、`exit`（变更保护 + 清缓存 + 清单例 + 删持久化，`action="remove"` 时调 `_remove_worktree`）。
- F8: 会话持久化：`save_worktree_session` 把 `WorktreeSession` 7 字段 dump 成 `<repo>/.mewcode/worktree_session.json`；`session=None` 时写 `"{}"`（等价清空）；`load_worktree_session` 容忍文件缺失、JSON 损坏、空 dict、缺字段全部返 `None` 并 warning 日志。
- F9: 启动恢复：`WorktreeManager.restore_session` 读持久化文件 → `read_worktree_head_sha` 验证 worktree 路径仍然存在 → 命中时把 `Worktree` 写回 `active` 字典 + `current_session`；HEAD SHA 读不到则反向调用 `save_worktree_session(None)` 清掉脏文件。
- F10: 自动清理 API `auto_cleanup(name, head_commit)`：调 `has_worktree_changes` 看脏不脏，干净直接 `_remove_worktree` 返 `CleanupResult(kept=False)`，脏返 `CleanupResult(kept=True, path, branch)`；供 SubAgent 完成后调用。
- F11: SubAgent 集成：`AgentTool._execute_with_worktree` 当 `definition.isolation == "worktree"` 时，调 `generate_worktree_name` 生成 `agent-<8hex>` slug → `worktree_manager.create(wt_name, "HEAD")` → `build_worktree_notice(parent_cwd, wt.path)` 拼接到 prompt 前 → `sub_agent.work_dir = wt.path` + `PathSandbox(wt.path)` 锁定权限边界。
- F12: 子 Agent 完成后决策：`auto_cleanup(wt_name, wt.head_commit)` 干净 → 自动清理 worktree，脏 → 保留并在结果末尾附 `[Worktree preserved at <path>, branch <branch>]` 给主 Agent review。
- F13: 变更保护：`ExitWorktreeTool` 在 `action="remove"` 且 `discard_changes` 不为 True 时调 `count_worktree_changes`，`uncommitted > 0 or new_commits > 0` 拒绝并把具体数（file/files 和 commit/commits 单复数正确）回吐给 LLM。
- F14: 变更检测 fail-closed：`count_worktree_changes` 的 `_run_git` 抛 `SubprocessError / OSError / ValueError` 时把对应计数置 1（按"有变更"处理）；`has_unpushed_commits` 在 git 失败时返 `True`，绝不在 git 命令失败时清掉用户工作。
- F15: LLM Tool 暴露：`EnterWorktreeTool`（input 仅可选 `name`，已有 session 时拒绝 "Already in a worktree session"）和 `ExitWorktreeTool`（input `action` 必填，`discard_changes` 可选，无 session 时返回 "No-op: there is no active EnterWorktree session..."）；两个工具 `should_defer = True`，由主 Agent loop 在工具批次结束时统一执行。
- F16: 临时 worktree 命名模式：用前缀化的固定模式区分自动产物（`agent-<8hex>` / `wf_<8hex>-<3hex>-<n>` / `wf-<n>` / `bridge-<id>` / `job-<slug>-<8hex>`）和用户手动命名；正则在 `EPHEMERAL_PATTERNS` 集中维护，便于新增来源时统一加入。
- F17: 后台过期清理三层过滤：`cleanup_stale_worktrees` 周期扫 `worktree_dir`，依次过滤 —— L1 命名模式（用户起名的永不删，廉价）→ L2 时态（跳过当前 session 占用的 + `info.stat().st_mtime > cutoff`）→ L3 git 状态 fail-closed（`has_worktree_changes` 或 `has_unpushed_commits` 任一为 True 都跳过）；通过的删 worktree + 删分支。

## 4. 非功能需求

- N1: `WorktreeManager` 用 `asyncio.Lock` 保护 `create`，并发创建同名 worktree 互斥；`active` 字典和 `current_session` 用同一锁覆盖。
- N2: 任何路径的 worktree 删除（会话级 exit / Agent 级 auto_cleanup / 后台清理）都要保证当前 cwd 不在 worktree 内（`_run_git` 的 `cwd` 缺省走 `repo_root`），否则 `git worktree remove` 会失败。
- N3: `git worktree remove` 和 `git branch -D` 之间必须 `await asyncio.sleep(0.1)` 等 git lockfile 释放，否则 branch 删除会偶发失败。
- N4: `restore_session` 在 HEAD SHA 读不到时必须主动 `save_worktree_session(None)` 清脏文件，否则下次启动会反复尝试恢复同一个已损坏的 session。
- N5: 三层过滤的执行顺序固定：先廉价的命名模式 → 再时态判断 → 最后贵的 git 检查；任何一层判定保留都立即 `continue`，不进入下一层。
- N6: 创建后设置的四项里软链接和 `.worktreeinclude` 复制是 best-effort —— 单文件失败只 `log.warning` 不抛，保证主路径鲁棒。
- N7: 变更保护的错误信息必须包含具体数字（N file/files + M commit/commits）和单复数语法正确，让 LLM 能据此判断要不要强删；不能只回 "has changes" 这种空话。
- N8: worktree 子系统不假设统一日志层存在，所有创建/退出/清理的信息通过工具结果文本传达；这同时是给 LLM 的运行时反馈。日志只用 `logging.getLogger(__name__)`。

## 5. 设计概要

- 核心数据结构（`mewcode/worktree/models.py`）:
  - `Worktree`：`name / path / branch / based_on / head_commit / created`（dataclass，活跃 worktree 注册项）。
  - `WorktreeSession`：`original_cwd / worktree_path / worktree_name / original_branch / original_head_commit / session_id / hook_based`（dataclass，会话级单例，序列化到 JSON）。
  - `Changes`：`uncommitted / new_commits`（dataclass，变更计数）。
  - `CleanupResult`：`kept / path / branch`（dataclass，Agent 级自动清理返回值）。
  - `WorktreeManager`：持有 `repo_root / file_cache / symlink_directories / worktree_dir / _lock / active / current_session`，是所有 worktree 操作的入口。
- 主流程:
  - **会话级 Enter**：`EnterWorktreeTool.execute` → guard `get_current_session() != None` → `validate_slug` → `WorktreeManager.create(slug)`（自动走快速恢复或 add + setup）→ `WorktreeManager.enter(slug)` → 返回带路径和分支的 Tool 文本。
  - **会话级 Exit**：`ExitWorktreeTool.execute` → guard 无 session → 若 `action="remove"` 且未 `discard_changes` 跑 `count_worktree_changes` → `WorktreeManager.exit(name, action, discard_changes)` → action=remove 时调 `_remove_worktree`（git worktree remove → sleep 0.1 → git branch -D）。
  - **Agent 级隔离**：`AgentTool.execute` 看到 `definition.isolation == "worktree"` → `_execute_with_worktree` → `generate_worktree_name` 出 `agent-<8hex>` → `worktree_manager.create(wt_name, "HEAD")` → `build_worktree_notice` 拼 prompt 前缀 → `sub_agent.work_dir = wt.path` + `PathSandbox(wt.path)` → 跑完调 `auto_cleanup`。
  - **后台过期清理**：`app.py` 启动 `asyncio.create_task(start_stale_cleanup_task(...))` → 死循环 `await asyncio.sleep(interval)` → `cleanup_stale_worktrees` 三层过滤 → 通过的删。
- 调用链（模块层级）:
  - `mewcode/app.py` 启动 → `WorktreeManager(repo_root=...)` 构造 → `restore_session` → 注册 `EnterWorktreeTool` / `ExitWorktreeTool` / `create_worktree_command` → `asyncio.create_task(start_stale_cleanup_task)`。
  - LLM Enter/Exit → 工具 registry → `mewcode/worktree/manager.py` 会话级 API。
  - `AgentTool` → 看到 isolation worktree → `WorktreeManager.create` + `build_worktree_notice` → 子 Agent 跑完 → `auto_cleanup`。
- 与其他模块的交互:
  - 依赖 `mewcode/tools`（注册两个工具）、`mewcode/agents`（隔离分流）、`mewcode/teams`（TeamManager 共用同一 manager）、`mewcode/commands`（`/worktree` 子命令）、`mewcode/cache`（FileCache 清理钩子）；底层只依赖 `asyncio` + `subprocess`（git）+ 标准库（`re` / `json` / `pathlib` / `secrets` / `fnmatch` / `shutil`）+ `pydantic`（工具 schema）。
  - 不依赖 `mewcode/memory` / `mewcode/prompt`。

## 6. Out of Scope

- 不实现非 git VCS 适配（hg / jj / sapling 等），所有 worktree 操作 hardcode 走 git 子命令
- 不实现 sparse checkout / partial clone 优化，大型 mono-repo 优化推到后续
- 不实现 `--worktree` / `--worktree --tmux` CLI 启动快速路径（涉及 tmux / iTerm2 子系统，留给 ch15）
- 不实现 PR fetch 或 pull request 头引用解析（远端协作场景）
- 不实现 prepare-commit-msg hook 注入 commit attribution（商业 feature 场景）
- 不实现 FindCanonicalGitRoot 穿透 commondir 的独立工具（Python 版仅以 `repo_root` 注入为主，多级嵌套 worktree 留给后续）
- 不引入第三方 gitignore 库（`fnmatch` 简化匹配够用）
- 团队成员（teammate）路径的 worktree 自动清理推到 ch15 收尾，本章 teammate 路径只创建并隔离、不负责清理

## 7. 完成定义

见 [checklist.md](checklist.md)，所有条目勾上即完成。
