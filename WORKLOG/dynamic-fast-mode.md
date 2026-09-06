# Claude Code 动态 Fast Mode

## 当前状态（始终覆盖更新）

### 项目理解
- Claude Code 通过 Anthropic Messages 顶层 `speed` 表达 fast/standard；Codex Responses 通过 `service_tier` 选择 fast lane。
- 代理已按请求读取 `speed`：fast 映射为 priority，显式 standard 清除模型 `-fast` 派生值；全局配置仍保持最高优先级。
- 响应按请求隔离；非流式优先采用 terminal response 的实际 tier，流式优先采用 `response.created` 的 tier，缺失时才回退到请求 tier。
- Codex WebSocket 实测会接受 priority 请求，但 non-streaming terminal response 可回报 standard；代理按实际结果回报，不虚报 fast。
- v0.1.35 引入新的 incomplete-response policy 与统一 terminal semantics；动态 fast 的 execution-mode 元数据必须与该 policy 同时保留。
- Anthropic Messages 与启用的 OpenAI-compatible JSON 路由共用请求体上限；定制版默认已从 16 MiB 提升为 64 MiB。
- Codex 凭据由同机 proxy 共享，真正的后端掉认证不会只影响一个 Claude Code 会话；单会话 `Not logged in` 首先检查该进程继承的 Claude OAuth/API/cloud-provider 环境。
- 定制 main 已越过官方 v0.1.35 tag，跟踪官方未发版的 main（合并到 `55bf0b5`）以取得 gpt-6-astra；Cargo 版本号仍是 0.1.35，官方尚未发新 release。
- `gpt-6-astra` 走 Responses Lite 通道；它的 `-fast` 别名与逐请求动态 fast 都是从 `ALLOWED_MODELS` 派生的通用机制，无需为新模型单独适配。
- 官方新增 `request-id` 响应头（Claude Code 用它填 transcript 的 `requestId`），错误响应与流式响应都会带上。
- 当前 A800-1 运行合入 astra 的构建（glibc 2.39）；4090 两机仍是 `~/.local/lib/claude-code-proxy/v0.1.32-glibc235/` 里的 v0.1.32，`ccp` 脚本的 BIN 也仍指向那份，跨集群版本不一致未解决。

<!-- 最后更新: Claude 2026-09-06 13:39 -->

### 进行中的任务
- gpt-6-astra 升级已完成并热部署到 A800-1（serve PID 2273461），定制 main 已推到 `Hydrofoooil/My_Claude_Code_Proxy`；无剩余实现任务。
- 仍未做：4090 两机的版本统一。那两台是 glibc 2.35，A800 上编出的二进制不能用，需要在 4090 上单独构建一份并更新 `ccp` 的 BIN 路径。
- `claude-proxy` 启动器已隔离认证环境；仍在跑的旧 proxy 会话若报 `Not logged in`，退出后用 `claude-proxy -c` 重启即可。

<!-- 最后更新: Claude 2026-09-06 13:39 -->

### 关键文件索引
- `src/providers/codex/translate/request.rs`：Anthropic 请求到 Codex Responses 请求的转换与 tier 决策。
- `src/providers/codex/translate/reducer.rs`：Codex SSE 归约及 Anthropic usage 类型。
- `src/providers/codex/translate/accumulate.rs`：非流式 Anthropic Message 构造。
- `src/providers/codex/translate/live_stream.rs`：实时 SSE 转换。
- `src/providers/codex/translate/stream.rs`：buffered SSE 转换 helper。
- `src/providers/codex/mod.rs`：Codex provider 主调用链、request-scoped 元数据传递与 standalone search fast 拒绝。
- `src/providers/codex/search.rs`：standalone search 的 standard usage 回报。
- `src/openai_compat/mod.rs`：Messages/OpenAI-compatible JSON 请求体的共享 64 MiB 上限。
- `tests/server.rs`：验证超过旧 16 MiB 上限的 Messages 请求可继续进入 JSON 与模型路由。
- `docs/src/content/docs/providers/codex.md`：Codex 动态 fast、优先级和 search 限制文档。
- `/mnt_scalelab/maoting/home/.local/bin/claude-proxy`：允许 custom gateway 会话操作 Claude Code `/fast` 的启动 wrapper。
- `/mnt_scalelab/maoting/home/.local/bin/ccp`：proxy 后台服务管理脚本（tmux 保活）；其 `BIN` 变量目前指向 v0.1.32-glibc235，与 `~/.local/bin/claude-code-proxy` 上部署的版本不一致。
- `src/providers/codex/translate/model_allowlist.rs`：允许模型清单、`-fast` 别名派生与 Responses Lite 判定，gpt-6-astra 在此注册。
- `src/registry.rs`：模型发现用的 provider 模型清单。
- `/mnt_scalelab/maoting/home/.cargo/config.toml`：把 crates.io 源替换为中科大镜像（直连约 7 KB/s，镜像约 240 KB/s）。

<!-- 最后更新: Claude 2026-09-06 13:39 -->

---

## 变更历史（按时间在尾部追加；仅在压缩合并时可改写旧条目）

### [2026-08-13 14:37][Claude] 启动动态 fast mode 实现

**做了什么**
- 确认官方协议、现有请求/响应链路、测试与部署边界，尚未修改产品代码。

**关键决策与发现**
- 保持全局配置高于请求、请求高于 `-fast`；显式 `speed: standard` 可取消模型后缀 fast。
- 不使用全局状态；响应优先回报上游实际 tier，缺失时才使用当前请求的 tier。
- tmux session 还有交互窗口，部署时不能执行会杀整个 session 的 `ccp restart`。

### [2026-08-13 15:19][Claude] 完成实现、验证与热部署

**做了什么**
- 完成逐请求 speed→service tier 桥接、actual tier usage 回报、streaming/non-streaming 转换、standalone search 约束、文档及 wrapper 更新。
- release 二进制原子替换到 `/mnt_scalelab/maoting/home/.local/bin/claude-code-proxy`；仅 TERM serve 子进程，由 tmux 保活循环拉起 PID 2418545，未中断 session/window。

**关键决策与发现**
- 验证证据：119 项 focused tests 通过；全套为 812+5+8+11+6+9+1+30+9+1+35+41 项测试全部通过；Clippy `-D warnings`、format、`git diff --check`、release build 均通过。
- non-streaming 实验产物 1：omitted→standard、fast 请求上游发送 priority 但 terminal 实际回报 standard、显式 standard→standard，三次均 HTTP 200。
- streaming 实验产物 2：fast 的 `message_start` 回报 fast/priority，standard 回报 standard/standard；两者 `message_delta` 均不携带 speed/tier，且均完整到达 `message_stop`。
- 热部署实验产物 3：旧 serve PID 587915、新 PID 2418545，部署后二进制 SHA-256 为 `c1763f0600be9b390cab4039584acf0eaa383c09f89d770588105500ef6afbbc`，`/healthz` 返回 `{"ok":true}`。
- 原生 `/fast` 实验产物 4：独立 Claude Code v2.1.227 会话中 OFF→ON 后下一请求日志为 `serviceTier: priority`，ON→OFF 后下一请求为未设置 tier（standard）；两次均收到预期回复，无需重启 proxy。实验后已关闭临时 tmux session。

### [2026-08-20 20:21][Claude] 发布私有定制仓库

**做了什么**
- 将动态 `/fast` 补丁提交为 `db5e6fc` 并发布到私有仓库 `Hydrofoooil/My_Claude_Code_Proxy` 的 `main` 分支。
- 将定制仓库设为本地 `origin`，官方 `raine/claude-code-proxy` 保留为 `upstream`，便于后续同步官方更新后移植补丁。

**关键决策与发现**
- 发布前对全部 tracked 文件及 staged diff 进行了凭据模式扫描，未发现 token、私钥或 JWT。
- 仓库文档已明确：其他机器通过 custom gateway 使用原生 `/fast` 时，需要设置 `CLAUDE_CODE_SKIP_FAST_MODE_ORG_CHECK=1`。
- GitHub 远端与本地提交 SHA 完全一致，目标仓库为 private，默认分支为 `main`。

### [2026-09-01 10:19][Claude] 合并官方 v0.1.35 并提升请求上限

**做了什么**
- 通过 merge 将官方 `v0.1.35` 合入定制 `main`，保留已发布提交历史；语义合并 Codex accumulator、live stream 与 reducer 冲突。
- 将 `src/openai_compat/mod.rs` 的 JSON 请求上限从 16 MiB 提升到 64 MiB，并在 `tests/server.rs` 增加超过旧上限的回归测试。

**关键决策与发现**
- 同时保留 v0.1.35 的 incomplete-response policy/terminal 修复与定制 execution-mode usage 回报，不能简单选择任一冲突侧。
- 回归测试确认大于 16 MiB 的 Messages 请求已越过 body reader 并进入模型路由；127 项 Codex translation tests 通过。
- 尚待全套测试、Clippy、release build、热部署与 GitHub 发布，当前不能标记升级完成。

### [2026-09-01 10:31][Claude] 完成升级验证

**做了什么**
- 增加 Anthropic HTTP 413 `request_too_large` 响应，替代超过 64 MiB 时误导性的 `Invalid JSON`；补充无大内存分配的响应单测。
- 修正 v0.1.35 既有 cancellation 测试夹具：用 typed retryable failure 触发重试，不再误把 informational rate-limit snapshot 当作失败。

**关键决策与发现**
- 原版 v0.1.35 的该 cancellation 测试在干净源码上也稳定失败，原因是测试事件与 v0.1.35 新的 rate-limit telemetry 语义冲突；生产语义无需回退。
- 最终验证：127 项 Codex translation tests、64 MiB focused tests、全套 1024 项测试、Clippy `-D warnings`、format、diff check 和 release build 全部通过。
- release artifact 为 v0.1.35，SHA-256 `4e9067382ebf5981df9b7922c204aa2f61f9e69f2caac969a75f44bf94ac4d01`。

### [2026-09-01 10:36][Claude] 热部署并发布升级版本

**做了什么**
- 原子部署 v0.1.35 release binary，仅 TERM 旧 serve PID 2418545，由 tmux 保活循环拉起新 PID 2284426；health check 通过。
- 将 merge commit `997ba8b` 发布到 `Hydrofoooil/My_Claude_Code_Proxy` 的 `main`；发布时仓库 visibility 为 public。

**关键决策与发现**
- 实验产物 5：non-streaming standard→fast→standard 均 HTTP 200；fast 请求日志确认为 priority，terminal 实际 tier 仍如实回报 standard。
- 实验产物 6：streaming fast 回报 fast/priority，standard 回报 standard/standard，terminal `message_delta` 均未携带非官方 speed/tier 字段。
- 实验产物 7：17,825,887-byte 请求越过旧 16 MiB 门槛并到达模型校验；68,157,507-byte 请求超过 64 MiB 后返回 HTTP 413 `request_too_large`，且错误明确报告 64 MiB。
- 远端与本地 merge commit SHA 一致；最终 WORKLOG 状态将在后续文档提交中同步。

### [2026-09-01 11:17][Claude] 隔离单会话客户端认证状态

**做了什么**
- 修改 `/mnt_scalelab/maoting/home/.local/bin/claude-proxy`，在 exec Claude Code 前清除 `CLAUDE_CODE_OAUTH_TOKEN`、`ANTHROPIC_API_KEY` 及 Bedrock/Vertex/Foundry 选择变量。
- 更新 `docs/src/content/docs/providers/codex.md`，记录 custom gateway 应只保留 bearer gateway 认证路径。

**关键决策与发现**
- 三份共享 proxy 轮转日志未出现真实 Codex 401、Not authenticated 或 refresh failure；共享 auth 有效至 2026-09-10。
- 20 个正在连接本地 proxy 的 Claude Code 进程中，有 2 个额外继承了 Claude OAuth token，构成已观察到的唯一逐进程认证差异。
- 实验产物 8：用伪 OAuth 和云厂商变量启动隔离会话后，新 wrapper 保留本地 base URL 与 gateway token，同时成功清除 OAuth、API key 和云厂商变量；临时 tmux 已关闭。
- 跨集群检查发现 4090-1/2 运行 v0.1.32，A800-1 运行 v0.1.35，三者共享同一 auth 文件；未将版本差异擅自归因为本次 UI 现象。

### [2026-09-06 13:39][Claude] 合并官方 main 取得 gpt-6-astra 支持

**做了什么**
- 澄清前提：官方最新 release 仍是 v0.1.35（GitHub Releases/tags API 双向确认），gpt-6-astra 只在未发版的官方 main 上（PR #129，提交 `55bf0b5`，2026-09-04）。因此按"合并官方 main"而非"升级到新 release"执行。
- 把官方 main（v0.1.35 之后 6 个提交）merge 进定制 main，得到合并提交 `f1d3e3d`；合并前先把上次会话遗留的暂存文档改动固化为 `6ff2f37`，并建回滚分支 `backup/pre-astra-merge-20260906`。
- 唯一冲突在 `src/server.rs`：双方都在 `_unused` 之后追加测试模块，保留两侧（我方 `request_limit_tests` 与官方 `request_id_header_tests`）。`dispatch_request` 的请求体超限分支自动合并正确，同时保住我方 HTTP 413 `request_too_large` 与官方的 `with_request_id` 标记。
- 重新安装 Rust 工具链（四台此前都已无 cargo/rustup），并在 `~/.cargo/config.toml` 配置中科大 crates 镜像。
- 原子替换 `~/.local/bin/claude-code-proxy` 并只 TERM serve 子进程，由 tmux 保活循环拉起新进程；已推送到 `Hydrofoooil/My_Claude_Code_Proxy` 的 main。

**关键决策与发现**
- 定制补丁全部无需改写即可适配 astra：动态 fast（speed→service_tier）、`-fast` 别名都基于 `ALLOWED_MODELS` 派生，不含逐模型清单；64 MiB 上限与 413 响应也不受影响。
- 验证证据：cargo fmt、clippy `-D warnings`、release build 均通过；12 个测试二进制共 1035 项测试全部通过。
- 首轮测试有 23 项失败，原因是本会话 shell 继承了 Claude Code 自己的 decodo 代理（`http_proxy`），本地 mock server 请求被代理返回 403；清空代理变量后全绿。这是环境问题，不是代码回归。
- 实验产物 9：部署后 serve 从 PID 2284426 换到 2273461，`/healthz` 返回 `{"ok":true}`，二进制 SHA-256 `e9a62b56dbd474ce941286cd51fc496e5b570141e509619a61df2cba2ef58907`；回滚备份 `~/.local/bin/claude-code-proxy.backup-20260906-133618`。
- 实验产物 10：`/v1/models` 已列出 `gpt-6-astra` 与 `gpt-6-astra-fast`；对 `gpt-6-astra` 的两次真实请求均 HTTP 200 并正常回话，`speed:"fast"` 那次按既有语义如实回报 terminal tier 为 standard。
- 实验产物 11：官方新增的 `request-id` 响应头在实际请求中生效（如 `3f2cda06-5d8e-4e7e-8415-01b105f33929`）。
