---
name: herdr-integrate-cbc
description: 将新 agent kind 集成进 herdr 上游源码的完整流程（以 codebuddy 为例）：qwen 形态接线清单、屏幕 manifest 编写、构建环境三大坑（rust 工具链 / zig / 代理下依赖拉取）、schema 与 endpoint fixture 再生成、验证与提交。触发词：herdr kind、herdr 集成新 agent、herdr add agent kind、herdr-integrate-cbc。
---

# herdr-integrate-cbc

把一个新的 agent kind（本例：CodeBuddy Code）加进 [herdr](https://github.com/herdrdev/herdr) 源码，使其获得原生检测、`herdr agent start --kind <x>`、`agent prompt`、resume 全链路。本 skill 是 2026-10-04 实战 codebuddy 的过程沉淀。

## 0. 定形态（先想清楚，少改一半）

herdr 的 agent kind 有两种接线形态，改动量差一倍：

| 形态 | 检测 | 集成 hook | 例子 |
|---|---|---|---|
| 检测-only | manifest | 无 | muse（最新，最小改动） |
| 会话身份 | manifest | SessionStart 上报 session id | qwen（推荐，打通 resume/prompt） |

选形方法：`grep -rn "qwen" --include="*.rs" src crates` 列出全部接线点，再用 `grep -rn "muse"` 对照减法。qwen 形态即"屏幕 manifest 管状态 + hook 管会话身份"。

## 1. 接线清单（qwen 形态，19 处）

**新建**
- `src/detect/manifests/<kind>.toml`（见 §2）
- `src/integration/assets/<kind>/herdr-agent-session.sh` + `.ps1`（抄 qwen 改 4 处：`HERDR_INTEGRATION_ID`、`--source herdr:<kind>`、`--agent <kind>`）
- `distribution/agent-detection/<kind>.toml`（manifest 镜像）+ `index.toml` 追加 `[[agents]]` 段

**修改（Rust）**
- `src/detect/mod.rs`：枚举 variant、`ALL`（长度 +1）、`SCREEN_MANIFEST_AGENTS`（+1）、`agent_label`、`interactive_agent_executable`、`lookup_agent` 别名串、`session_identity_only_integration` 加 `("herdr:<kind>", "<kind>")`
- `src/detect/manifest.rs`：`BUNDLED_MANIFESTS` 注册
- `src/api/schema/integrations.rs`：`IntegrationTarget` variant、`ALL`（+1）
- `src/integration/mod.rs`：`<KIND>_HOOK_INSTALL_NAME/_ASSET/_INTEGRATION_VERSION/_HOOK_EVENTS` 四常量
- `src/integration/env.rs`：`<KIND>_CONFIG_DIR_ENV_VAR` + `<kind>_dir()`（`config_dir_from_env_or_home(VAR, &[".<kind>"])`）
- `src/integration/types.rs`：`InstallPaths` / `UninstallResult` 两个 struct
- `src/integration/registry.rs`：label、command_names、windows 支持列表、`integration_specs()` 三元组（数组长度 +1）
- `src/integration/targets.rs`：三处 import + `install_<kind>()` + `uninstall_<kind>()`（照抄 qwen）
- `src/integration/actions.rs`：install/uninstall 两个 dispatch arm
- `src/cli/integration.rs`：parse arm + usage/help 字符串
- `src/agent_resume.rs`：`is_reserved_native_state_source`、resume argv（`<kind> --resume <id>`，先 `--help` 核实旗标）、`is_official_agent_source`、plan 测试
- `src/terminal/state.rs`：session replacement 白名单加 `(source, kind, Some("startup"|...))`
- `src/config/sound.rs`：字段 + for_agent arm + Default
- `src/config/model.rs`、`src/main.rs`：cjk_ime 文档名单（纯注释）

**测试**
- `src/detect/mod.rs` tests：identify / parse_label / executable 表（有 `len == ALL.len()` 断言，漏了必红）
- `src/integration/tests.rs`：env 清理列表、windows 支持断言、install/uninstall 三连测
- 编译器兜底：对 `Agent` 的穷尽 match 漏一处即编译错；编译器看不到的只有 `matches!((source, agent))` 字符串对清单，逐条自查

## 2. 屏幕 manifest 编写（codebuddy.toml 实战要点）

- 先抓真实画面：`herdr pane read <pane> --lines N` 看 idle/working 各态底部行，`herdr agent list` 看 `terminal_title` 的 OSC 标题前缀
- codebuddy 是 Claude Code fork：工作态标题为 braille spinner 前缀 `^[\x{2800}-\x{28FF}] `，停稳后为 `✳ `；输入框是 `>` 不是 `❯`
- composer 在响应时保持可见，所以 idle 规则（prompt_box_body）优先级要压低（950），working/blocked 规则（970/980/1100）压它一头
- 抄 `claude.toml` 的对话框规则（"do you want to proceed?" / "esc to cancel" 同款字符串），把 `❯` 判断换成 `>`
- TOML 里 unicode 用 `\x{...}` 转义；region 合法名见 `manifest.rs` 的 `validate_region_name`

## 3. 构建环境三坑（代理内网必读）

1. **rust 工具链**：仓内 `rust-toolchain.toml` 钉死版本（如 1.96.1），rustup 自动下载可能因网络失败；`rustup toolchain install <ver> --profile minimal` 重试即可
2. **Zig 必装**：vendored `libghostty-vt` 需要 Zig 0.16.0，`cargo build` 报 `zig executable not found` 时设 `ZIG=<path>/zig`。0.16 布局是顶层 `zig` 二进制（无 bin/ 子目录）
3. **zig 不走代理**：`zig build` 直连下载依赖会 400/断流。 workaround：用 curl 代下 `build.zig.zon` 里全部 url（含传递依赖，需循环跑 build 收集缺包，约 3 轮），`git+https://` 的用 `git clone` + `git archive` 成 tarball，然后在**有 build.zig 的目录里** `zig fetch <tarball>`（0.16 怪癖：在 /tmp 裸跑会报 no build.zig）。哈希是内容寻址，与 zon 对上即成功

## 4. 再生成与契约（改枚举必做）

- API schema 产物：`HERDR_UPDATE_API_SCHEMA=1 cargo test -p herdr --bin herdr generated_protocol_schema_artifact_is_current`，会更新 `docs/next/api/herdr-api.schema.json`
- endpoint 形状 fixture：`integration.install` 的 SHA256 会因枚举加 variant 而变；跑失败测试，取 **left（actual）** 的哈希更新 `tests/fixtures/endpoint-method-shapes-v1.json`（别取 right，那是旧 fixture 值）

## 5. 验证与提交

```bash
cargo fmt && cargo clippy --all-targets   # 零告警
cargo build                                # 需要 ZIG env
cargo test -p herdr --bin herdr            # ~2960 个
```

已知环境 flaky（与改动无关，单独跑即过）：`manifest_update::*`（远端目录）、`finite_clipboard_commands*`、`manifest_action_invoke_injects_plugin_paths`，并发全跑时偶发，失败集每轮不同。判定标准：与本次改动相关的模块（detect / integration / agent_resume / schema）必须稳定全绿。

注意上游有外部贡献者门槛：`CONTRIBUTING.md` + `.github/APPROVED_CONTRIBUTORS`，build.rs 会打提醒；提 PR 前先看自己是否在名单里、要不要先找 sponsor。

## 6. 产出门槛（验收 codebuddy 的实际效果）

装好新 herdr 后逐项验证：
1. `herdr agent start cb --kind codebuddy --pane <pane>`（pane 停在 shell 提示符）
2. `herdr agent list` 出现 codebuddy 且状态随 TUI 变（working/idle/blocked）
3. `herdr agent prompt cb "..." --wait --timeout 120000` 走通（不再报 agent_not_ready）
4. 会话身份 hook 上报：`herdr agent get cb` 可见 session id；resume 生效
