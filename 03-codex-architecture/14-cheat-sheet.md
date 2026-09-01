# Codex 架构速查表

> 一页纸记住 Codex。详细论证见各课。

## 架构骨架

- **一个内核**：`codex-core`（Rust），\~120 crate workspace，153 万行
- **五种形态**：TUI / exec / IDE 插件 / 桌面 App / 云任务，外加 TS/Python SDK
- **协议**：SQ（Submission Queue）/ EQ（Event Queue）异步双队列；app-server = JSON-RPC 2.0
- **分发**：npm/brew/安装脚本 → 平台预编译二进制，`codex-cli/bin/codex.js` 纯转发

## 主循环（run_turn）

1. 回收迟到 hook → 2. 采样前压缩 → 3. 解析 @提及拉起 MCP → 4. 捕获世界快照 → 5. 注入技能/插件 → 6. loop { 排空插话 → 采样 → 工具分发 } → 7. TurnDiffTracker 收尾

- 压缩四层：pre-sampling / auto / inline（换模型）/ model_fallback
- 插话（pending input）在 loop 开头排空

## 工具系统

- spec（模型可见说明书）/ handler（执行器）分离
- 自我意识工具：get_context_remaining / request_permissions / request_user_input
- 生态工具：tool_search / request_plugin_install / mcp
- 并行分发：tools/parallel.rs

## 安全四层楼

| 层 | 机制 |
|-|-|
| 物理 | bwrap（Linux）/ Seatbelt（macOS）/ 受限令牌（Windows）：全盘只读 + 显式可写根 + seccomp 断网 |
| 策略 | execpolicy（Starlark）：prefix_rule → allow/prompt/forbidden，match/not_match 加载时校验 |
| 交互 | 审批事件走 SQ/EQ 协议，命令审批与网络审批分离 |
| 协商 | 模型用 request_permissions 主动谈判 |

- .git / .codex 在可写根内盖回只读（保护后悔药）
- WSL1 拒绝沙箱路径；bwrap 缺失 → 内置回退 + 警告

## 扩展三层

| 层 | 提供 | 加载时机 |
|-|-|-|
| Skill | 知识 | 回合内按需注入（进历史，参与压缩） |
| Plugin | 能力 | 安装后常驻 |
| MCP | 工具 | 回合级按需拉起 |

## 持久化

- **Rollout**：会话 JSONL 录像，游标分页，格式迁移命令存在
- **Memories**：后台异步两阶段（抽取→整合），citation 可溯源；ephemeral/子代理不触发

## 多智能体

- 工具化派发（multi_agents v1/v2 并存），子代理任务级生灭
- 云端任务同内核，diff 回传本地

## 十二条 PM 判断

见第 12 课。终极三问：你的".git"是什么？你的"回合"边界在哪？你的信任模型落在哪一层？
