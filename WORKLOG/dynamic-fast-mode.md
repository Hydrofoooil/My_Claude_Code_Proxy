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
- 当前 4090 两机仍运行 glibc 2.35 构建的 v0.1.32，A800-1 运行 v0.1.35；该版本差异未发现后端 401 证据，但会造成跨集群功能不一致。

<!-- 最后更新: Claude 2026-09-01 11:17 -->

### 进行中的任务
- 已修复 `claude-proxy` 启动器认证隔离：新进程只保留 gateway 的 `ANTHROPIC_AUTH_TOKEN`，显式清除继承的 Claude OAuth、API key 与云厂商选择变量。
- 现有两个 proxy 会话仍携带旧 OAuth 环境；环境变量无法在运行中移除，出现 `Not logged in` 的会话需要单独退出并用 `claude-proxy -c` 重启。
- 后端 Codex auth 当前有效，全部轮转日志中未发现真实 401/Not authenticated；跨集群 proxy 版本统一仍未执行。

<!-- 最后更新: Claude 2026-09-01 11:17 -->

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

<!-- 最后更新: Claude 2026-08-13 15:19 -->

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
