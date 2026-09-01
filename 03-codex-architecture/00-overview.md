# Codex 架构实战课程（K3）

> 面向 AI 产品经理的 OpenAI Codex 源码级架构课。  
> 所有技术事实均以 `openai/codex` 仓库 main 分支源码快照为据（153 万行 Rust，Apache-2.0）。  
> 分析视角沿袭本课程系列传统：**先给结论，再展开源码证据，最后落到 PM Takeaways**。

## Codex 是什么

Codex 是 OpenAI 官方的**本地编码智能体**（"Lightweight coding agent that runs in your terminal"）。  
同一个 Rust 内核（`codex-core`），支撑五种产品形态：

| 产品形态 | 入口 | 界面层 |
|-|-|-|
| 终端 TUI | `codex` | `codex-rs/tui`（ratatui） |
| 非交互执行 | `codex exec "任务"` | `codex-rs/exec` |
| IDE 插件 / 桌面 App | VS Code 扩展、`codex app` | 经 `codex app-server`（JSON-RPC）桥接 |
| 云端任务 | Codex Web（chatgpt.com/codex） | `cloud-tasks` 对接云端环境 |
| 嵌入式 SDK | `@openai/codex-sdk` / Python SDK | spawn CLI，JSONL over stdio |

## 课程地图

| 课 | 主题 | 核心问题 |
|-|-|-|
| 00 | 课程导论 | 为什么 153 万行 Rust？为什么值得 PM 读？ |
| 01 | 总览：一个内核，五种形态 | 单体二进制 + 120 crate workspace 如何组织？ |
| 02 | Agent 主循环：run_turn | 一次对话回合在源码里到底发生了什么？ |
| 03 | 工具系统：注册表与分发 | shell / apply_patch / plan / 多智能体工具怎么挂载？ |
| 04 | apply_patch：自研补丁格式 | 为什么不让模型直接写文件，而要发明一种补丁语法？ |
| 05 | 沙箱：Seatbelt / bwrap / Windows 三端 | Agent 在自己电脑上跑命令，安全边界怎么画？ |
| 06 | 审批体系：人如何保持控制权 | allow / prompt / forbidden 三级决策怎么落地？ |
| 07 | MCP：外部工具的标准接口 | Codex 如何既当 MCP 客户端又当 MCP 服务器？ |
| 08 | Skills：可复用的任务知识 | 技能如何被发现、按需注入、@提及触发？ |
| 09 | 记忆与 Rollout 持久化 | 会话如何落盘？跨会话记忆如何两阶段提炼？ |
| 10 | 多智能体与云端任务 | 子代理怎么派？云端任务和本地是什么关系？ |
| 11 | app-server 与 SDK：嵌入一切 | 一个 JSON-RPC 协议如何撑起 IDE/App/SDK 三端？ |
| 12 | 实战综合 + PM 总复盘 | 把 12 课收成一张架构全景图 |

## 学习路径

**第一阶段（架构骨架）**：00 → 01 → 02，建立"内核 + 协议 + 界面"分层心智模型。

**第二阶段（核心机制）**：03 → 04 → 05 → 06，吃透 Agent 的"手"（工具）与"镣铐"（沙箱与审批）。

**第三阶段（扩展与生态）**：07 → 08 → 09 → 10 → 11 → 12，理解 Codex 如何从 CLI 长成一个平台。

## 与 OpenClaw 课程的关系

两套课程是同一主题的对照样本：

- **OpenClaw**：社区驱动、多渠道接入（Telegram/WhatsApp）、面向个人助理场景；
- **Codex**：大厂工程、编码垂直场景、Rust 单体 + 强沙箱，**把"安全"做成一等公民**。

读完两套，你会看到 Agent 产品在"自由度 vs 控制力"光谱上的两种典型取舍。
