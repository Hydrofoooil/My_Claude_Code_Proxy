# Claude Code 动态 Fast Mode

## 当前状态（始终覆盖更新）

### 项目理解
- Claude Code 通过 Anthropic Messages 顶层 `speed` 表达 fast/standard；Codex Responses 通过 `service_tier` 选择 fast lane。
- 代理已按请求读取 `speed`：fast 映射为 priority，显式 standard 清除模型 `-fast` 派生值；全局配置仍保持最高优先级。
- 响应按请求隔离；非流式优先采用 terminal response 的实际 tier，流式优先采用 `response.created` 的 tier，缺失时才回退到请求 tier。
- Codex WebSocket 实测会接受 priority 请求，但 non-streaming terminal response 可回报 standard；代理按实际结果回报，不虚报 fast。

<!-- 最后更新: Claude 2026-08-13 15:19 -->

### 进行中的任务
- 实现、验证和本机热部署已完成；无剩余实现任务。
- 备份二进制保留在 `/mnt_scalelab/maoting/home/.local/bin/claude-code-proxy.backup-20260813-151612`，供后续手工回退。

<!-- 最后更新: Claude 2026-08-13 15:19 -->

### 关键文件索引
- `src/providers/codex/translate/request.rs`：Anthropic 请求到 Codex Responses 请求的转换与 tier 决策。
- `src/providers/codex/translate/reducer.rs`：Codex SSE 归约及 Anthropic usage 类型。
- `src/providers/codex/translate/accumulate.rs`：非流式 Anthropic Message 构造。
- `src/providers/codex/translate/live_stream.rs`：实时 SSE 转换。
- `src/providers/codex/translate/stream.rs`：buffered SSE 转换 helper。
- `src/providers/codex/mod.rs`：Codex provider 主调用链、request-scoped 元数据传递与 standalone search fast 拒绝。
- `src/providers/codex/search.rs`：standalone search 的 standard usage 回报。
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
