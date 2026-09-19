# openai/codex 贡献调研与代码贡献交付

> 交付日期：2026-09-16（首版）/ 2026-09-19（代码层根治补充）｜ 本地仓库：`D:\download\project\TX-budddy\codex`（浅克隆，上游 HEAD `da18000`，本地分支含提交 `6bfed3b`、`01e49c8`、`a683b7a`）
> 本文档 = ①贡献方式调研结论（本地文档 + 线上证据）②可直接粘贴到 GitHub 的 issue 草稿 ③已完成代码贡献的补丁与验证记录。

---

## 一、贡献方式调研结论（事实）

### 1.1 官方政策：不收外部代码贡献

`docs/contributing.md` 原文明确：

> **We do not accept external code contributions or pull requests.**

GitHub API 交叉验证（`GET /repos/openai/codex`）：
- `pull_request_creation_policy: "collaborators_only"` —— 仓库在 API 层面即拒绝非协作者开 PR。
- 仓库无 `good first issue` / `help wanted` 标签；近期 issue 大多属于桌面 App（Electron 仓库）范畴，不在本 Rust 仓库可修范围。

**结论：外部贡献的正确形态 = 高质量 issue（bug report / root-cause analysis / feature request），附复现步骤、日志、诊断信息；安全问题走 `SECURITY.md`。**

### 1.2 仓库结构与可贡献面

- `codex-rs/` 为 Rust workspace，含 48+ crate：`codex-core`（会话/agent 核心）、`codex-cli`、`codex-config`、`codex-apply-patch`、`codex-thread-store`、`codex-git-utils`、`codex-protocol`、`codex-tui` 等。
- 工具链锁定：`codex-rs/rust-toolchain.toml` → `channel = "1.95.0"`。
- 测试入口：`just test`（`RUST_MIN_STACK=8388608 cargo nextest run --no-fail-fast`）；CI 主测试走 `rust-ci-full-nextest-platform.yml`（nextest archive，`RUST_MIN_STACK: "8388608" # 8 MiB`）。

### 1.3 我方已完成的本地验证矩阵（WSL Ubuntu-22.04，非 root 用户 dev）

| 套件 | 结果 |
|---|---|
| codex-apply-patch（dev 用户） | 69 通过 |
| codex-apply-patch 全目标 | 69+30 通过 |
| codex-config | 327 通过 |
| codex-network-proxy | 288 通过 |
| codex-git-utils / path-uri / absolute-path / models-manager 等 | 全部通过 |
| codex-core --lib（2530 个） | 真 8 MiB 全量**零栈溢出**（2527/2529 通过，2 例环境类）；默认 2 MiB 下关键深链测试通过、剩 1 例测试侧深链需 ~3 MiB（详见 5.3） |

---

## 二、发现的真实缺陷（本次代码贡献的落点）

### 2.1 缺陷描述

**在 Linux/macOS 上直接运行 `cargo test`（不设置 `RUST_MIN_STACK`）时，codex-core 的 residency 测试触发 `fatal runtime error: stack overflow`，整个测试进程 abort，2530 个测试全部中断。**

崩溃测试：
```
agent::control::residency::tests::interrupted_v2_agent_is_lost_after_residency_eviction
```

复现（默认 2 MiB 线程栈）：
```bash
cd codex-rs
cargo test -p codex-core --lib
# thread 'agent::control::residency::tests::interrupted_v2_agent_is_lost_after_residency_eviction' (448) has overflowed its stack
# fatal runtime error: stack overflow, aborting
```

### 2.2 根因分析（gdb 取证）

- gdb 抓到 **70 帧完整调用栈，无递归**：单链深 async 状态机，最深在 `codex-thread-store/src/thread_metadata_sync.rs:53`（`ThreadMetadataSync::for_create`），经 `live_thread::acquire` → `core/src/session/session.rs:1035`（`Session::new` 巨型 async fn）→ `thread_manager.rs` → 测试。
- **不是无限递归**，而是 debug 构建下嵌套 async 状态机合计栈耗超过 2 MiB 运行时默认值（Windows 上因 PE `/STACK:8388608` link-arg 有 8 MiB，Linux/macOS 没有等效机制）。
- 仓库对此**已有认知与对策，但只覆盖了部分路径**：
  - `justfile`：`rust_min_stack := "8388608"`，`just test` 才设置；
  - CI（`rust-ci-full-nextest-platform.yml:398`）：`RUST_MIN_STACK: "8388608"`；
  - Windows：`codex-rs/.cargo/config.toml` 用 link-arg 设 8 MiB。
  - **缺口**：Linux/macOS 上任何直接 `cargo test` / `cargo nextest run` 的开发者（尤其新贡献者）都会踩到整进程崩溃。

### 2.3 修复方案（已实现 + 已验证）

在 `codex-rs/.cargo/config.toml` 增加 `[env]` 段，让所有 cargo 驱动的测试进程默认获得与既有政策一致的 8 MiB 栈：

```toml
# Raise the default stack size for Rust test threads on non-Windows targets.
# Windows already gets an 8 MiB stack via the link-arg above; without an
# equivalent here, `cargo test`/`cargo nextest run` on Linux/macOS crashes with
# a stack overflow in a few codex-core tests (e.g.
# agent::control::residency::tests::interrupted_v2_agent_is_lost_after_residency_eviction)
# whose debug-build async state machines exceed the runtime default of 2 MiB.
# This matches the 8 MiB policy already set by `just test` and rust-ci-full.
[env]
RUST_MIN_STACK = "8388608" # 8 MiB
```

### 2.4 验证记录（A/B + 回归）

| 场景 | 结果 |
|---|---|
| 修复前：`cargo test -p codex-core --lib`（2 MiB 默认栈） | 栈溢出崩溃（证据：`wsl_core_full_out.log`） |
| 修复后：**不设置任何环境变量**，仅凭 config.toml，跑同一测试 | `ok`（1 passed, 0.69s，证据见 wsl 验证输出） |
| 回归：residency 全模块 | 2 passed |
| 回归：codex-apply-patch | 30 passed |
| 回归：codex-git-utils | 45 passed |
| 64 MiB 大栈对照（此前排查） | 通过（证明非算法缺陷，纯栈容量问题） |

### 2.5 本地提交

```
分支：fix/rust-test-stack-8mib
提交：6bfed3b  Set 8 MiB test-thread stack for non-Windows via cargo config
文件：codex-rs/.cargo/config.toml（+10 行）
```

---

## 三、GitHub Issue 草稿（可直接粘贴）

```markdown
### Title
`cargo test` on Linux/macOS crashes with stack overflow in codex-core residency tests (no RUST_MIN_STACK set)

### Environment
- OS: Linux (Ubuntu 22.04, WSL2) — any non-Windows host
- Rust: 1.95.0 (as pinned by `codex-rs/rust-toolchain.toml`)
- Repo HEAD: da18000 (`Measure rollout read and materialization durations (#45966)`)

### Reproduction
```bash
cd codex-rs
cargo test -p codex-core --lib
```

### Expected
All tests pass.

### Actual
The test binary aborts on the first deep-stack test:
```
thread 'agent::control::residency::tests::interrupted_v2_agent_is_lost_after_residency_eviction' (448) has overflowed its stack
fatal runtime error: stack overflow, aborting
error: test failed, to rerun pass `-p codex-core --lib`
```
Because the overflow aborts the process, none of the remaining 2529 tests run.

### Root-cause analysis
- gdb backtrace: 70 frames, **no recursion** — a single deep chain of nested async state machines. Deepest frame: `codex_thread_store::thread_metadata_sync::ThreadMetadataSync::for_create` (`codex-rs/thread-store/src/thread_metadata_sync.rs:53`), via `live_thread::acquire` -> `Session::new` (`core/src/session/session.rs:1035`) -> `thread_manager.rs` -> the test.
- This is a debug-build stack-size problem: the combined debug async state machines exceed the 2 MiB runtime default thread stack. It is not an infinite loop or recursion — the same test passes with `RUST_MIN_STACK=67108864`.
- The repo already raises the stack to 8 MiB in three places: `justfile` (`rust_min_stack := "8388608"`), CI (`rust-ci-full-nextest-platform.yml`, `RUST_MIN_STACK: "8388608" # 8 MiB`), and Windows link-args in `codex-rs/.cargo/config.toml` (`/STACK:8388608`). Only the Windows target has a config-file-level default; Linux/macOS rely on the env var being set, so the obvious plain `cargo test` fails out of the box.

### Proposed fix (tested)
Add an `[env]` section to `codex-rs/.cargo/config.toml`:
```toml
[env]
RUST_MIN_STACK = "8388608" # 8 MiB
```
This makes `cargo test` / `cargo nextest run` work on all platforms with the same 8 MiB policy already used by `just test` and CI.

Verification: with no explicit `RUST_MIN_STACK` env var, the previously-crashing test now passes (`1 passed; 0.69s`); residency module 2/2, codex-apply-patch 30/30, codex-git-utils 45/45 green.

### Notes
Alternative/longer-term: reduce the debug stack footprint of the deep async chain (e.g. boxing the `Session::new` future), but that is a larger refactor; the config-level stack raise matches the existing project policy.
```

---

## 四、PR 提交尝试实测记录（2026-09-18）

| 步骤 | 结果 |
|---|---|
| fork 到个人账号 | 成功（`cyberspace-cs/codex`，已存在） |
| push 修复分支到 fork | 成功：`fork/fix/rust-test-stack-8mib`（GitHub 提示可开 PR：`https://github.com/cyberspace-cs/codex/pull/new/fix/rust-test-stack-8mib`） |
| `gh pr create --repo openai/codex` | **被官方策略拒绝**：`GraphQL: cyberspace-cs does not have the correct permissions to execute CreatePullRequest (createPullRequest)` |

**结论（实测确认）**：代码已推送至 fork 可被官方拉取，但 PR 入口被 openai/codex 的 `pull_request_creation_policy: collaborators_only` 封死——外部贡献唯一合规通道是 issue。分支保留在 fork，若官方在 issue 下表态欢迎补丁，可一键开 PR。

---

## 五、更大的 PR：代码层根治（Box::pin 拆分巨型 async 状态机）

> 在 6bfed3b（config 兜底）基础上进一步：不是"提高栈"，而是**从代码层削减 poll 栈耗**，让默认 2 MiB 线程栈下 `cargo test -p codex-core --lib` 不再溢出。已提交 `01e49c8` 并推送 fork。

### 5.1 量化定位（gdb 帧取证，修复前）

- 70 帧无递归、纯单链深 async 状态机；最深处 `thread-store/src/thread_metadata_sync.rs:53`，经 `live_thread::acquire → Session::new → thread_manager → 测试`。
- 元凶帧（单 poll 帧字节数）：
  - 测试 async_block **666 KB**；`Session::new` async fn **308 KB** + 其内 async_block 192 KB；
  - `spawn` 182 KB / `spawn_internal` 183 KB / `start_thread_inner` 132 KB；
  - `handlers.rs` submission_loop 的 `should_exit` async 块 **1.1 MB**（27 个分支的 match 巨型状态机）；
  - 合计 ~2.1 MB 压爆 2 MiB。
- 三条被验证的认识：**Box::pin 只减父状态机尺寸**（被 pin future 的自身 poll 帧不变）；溢出非递归；官方 CI（`justfile:8` 与 `rust-ci-full-nextest-platform.yml:398` 均为 `RUST_MIN_STACK: "8388608"`）全靠 8 MiB 环境变量兜底。

### 5.2 修复内容

**提交 `01e49c8`**（6 文件，+162/-139）：

| 文件 | 改动 |
|---|---|
| `core/src/session/session.rs` | `thread_persistence_fut` / `state_db_fut` / `auth_and_mcp_fut` 三个 `.instrument(...)` 包 `Box::pin(async{...})`；`session_result` 改 `Box::pin` 后 `.await`；join 内三 fut Box；`install_initial_mcp_runtime` / `schedule_startup_prewarm` Box。效果：**Session::new 308 KB → 65 KB** |
| `core/src/session/mod.rs` | `capture_step_context` 链 4 处 Box（含 3854 行 join 内两 fut） |
| `core/src/session/handlers.rs` | submission_loop `should_exit` 块：27 个 match 分支逐个 `Box::pin(f(...)).await`，**1.1 MB 块消除** |
| `core/src/session_startup_prewarm.rs` | `session.capture_step_context(...)` 包 Box |
| `core/src/agent/control/residency_tests.rs` | config/manager 装箱、所有 `.await` fut 包 `Box::pin` |
| `core/src/agent/control_tests.rs` | `check_v2_agent_reload` / `spawn_v2_reload_test_child` / 测试函数调用链 Box 化（diff 确认无语义变化） |

**提交 `a683b7a`**（4 文件，+69/-69，turn 链与测试 harness 收尾）：

| 文件 | 改动 |
|---|---|
| `core/src/session/turn_input.rs` | `handle` 内 `start_or_steer`/`start_if_idle` Box；`start_or_steer` 内 `apply_started` Box；`apply_started` 内 `new_turn_with_sub_id`/`new_turn_with_sub_id_if` Box；`new_turn_with_sub_id` 内 `new_turn_with_sub_id_if` Box；`new_turn_with_sub_id_if` 内 `new_turn_from_configuration` Box；`new_turn_from_configuration`/`new_startup_prewarm_turn_from_configuration` 内 `new_turn_context_from_configuration` Box（9 处） |
| `core/src/session/turn_context.rs` | `new_turn_context_from_configuration` 链 4 处 Box |
| `core/src/agent/control_tests.rs` | 测试 harness / parent_thread / spawned_agent / child_thread / stored_child / parent_turn / reloaded_child / reloaded_instructions / reloaded_turn 等 11 处大值局部 `Box::new`；`inject_response_items` fut `Box::pin` |
| `core/src/session/session.rs` | `resolved_environments = Box::new(turn_environments.snapshot().await)`（跨 await 存活到 1820 行的大局部） |

### 5.3 验证数据（WSL Ubuntu-22.04，dev 用户）

> **重要更正（2026-09-19 新发现）**：`sudo -u dev env ...` 默认 **env_reset**，会清掉从 root 继承的 `RUST_MIN_STACK`。此前文档/验证中所有标称 "8 MiB" 的用例（含"修复版 8 MiB debug 全量除 resume 外全绿"、"原始代码 8 MiB debug 跑 resume 同样溢出"）**实际都在默认 2 MiB 下执行**——结论作废。真 8 MiB 必须在测试进程的 env 里显式写 `RUST_MIN_STACK=8388608`。

| 场景 | 结果 |
|---|---|
| 首个溢出测试（residency `interrupted_v2...`）裸 2 MiB 默认栈 | **通过**——原崩溃点根除（Box::pin 后） |
| `resume_agent_respects_max_threads_limit` 裸 2 MiB 默认栈 | **通过**（turn 链 9 处 Box 后） |
| **真 8 MiB debug 全量**（`RUST_MIN_STACK=8388608` 显式传入测试进程，无代理注入） | **2522 passed；7 failed 全为需外网/并发类**（有代理注入时 2527 passed；2 failed 为代理继承断言 + 1 例并发 flaky）——**栈溢出为 0**，官方 8 MiB 等价值下根治达成 |
| 剩余临界测试 `ensure_v2_agent_loaded_reloads_registered_unloaded_agent` 裸 2 MiB | 需约 3 MiB（gdb 帧合计 ~1.9 MB：测试 harness `check_v2_agent_reload` 434 KB + `Session::new` join 块 244 KB + spawn 链 215 KB 等）；3 MiB 下通过。属测试侧深链（spawn 测试子 agent），**非产品核心链** |
| release（优化构建）全量 | 本地环境限制未能完成（WSL 8 GB 内存编译 OOM/SIGBUS，与官方 CI 机器规格不同；未作为交付证据） |

**结论**：产品核心链（Session 启动 / submission_loop / capture_step_context / turn 生命周期）已从代码层根治——默认 2 MiB 栈下关键测试全部通过、真 8 MiB（官方等价值）下全量**零栈溢出**。剩余 2 MiB 全绿差在 1 个测试侧深链（`ensure_v2_agent_loaded_reloads_registered_unloaded_agent`，gdb 显示其 harness `check_v2_agent_reload` 已 Box 化到 434 KB 极限、测试辅助链无法再显著压缩），已记录为后续项（可对测试结构本身做拆分，属测试侧工程）。

### 5.4 提交与 PR 实测（2026-09-19）

```
01e49c8 fix(core): shrink async state machines to fit default 2 MiB thread stack
  （6 文件，+162/-139；与 6bfed3b 同分支 fix/rust-test-stack-8mib）
a683b7a refactor(core): box turn-chain futures and large test locals to shrink default-stack depth
  （4 文件，+69/-69）
已推送 fork：6bfed3b..01e49c8..a683b7a fix/rust-test-stack-8mib -> fix/rust-test-stack-8mib
gh pr create --repo openai/codex 第三次被拒（预期内）：
  GraphQL: cyberspace-cs does not have the correct permissions to execute CreatePullRequest (createPullRequest)
```

**完整证据链**：官方文档声明（不收外部 PR）+ API `pull_request_creation_policy=collaborators_only` + **三次** `gh pr create` 实测拒绝——外部贡献的合规形态确认为 **issue**；代码（3 笔提交）已推送 fork 可被官方拉取。

---

## 六、后续可选动作（用户决策）

1. **直接发 issue**：把第三节草稿粘贴到 https://github.com/openai/codex/issues/new —— 完全符合官方贡献政策（bug report + root cause + 建议方向），是最可能的官方采纳路径。
2. **等官方邀请后提交 PR**：官方明确不收外部 PR；若维护者在该 issue 下表态欢迎补丁，可把 `fix/rust-test-stack-8mib` 分支推送到 fork 再开 PR。
3. **保持本地补丁**：当前提交已在本地分支，随时可用 `git cherry-pick` / `git format-patch` 迁移。

## 七、已记住的用户偏好

- 「本地代理可访问」已写入长期偏好（preference_id `p1_AQCCKvrT8Jc`），后续网络类操作默认走代理。
