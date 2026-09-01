# AI 产品经理访谈资源清单

> 整理日期：2026-08-15  
> 用途：收集 AI 产品经理亲自接受访谈的内容（播客 / 大师课），供持续学习与对照参考。

---

## 一、英文圈：AI 产品经理硬核访谈（最值得听）

### 1. Lenny's Podcast × Dianne Penn（Anthropic 首位技术 PM）⭐ 最推荐

- 单集：*Anthropic's first technical PM on token maxing, the jagged edge, and living in the future*
- 时间：2026-07-26，约 1 小时 34 分
- 嘉宾：Dianne Penn，Anthropic 研究团队 Product Head，2023 年加入时为第一位 technical PM（当时团队只有 5 个工程师），参与孵化 Claude Code、MCP、Skills、computer use、reasoning 等核心产品

**核心内容：**

- **Eval 驱动的开发循环**：evals 正在取代 PRD，是当前 AI 产品研发的核心机制
- **Token maxing**：产品人得像打磨像素一样去琢磨 token 消耗
- **Jagged edge（锯齿边缘）**：AI 能力不是均匀增长的，产品要守住模型还没跨过的边缘，用测试、复核和责任边界兜住
- 为什么 Claude「愿意反驳用户」是它成功的关键
- 个人方法：选一两个真实问题，亲自做、和别人一起做，先形成自己的判断，再让 AI 加速、反驳、复核

### 2. Lenny's Podcast × Cat Wu（Claude Code / Cowork 产品负责人）

- 单集：*How Anthropic's product team moves faster than anyone else*
- 时间：2026-04-23，约 1 小时 25 分
- 嘉宾：Cat Wu，Anthropic 负责 Claude Code 与 Cowork 的产品负责人

**核心内容：**

- 代码变便宜后，PM 的时间要花在：**决定做什么、让产品尽快到用户手里、把自动化推到真正可靠**
- 一句话：AI 产品经理，先缩短「想法到用户」的距离

### 3. Peter Yang × Alex Albert（Anthropic 研究 PM）

- 单集：*Inside How Anthropic Is Building the Next Claude*
- 时间：2026-05-17
- 嘉宾：Alex Albert，Claude 研究产品经理（Research PM）

**核心内容：**

- **把模型当产品来打造**：「每一个新模型开始时，我们都会明确它的要求是什么、希望它擅长什么」
- 模型开发更像"培养"：直到训练开始才知道它会成长成什么样
- 如何把用户反馈聚类成 **eval**（需求文件），判断能否变成需求文档或诊断问题的方式
- 对 Claude 意识/性格的内部研究：「当模型长时间自主执行任务时，它『关心什么』会变得和能力本身一样重要」

### 4. Jyothi Nookula AIPM 大师课（完整方法论框架）

- 嘉宾背景：AI 领域 13.5 年经验，12 项专利；曾任 Amazon（SageMaker）、Meta（PyTorch）、Netflix（Developer Platform）、Etsy 的 AIPM

**核心内容：**

- **两类 AIPM**：传统 PM + AI 功能（约 80%） vs AI 原生 PM（约 20%）——后者产品本身就是 AI（ChatGPT、Copilot、Claude、Cursor、Perplexity），本质是概率性产品
- **技术栈三层**：应用层 PM（约 60%，端到端用户体验）/ 平台层 PM（约 30%，给开发者建工具）/ 基础设施 PM（约 10%，向量库、GPU 调度等）
- **AIPM 与传统 PM 核心差异**：确定性 vs 概率性。要思考"用户可接受错误率是多少、信任何时崩溃、要不要确定性兜底系统"
- **数据是第一优先级**：垃圾进垃圾出，数据策略是前提条件
- **什么时候用 AI / 不用 AI**：适合=复杂模式识别、基于历史预测、大规模个性化；不适合=必须可解释、规则清晰、数据不足、需要快速上线
- **技术选型**：传统 ML（表格数据）→ 深度学习（感知类）→ 生成式 AI（自然语言）；Fine-tuning 不是第一选择，推荐顺序：Prompt → Context → RAG → Fine-tuning
- **成长路径**：不要只做项目，要做产品（找问题→构建→上线→有用户→修问题）；转型作品集 = 一个应用 + 一个 Agent + 一个 RAG 系统

---

## 二、中文圈：一线 AI 产品经理实战分享

### 5. 《The Z Prompt》V18：AI 搜索 PM 对谈

- 平台：小宇宙
- 嘉宾：01 年出生的大厂 AI 搜索产品经理 Tilly

**核心内容：**

- 从传统产品到 AI 产品的转型路径；AI 产品分为"原生产品"和"传统 + AI 增益"两大类
- **被实习生"教育"的震撼经历**：实习生用 AI 一天产出上千条高质量测试集、半小时做出高保真 Demo
- **AI 产品经理的"脏活累活"**：高质量数据标注需要深入分析用户需求，往往是 PM 亲自上阵
- 用 AI 写 PRD 为何招人反感：涉及复杂历史背景和底层逻辑时 AI 作用有限，且大家对 AI 的不信任感仍普遍存在

### 6. 《码农姐妹》27 期：传统 PM 如何切入 AI Agent 基建

- 平台：小宇宙 / Podwise
- 嘉宾：雪鱼，物理师范转型的 AI 应用基建产品负责人

**核心内容：**

- **AI Native 产研流程**：产品经理先搭 Demo 再写 PRD
- AI coding 模糊角色边界：PM 成为问题解决者，先动手验证再定义需求
- AI 反推产品设计：补完想法、识别无用功能
- **「氛围茧房」概念**：身处积极 AI 讨论氛围能促进学习和创新
- 个人案例：用 AI 开发应用「小龙手杖」

### 7. 喜马拉雅《对话 AI 产品经理》：AI 时代职场人的适应与自洽

- 主持人：职业顾问 阮妍 Yennis；嘉宾：AI 产品经理 阿桂
- 录制时间：2026 年初

**核心内容：**

- 从实施交付转 AI 产品经理的完整路径；「地狱级项目」全景复盘
- **祛魅心态**：不贩卖焦虑、不鼓吹速成，把 AI 当不完美的伙伴
- OpenClaw + Claude Skill 对未来影响的前瞻；AI 中台逐渐消失与「万能路由器」的未来
- 35+ 职场焦虑与 AI 转型必要性的现实讨论

---

## 三、收听优先级建议

| 优先级 | 资源 | 理由 |
|-|-|-|
| ⭐ 只听 1 期 | Dianne Penn × Lenny's | Anthropic 首位技术 PM，evals 驱动开发，与"管评测"方法论最呼应 |
| ⭐ 完整方法论 | Jyothi Nookula 大师课 | 两类 AIPM、三层技术栈、用/不用 AI 决策框架 |
| ⭐ 中文一线实操 | 《The Z Prompt》V18 | 00 后 AI 搜索 PM，最贴近中文职场现实 |

---

*来源：基于 2026 年公开播客与访谈整理。*
