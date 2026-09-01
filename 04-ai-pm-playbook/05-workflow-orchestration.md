# 第 05 课：工作流编排——多步任务怎么拆解和并行

## 一句话结论

AI 产品的复杂任务不是靠"更聪明的模型"解决的，而是靠**显式的工作流编排**——把大任务拆成小步骤，让 AI 按计划执行、按结果反馈、按需要并行。

## 从"问答"到"任务"

传统聊天机器人：用户问 → AI 答（单轮）  
AI Agent：用户给目标 → AI 拆解 → 执行 → 反馈 → 迭代（多轮）

**关键转变**：从"响应式"到"主动式"。

## 案例：Codex 的 plan 工具

Codex 的 `plan` 工具让模型**显式维护任务计划**：

```rust
// codex-rs/core/src/tools/handlers/plan_spec.rs
pub fn create_update_plan_tool() -> ToolSpec {
    let plan_item_properties = BTreeMap::from([
        ("step".to_string(), JsonSchema::string(Some("Task step text.".to_string()))),
        ("status".to_string(), JsonSchema::string_enum(
            vec![json!("pending"), json!("in_progress"), json!("completed")],
            Some("Step status.".to_string()),
        )),
    ]);

    // ... 工具定义
}
```

**使用场景**：

```
用户：帮我重构这个模块，把数据库层抽出来

AI（调用 plan 工具）：
1. [pending] 分析当前模块结构，识别数据库相关代码
2. [pending] 设计新的数据库抽象接口
3. [pending] 创建新的数据库层模块
4. [pending] 迁移现有代码到新接口
5. [pending] 更新测试
6. [pending] 验证所有测试通过
```

**产品价值**：

- 用户能看到 AI 的"思路"，增加透明度
- AI 能自我跟踪进度，不会迷失
- 计划可以动态调整（状态从 pending → in_progress → completed）

<callout emoji="💡">
**拐杖警示**：Claude Code 的 to-do list 与 Codex 的 plan 工具同类——都是强制模型维护任务列表的"拐杖"。Cat Wu（Claude Code 产品负责人）的复盘：to-do list 最初是为了强制模型完成 20 个调用点的重构而加的；**Opus 4 上线后这个功能就不需要了**——模型能力自然覆盖了。今天设计的编排工具，可能是为旧模型的弱点加的拐杖，新模型上线时要记得审计拆掉（详见第 09 课案例 7）。
</callout>

## 案例：Codex 的多智能体派发

Codex 的 `multi_agents` 工具支持**任务级子代理**：

```rust
// 主 Agent 派生子代理处理子任务
let subagent_result = session.call_tool("multi_agents", json!({
    "action": "spawn",
    "task": "Run the test suite and summarize any failures",
    "context": { "working_directory": "/path/to/project" }
})).await;
```

**设计特点**：

- 子代理是**任务级**的，不是常驻的（对比 OpenClaw 的常驻 Agent 团队）
- 子代理的结果作为工具结果回传给主 Agent
- 子代理不进入长期记忆（`memories/README.md`：子代理会话不触发记忆提炼）

**与 OpenClaw 的对比**：

| 维度 | Codex | OpenClaw |
|-|-|-|
| 子代理形态 | 任务级临时子代理 | 常驻人格化 Agent |
| 生命周期 | 随任务生灭 | 长期存在 |
| 协作方式 | 主代理派发，结果回传 | 共享文件 + 群聊路由 |
| 适用场景 | 编码任务拆解 | 个人助理多角色协作 |

## 案例：Codex 的并行工具执行

Codex 的 `tools/parallel.rs` 支持**无依赖工具的并行执行**：

```rust
// 模型一轮输出多个工具调用时，并行执行
let results = execute_tools_in_parallel(tool_calls).await;
```

**产品体验**：用户让 AI"读三个文件并总结"，三个文件读取是并行的，延迟从 3×T 降到 1×T。

## 工作流编排的四个模式

### 1. 顺序执行（Pipeline）

```
步骤 1 → 步骤 2 → 步骤 3
```

适用：步骤之间有依赖关系（先分析，再设计，再实现）

### 2. 并行执行（Parallel）

```
       ┌→ 步骤 2a ─┐
步骤 1 ┼→ 步骤 2b ─┼→ 步骤 3
       └→ 步骤 2c ─┘
```

适用：步骤之间无依赖（读多个文件、查多个数据源）

### 3. 条件分支（Branch）

```
步骤 1 → 判断 → 条件 A → 步骤 2A
              → 条件 B → 步骤 2B
```

适用：根据中间结果决定后续路径（测试通过 → 部署，测试失败 → 修复）

### 4. 循环迭代（Loop）

```
执行 → 评估 → 不满足 → 调整 → 再执行
         ↓
       满足 → 结束
```

适用：需要逐步逼近目标的场景（优化、调试、迭代设计）

## 编排的三种粒度（播客方法论）

《AI 产品经理实操手册》从 Satya Nadella 的访谈中提炼了编排的三种粒度：

| 粒度 | 适用场景 | PM 要做的 |
|-|-|-|
| 单 agent 直跑 | 简单生成类（摘要、翻译） | 写好 prompt + eval 门禁 |
| 多 agent 流水线 | 有明确步骤的复杂任务 | 定义步骤拆分、每步验收标准、交接格式 |
| agent 团队 | 需要协作的长任务 | 定义角色分工、共享上下文、冲突仲裁规则 |

**关键认知**：你最大的精力投入点在**定义任务包的边界和质检点**。边界定得清楚，agent 出错少；质检点放得对，出错了也兜得住。

## 对抗 Agent：Jyothi 的黑客松方案

Jyothi Nookula 用 GAN 启发的对抗架构赢得了公司黑客松：

```
一个 agent 构建 → 另一个 agent 找茬 → 反馈回去改进 → 循环到过质量线
```

- 灵感来自 Anthropic 博客的 adversarial agents 和 GAN 架构（生成器 vs 判别器）
- 启动 prompt 大意："构建一个对抗评估器来评估我的 agent，标准要宽，GAN 架构"
- 她强调：赢靠的不是写代码（写代码已经很容易），而是**"问题优先"的 PM 本能**

## PM 检查清单

设计 AI 工作流时，问自己：

- [ ] 复杂任务有没有显式的计划工具？（plan）

- [ ] 无依赖的步骤能不能并行？（parallel）

- [ ] 子任务能不能派生子代理？（multi_agents）

- [ ] 用户能不能看到任务进度？（透明度）

- [ ] 任务失败能不能从断点恢复？（韧性）

## 本周练习

1. 选一个你过去设计的复杂功能（比如"发布流程"），把它改写成 AI 工作流（用 plan 工具拆解）
2. 识别你产品中的"可并行"场景：哪些步骤之间没有依赖关系？
3. 设计一个"条件分支"工作流：根据中间结果决定后续路径

## 延伸阅读

- Codex 仓库：`codex-rs/core/src/tools/handlers/plan_spec.rs`（计划工具定义）
- Codex 仓库：`codex-rs/core/src/tools/handlers/multi_agents.rs`（多智能体派发）
- Codex 仓库：`codex-rs/core/src/tools/parallel.rs`（并行执行）
