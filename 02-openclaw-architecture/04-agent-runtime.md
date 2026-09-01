# 第 4 课：Agent 运行时 —— 大脑如何思考

## 本课问题

1. "Agent" 在代码里到底是什么？一个循环？一套状态机？
2. 用户口中的"人格、记忆、技能"如何变成模型能看到的东西？
3. 多个用户、多个 Agent、多个任务并发时，系统怎么保证不乱？

---

## 1. Agent 运行时的代码版图

OpenClaw 的 Agent 运行时是**自研并内化**的（曾经依赖外部框架，2025 年底通过 #85341 大重构收回自有）。官方文档 `docs/agent-runtime-architecture.md` 给出了权威版图：

| 路径 | 职责 |
|-|-|
| `packages/agent-core/` | **可复用核心**（`@openclaw/agent-core`）：agent 循环、harness 类型、消息、压缩、prompt 模板、会话存储契约 |
| `src/agents/embedded-agent-runner/` | **内置尝试循环**：`run.ts`、模型选择与归一化、各 provider 请求参数、compaction、transcript 接线 |
| `src/agents/sessions/` | 会话持久化（`session-manager.ts`）、资源发现、技能/prompt 模板加载 |
| `src/agents/runtime/` | 门面层：把 `@openclaw/agent-core` 接到 plugin-sdk 的 LLM 运行时 |
| `src/agents/agent-tools*.ts` | OpenClaw 自有工具定义、参数 schema、工具策略、沙箱编辑工具 |
| `src/agents/agent-hooks/` | 内置钩子：compaction 守卫、上下文裁剪 |
| `src/agents/harness/` | harness 注册表与选择策略（内置 runtime vs 插件 runtime 如 codex） |
| `src/llm/` | 模型/provider 注册表、流式传输（第 5 课展开） |

**关键边界**：核心通过 SDK barrel 调用内置运行时；插件只能用 `openclaw/plugin-sdk/*` 公开入口，不得 import `src/**` 内部。

## 2. Agent Loop：一切的发动机

核心循环在 `packages/agent-core/src/agent-loop.ts`。一次完整运行（run）的编排逻辑：

```
agent RPC 收到请求
  ├─ 校验参数、解析会话（sessionKey/sessionId）、立即返回 { runId, acceptedAt }
  └─ agentCommand 异步执行：
      ├─ 解析模型 + thinking/verbose 默认值
      ├─ 加载技能快照
      └─ runEmbeddedAgent:
          ├─ 按会话车道 + 全局车道串行化（队列）
          ├─ 解析模型 + 认证 profile
          ├─ 准备工作区（沙箱运行可重定向到沙箱工作区）
          ├─ 注入 bootstrap 文件 → 组装系统提示词
          ├─ 获取会话写锁
          ├─ 订阅运行时事件（assistant 流、tool 流、lifecycle 流）
          ├─ 执行 Agent Loop（模型⇄工具循环，可流式输出）
          ├─ 强制 run 超时（默认 48h，可配 0=不限）
          └─ 返回 payloads + usage 元数据
```

**三类事件流**（客户端可订阅）：

- `lifecycle`：`phase: start | end | error`——run 的生死边界
- `assistant`：模型输出的流式增量
- `tool`：工具开始/更新/结束事件

`agent.wait` RPC 可以阻塞等一个 runId 的 lifecycle 终态——这是自动化和脚本化调用的基础。

## 3. Prompt 组装：产品定义的"人设"如何落地

第 2 课讲过组装的**分区结构**，这里讲它的**三层实现架构**（对 PM 的意义：理解"改人设"和"改策略"分别动哪里）：

| 层 | 组件 | 类比 |
|-|-|-|
| 渲染层 | `buildAgentSystemPrompt` | 纯模板引擎，不读全局配置 |
| 配置解析层 | `resolveAgentSystemPromptConfig` | 把 openclaw.json 里的 agent 配置翻译成渲染参数 |
| 运行时适配层 | embedded / CLI / compaction 适配器 | 收集实时事实（工具、沙箱状态、渠道能力）喂给渲染层 |

**Provider 也能"插嘴"**：模型厂商插件可以替换三个命名分区（`interaction_style`、`tool_call_style`、`execution_bias`），或在缓存边界上/下注入稳定前缀/动态后缀。例：内置的 GPT-5 家族覆盖层会注入执行纪律与输出契约——**不同模型的"脾气"用提示词补丁来对齐**，而不是一套 prompt 打天下。

**Prompt 模式**：主 Agent 用 `full`（全分区）；子代理用 `minimal`（剥掉记忆召回、自我更新、心跳等）；`none` 只有基础身份行。

## 4. 工具系统与策略

- **核心工具**（read/exec/edit/write 等）始终可用，但受**工具策略**约束：`tools.allow` / `tools.deny`、按 agent 覆盖、`tools.elevated` 有全局门+per-agent 门双重校验
- `apply_patch` 默认对 OpenAI 模型开启，由 `tools.exec.applyPatch` 门控
- 工具前后有钩子：`before_tool_call` 可阻断（`{block:true}` 是终局裁决）、`after_tool_call` 可改写结果、`tool_result_persist` 在写入 transcript 前同步转换结果
- 工具结果被消毒（尺寸、图片载荷）后才记录/广播

## 5. 并发模型：车道（lane）系统

```
入站消息 ──> session:<key> 车道（每会话并发=1，严格串行）
                  │
                  └─> 全局车道 main（默认并发=4，跨会话并行上限）

其他车道：cron（定时任务）、cron-nested、subagent（子代理，默认并发=8）、nested
```

- 会话车道保证：**同一会话同一时刻只有一个 run** → 不会有两个 run 同时写 transcript
- 全局车道保证：整机 LLM 并发有上限 → 不会把 API 速率打爆
- 独立车道保证：后台 cron 任务不会堵住用户对话
- transcript 写入另有**跨进程文件锁**（防绕过队列的写入者）

**卡死自愈**：诊断系统持续观察 `processing` 会话——有进展的标 `long_running`（不动它），无进展的标 `stalled`（过阈值后主动 abort 释放车道），簿记残留标 `stuck`（立即回收车道）。

## 6. Steering：被低估的产品创新

默认队列模式 `steer` 值得单独讲——它是"让 Agent 感觉活着"的关键：

- 用户在 run 进行中发消息 → 不新开 run，而是**注入当前运行时**
- 注入时机有讲究：**等当前 assistant 回合的工具调用执行完、下一次 LLM 调用前**送达——既不打断进行中的工具（可能是有副作用的操作），又能让模型"尽早知道"用户改主意了
- 如果运行时无法接受 steering（不在流式中），降级为 run 结束后 followup
- `/queue interrupt` 才是唯一会中止进行中工具的模式

对比主流方案：ChatGPT 的"停止生成"是全有或全无；OpenClaw 的 steer 是**第三种状态——"继续，但听我说"**。

## 7. 回复成型（Reply Shaping）

run 结束时的收尾规则，全是产品细节：

- 精确等于 `NO_REPLY` 的输出被过滤（AI 的"已读不回"机制——群聊里尤其重要）
- 消息发送工具已发出的内容从最终 payload 里去重（防双发）
- 没有任何可渲染内容但工具报错了 → 发兜底的工具错误回复（除非消息工具已经发过用户可见回复）
- verbose 模式下内联工具摘要

## 8. 子代理：一个 Agent 拆成多个专业执行者

前面讲的都是"单 Agent 的 run"，OpenClaw 还支持**子代理（subagent）**——主 Agent 在 run 中按需派生专业子代理来分担任务。这对应第 4 节车道里的 `subagent` 独立车道（默认并发 8）。

**典型协作模式**（mimo 课程把它抽象为三种，与 OpenClaw 的实现吻合）：

| 模式 | 描述 | 适用场景 |
|-|-|-|
| **主从式** | 主 Agent 拆任务、派子代理、收结果 | 一次调研拆成多个并行搜索 |
| **对等式** | 多个 Agent 平等协作、共享上下文 | 多角色头脑风暴 |
| **层级式** | 子代理还能再派孙代理 | 大任务逐层分解 |

**关键设计**：子代理用 `minimal` prompt 模式（第 3 节提过——剥掉记忆召回、自我更新、心跳，只有基础身份 + 任务指令），**它们是"用完即走"的执行者，不背长期人格**。这也解释了为什么子代理有独立车道：它们不参与主会话的串行队列，可以并行跑，跑完结果交回主 Agent 整合。

> **PM 视角**：子代理是"一个 Agent 系统"到"多 Agent 系统"的中间形态——它不需要部署多个完整实例（第 11 课那种多智能体编排），而是在**单次 run 内**动态编排。对产品经理的判断标准：当主 Agent 的上下文开始"一锅粥"、任务可自然拆解时，就该考虑子代理了；当需要长期分工、独立记忆、各自定时任务时，才需要真正的多实例编排。

## PM Takeaways

1. **Agent Loop 本身很简单（模型⇄工具循环），复杂度全在"边界管理"**：超时、车道、锁、steering、降级。评估一个 Agent 系统的成熟度，别看 demo，看它的边界处理清单。
2. **Prompt 是代码，不是文案**：OpenClaw 把 prompt 拆成"渲染层/配置层/适配层"，还允许 provider 插件打补丁。人设、纪律、能力说明各有归属，改起来才知道动哪里。
3. **"会话内串行 + 会话间并行 + 全局有上限"是 Agent 并发模型的标准答案**，直接可抄。
4. **Steer 模式值得所有对话式 Agent 产品借鉴**：它把"用户补充指令"从"打断"重新定义为"注入"，既不浪费已完成的工具执行，又保持对话的响应感。

## 实证

- 循环本体：`packages/agent-core/src/agent-loop.ts`
- 运行编排：`src/agents/embedded-agent-runner/run.ts`
- Prompt 文档：`docs/concepts/system-prompt.md`
- 队列文档：`docs/concepts/queue.md`、`docs/concepts/queue-steering.md`
- Agent Loop 文档：`docs/concepts/agent-loop.md`

---

上一课：第 3 课：Gateway 与渠道层 ｜ 下一课：第 5 课：LLM 抽象层与故障转移

![Agent运行时知识图谱](https://feishu.cn/file/Gz6dblb8oosAxmx05cJczvyenVd)
