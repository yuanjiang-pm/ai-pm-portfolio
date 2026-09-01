# NCA-GENL备考 D7：软件开发（Python库与LLM应用开发）

## 📖 今日主题

LLM应用开发的三件套：HuggingFace加载模型、PyTorch训练、LangChain编排

---

## 概念讲解

### HuggingFace Transformers

HuggingFace不只是模型库，它是一个完整的ML工具生态：

| 组件 | 作用 |
|-|-|
| **Transformers** | 模型库，主流模型全覆盖 |
| **Datasets** | 数据集加载和处理 |
| **Tokenizers** | 分词器，文本→数字 |
| **Accelerate** | 分布式训练加速 |
| **PEFT** | 参数高效微调工具 |

**核心用法**：

```Python
from transformers import pipeline

# 加载模型，三行代码搞定
generator = pipeline("text-generation", model="gpt2")
result = generator("今天天气真", max_length=50)

```

Pipeline封装了预处理→推理→后处理的全流程，用起来超简单。

---

### PyTorch基础

PyTorch是深度学习的"通用语言"，核心概念：

```text
📦 Tensor（张量）
   └─ GPU上的多维数组，计算基本单位
   
🔄 自动求导 (Autograd)
   └─ 自动计算梯度，反向传播的基础
   
🔁 训练循环
   ┌─ 前向传播：输入→预测
   ├─ 计算损失：预测 vs 真实
   ├─ 反向传播：计算梯度
   └─ 更新参数：optimizer.step()

```

**类比**：PyTorch训练就像教小孩骑自行车：

- Tensor = 自行车
- Forward = 骑
- Loss = 偏离程度
- Backward = 感觉偏离后大脑调整
- Optimizer = 控制身体平衡

---

### LangChain

LangChain是LLM应用的"编排框架"，帮你把大模型和其他工具串联起来：

**核心模块**：

- **Chains**：把多个步骤串成流水线
- **Agents**：让模型自主决定调用什么工具
- **Tools**：接入外部能力（搜索、数据库、API）
- **Memory**：让对话有记忆

```Python
# LangChain Agent示例逻辑
agent = initialize_agent(
    tools=[search_tool, calculator_tool],
    llm=llm,
    agent="zero-shot-react-description"
)
agent.run("北京今天适合穿什么？")

```

**为什么产品经理要了解LangChain**：

1. 理解AI产品技术可行性
2. 评估开发复杂度和成本
3. 和工程师在同一频道沟通

---

### 模型部署

把模型从实验室搬到生产环境的流程：

```text
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  模型导出   │ →  │  API封装    │ →  │  容器化部署  │
│  (ONNX等)   │    │  (FastAPI)  │    │  (Docker)   │
└─────────────┘    └─────────────┘    └─────────────┘

```

**常见部署架构**：

- **同步推理**：实时响应，适合聊天机器人
- **异步推理**：批量处理，适合文档分析
- **推理管线**：预处理→推理→后处理串联

---

## 关键对比

| 工具/库 | 定位 | 核心价值 |
|-|-|-|
| HuggingFace Transformers | 模型库 | 开箱即用的预训练模型 |
| PyTorch | 深度学习框架 | 灵活的训练和推理 |
| LangChain | 应用编排框架 | 快速搭建LLM应用 |
| FastAPI | Web框架 | 模型服务化 |

---

## 🕯️ 易错陷阱

> **⚠️ 考试常见坑点：HuggingFace不只是模型库，还有Datasets、Tokenizers等组件！**

> 很多同学以为HuggingFace只有一个Transformers库，实际上它是一个完整的生态，包括数据处理、分词、微调等全套工具。

---

## 练习题

### Q1: 使用HuggingFace加载一个文本生成模型，最快的方式是？

A) 从PyTorch手动构建模型架构B) 使用pipeline API一行加载C) 自己训练一个新模型D) 只能用TensorFlow加载

**答案：B解析**：pipeline是HuggingFace的高级API，封装了模型加载、分词、推理全流程，三行代码就能用。

---

### Q2: PyTorch中自动求导系统(Autograd)的作用是？

A) 自动进行模型推理B) 自动计算梯度，支持反向传播C) 自动选择最优学习率D) 自动管理GPU内存

**答案：B解析**：Autograd是PyTorch的核心特性，能自动追踪张量操作并计算梯度，是深度学习训练的基础。

---

### Q3: LangChain中Agent和Chain的主要区别是？

A) 没有区别，是同一个概念B) Chain是固定流程，Agent可以动态决定下一步行动C) Agent只能处理文本，Chain可以处理图像D) Chain不需要LLM，Agent必须用LLM

**答案：B解析**：Chain是预定义的固定流程，Agent则能根据输入自主决定调用哪个工具、更适合复杂多变的任务场景。

---

**📝 下期预告：D8 实验设计——A/B测试、交叉验证、假设检验！**
