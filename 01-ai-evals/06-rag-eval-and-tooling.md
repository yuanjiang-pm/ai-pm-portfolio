# D6: RAG评估与标注工具

## 📍 开篇引子

RAG（检索增强生成）是现在AI产品的标配。但我发现很多团队在评估RAG时犯了一个致命错误：**把检索和生成混在一起评**。

他们只看最终回答质量，却不知道问题是出在"没检索到对的文档"，还是"检索到了但生成砸了"。这两个问题的解法完全不同，混在一起评就是瞎子摸象。

今天聊聊RAG评估的正确姿势。

---

## 🎯 核心概念：RAG评估必须"分家"

RAG系统本质上是个**两阶段流水线**：

```text
用户Query → 检索模块 → Top-K文档 → 生成模块 → 最终回答

```

**检索阶段**的目标是：找到跟用户问题最相关的文档。**生成阶段**的目标是：基于检索到的文档，生成准确、完整的回答。

这两个阶段的目标完全不同，评估方法也完全不同：

| 阶段 | 评估什么 | 怎么评估 |
|-|-|-|
| 检索 | 文档相关性排序 | IR指标（Recall、Precision、MRR） |
| 生成 | 回答质量 | LLM-as-Judge（忠实度、完整性） |

**如果你只评最终回答，你永远不知道问题在哪。**

打个比方：厨师做了一道难吃的菜。原因可能是：

- 食材不新鲜（检索失败）
- 厨艺不行（生成失败）
- 或者两者都有问题

如果你只看"菜好不好吃"，你怎么知道该换食材还是该换厨师？

---

## 🛠️ 实操方法：三层RAG评估框架

### Tier 1：传统IR指标（评估检索）

```Python
from typing import List

def calculate_retrieval_metrics(
    query: str,
    retrieved_docs: List[str],
    relevant_docs: List[str],  # ground truth相关文档
    k: int = 5
) -> dict:
    """
    计算检索指标
    """
    # Recall@k: top-k中召回的比例
    retrieved_at_k = set(retrieved_docs[:k])
    relevant = set(relevant_docs)
    
    recall_at_k = len(retrieved_at_k & relevant) / len(relevant) if relevant else 0
    
    # Precision@k: top-k中准确的比例
    precision_at_k = len(retrieved_at_k & relevant) / k if k > 0 else 0
    
    # MRR: 第一个相关文档的位置的倒数
    mrr = 0
    for i, doc in enumerate(retrieved_docs):
        if doc in relevant:
            mrr = 1 / (i + 1)
            break
    
    return {
        "recall@k": recall_at_k,
        "precision@k": precision_at_k,
        "mrr": mrr,
    }

# 使用示例
metrics = calculate_retrieval_metrics(
    query="退货政策是什么",
    retrieved_docs=["doc_A", "doc_B", "doc_C", "doc_D"],
    relevant_docs=["doc_B", "doc_E"],  # 实际上只有B和E相关
    k=4
)
# recall@4 = 1/2 = 0.5（只召回了B，没召回E）
# precision@4 = 1/4 = 0.25
# MRR = 1/2 = 0.5（B在第2位）

```

### Tier 2：Context-Relevance评估（评估检索质量）

IR指标看的是"有没有找到对的文档"，但**文档找到了不代表文档有用**。

```Python
def evaluate_context_relevance(
    query: str,
    retrieved_doc: str
) -> dict:
    """
    评估单条检索结果与query的相关性
    """
    prompt = f"""
你是一个检索评估专家。

用户问题：{query}

检索到的文档：
---
{retrieved_doc}
---

请评估：
1. 这篇文档是否包含回答问题所需的信息？（完全包含/部分包含/不包含）
2. 这篇文档的信息是否准确？（准确/部分准确/有误）
3. 给出0-10的相关性评分

输出格式：
相关度标签：...
信息准确性：...
相关性评分：X/10
"""
    response = call_llm(prompt)
    return parse_evaluation(response)

```

### Tier 3：Answer质量评估（评估生成）

基于检索到的文档，评估最终回答的质量：

```Python
def evaluate_answer_quality(
    query: str,
    retrieved_docs: List[str],
    answer: str
) -> dict:
    """
    评估生成阶段的质量
    """
    docs_text = "\n\n---\n\n".join(retrieved_docs)
    
    prompt = f"""
你是一个回答质量评估专家。

用户问题：{query}

系统检索到的参考文档：
---
{docs_text}
---

系统生成的回答：
---
{answer}
---

请从以下维度评估：

1. **忠实度（Faithfulness）**：回答是否与参考文档一致？是否存在幻觉？
2. **完整性（Completeness）**：回答是否完整覆盖了用户问题？
3. **相关性（Relevance）**：回答是否针对用户问题？

对于每个维度，给出：
- 评分（pass/fail）
- 一句话理由

最后，给出总体评价。
"""
    response = call_llm(prompt)
    return parse_evaluation(response)

```

### 诊断矩阵：快速定位问题

```Python
def rag_diagnosis_matrix(results: list) -> dict:
    """
    诊断RAG系统的瓶颈在哪里
    """
    retrieval_good = 0
    generation_good = 0
    both_good = 0
    both_bad = 0
    
    for r in results:
        retrieval_ok = r["retrieval_metrics"]["recall@k"] >= 0.8
        generation_ok = r["answer_evaluation"]["overall"] == "pass"
        
        if retrieval_ok and generation_ok:
            both_good += 1
        elif retrieval_ok and not generation_ok:
            retrieval_good += 1
        elif not retrieval_ok and generation_ok:
            generation_good += 1
        else:
            both_bad += 1
    
    return {
        "检索好生成好": both_good,
        "检索好生成差": retrieval_good,  # 问题在生成
        "检索差生成好": generation_good,  # 检索差但生成"编"出来了
        "检索差生成差": both_bad,  # 两边都有问题
    }

```

---

## 🛠️ 标注工具设计：构建审核界面

评估RAG，光有指标不够，你需要一个人工审核的界面。

### 核心原则

1. **展示完整Trace，而非仅最终输出**用户需要看到：Query → 检索文档 → 生成回答 → 评估结果
2. **格式化人类可读的内容**不要展示原始JSON，让非技术人员也能看懂
3. **支持快速标注和反馈**点一下就能打分，比填表单快10倍

### 简单实现

```Python
from datetime import datetime
import json

class RAGEvaluationInterface:
    """RAG评估人工审核界面（命令行版）"""
    
    def __init__(self, traces: list):
        self.traces = traces
        self.annotations = []
    
    def display_trace(self, trace: dict):
        """展示一条trace"""
        print("=" * 60)
        print(f"[{trace.get('id', 'N/A')}] {trace.get('timestamp', '')}")
        print("=" * 60)
        
        print("\n📝 用户问题:")
        print(f"   {trace['query']}")
        
        print("\n📄 检索到的文档:")
        for i, doc in enumerate(trace.get('retrieved_docs', [])):
            print(f"   [{i+1}] {doc[:200]}...")  # 截断显示
        
        print("\n🤖 系统回答:")
        print(f"   {trace['response']}")
        
        if 'ground_truth' in trace:
            print("\n✅ 参考答案:")
            print(f"   {trace['ground_truth']}")
    
    def prompt_annotation(self) -> dict:
        """收集人工标注"""
        print("\n" + "-" * 40)
        print("请评估（输入编号）：")
        print("  1. 检索质量 - 文档相关吗？")
        print("  2. 生成质量 - 回答满意吗？")
        print("  3. 整体评价 - 问题解决了吗？")
        print("  q. 跳过这条")
        print("-" * 40)
        
        annotation = {
            "retrieval_quality": input("检索 [1-差 2-中 3-好]: "),
            "generation_quality": input("生成 [1-差 2-中 3-好]: "),
            "overall": input("整体 [1-差 2-中 3-好]: "),
            "notes": input("备注（可选）: "),
        }
        
        return annotation
    
    def run_review(self, n: int = None):
        """运行审核流程"""
        traces_to_review = self.traces if n is None else self.traces[:n]
        
        for i, trace in enumerate(traces_to_review):
            print(f"\n[{i+1}/{len(traces_to_review)}]")
            self.display_trace(trace)
            
            annotation = self.prompt_annotation()
            
            if annotation["retrieval_quality"].lower() != "q":
                self.annotations.append({
                    "trace_id": trace.get("id"),
                    "annotation": annotation,
                    "timestamp": datetime.now().isoformat()
                })
    
    def export_annotations(self, filepath: str = "annotations.json"):
        """导出标注结果"""
        with open(filepath, "w") as f:
            json.dump(self.annotations, f, ensure_ascii=False, indent=2)
        print(f"\n已保存 {len(self.annotations)} 条标注到 {filepath}")


# 使用示例
interface = RAGEvaluationInterface(traces)
interface.run_review(n=20)
interface.export_annotations()

```

### Web界面建议

如果你是技术团队，建议做一个简单的Web界面：

| 功能 | 说明 |
|-|-|
| 左侧面板 | 展示Query + 检索文档 |
| 右侧面板 | 展示生成回答 |
| 底部 | 快速打分按钮 + 备注输入框 |
| 快捷键 | 1/2/3快速打分，N跳过，方向键切case |

核心是**让标注员能用最快的速度完成标注**，UI越简洁越好。

---

## 📊 标注流程最佳实践

### Step 1: 定义评估维度

在开始标注之前，明确你要收集什么数据：

```Python
ANNOTATION_DIMENSIONS = {
    "retrieval": {
        "relevance": "检索文档与Query的相关程度",
        "coverage": "是否覆盖了回答问题所需的全部信息",
    },
    "generation": {
        "faithfulness": "回答是否忠实于检索到的文档",
        "completeness": "回答是否完整解答了用户问题",
        "conciseness": "回答是否简洁，没有冗余信息",
        "safety": "回答是否有安全风险",
    },
    "overall": {
        "user_satisfaction": "如果你是用户，你满意吗？",
    }
}

```

### Step 2: 选择指标类型

| 指标类型 | 适用场景 | 示例 |
|-|-|-|
| 二元判断 | 需要明确对错 | 是否有幻觉？是否安全？ |
| 量表评分 | 需要质量梯度 | 完整性1-5分 |
| 排序 | 比较多个方案 | A/B/C哪个更好？ |
| 自由文本 | 需要解释性数据 | 说说你的判断理由 |

**推荐组合**：主要用二元判断 + 自由文本备注（理由）

### Step 3: 建立标注指南

给标注员一份清晰的指南：

```text
📋 标注指南

【忠实度评判标准】
- pass: 回答内容都可以从检索文档中找到依据
- fail: 回答中包含文档中没有的信息（幻觉）

【完整性评判标准】
- pass: 回答覆盖了用户问题的主要方面
- fail: 回答遗漏了重要信息，或没有直接回答问题

```

---

## ⚠️ 避坑指南

**❌ 反模式1：只评最终答案**"最终答案看着还行"——你不知道是检索好还是生成好。必须分层评估。

**❌ 反模式2：忽略检索质量**检索是RAG的瓶颈。大多数RAG问题，优化检索比优化生成更有效。

**❌ 反模式3：标注界面太复杂**如果你让标注员填10个字段、点5个下拉框，他们要么罢工要么乱填。**简单 > 全面**。

---

## 🤔 今日一问

假设你的RAG系统最近用户投诉"回答不准确"：

1. 你怎么用三层评估框架来定位问题？
2. 你会设计什么人工审核界面来收集反馈？
3. 如果"检索差但生成好"的case很多，说明了什么？你会怎么改进？

---

## 📚 延伸阅读

1. **RAGAS评测框架** — 专门针对RAG的评估框架，支持faithfulness、answer relevance等指标，有代码有论文。
2. **Jason Liu的RAG评估博客** — 他提出的6种RAG评估维度框架（Faithfulness、Relevance等），被广泛引用。
3. **Label Studio** — 开源标注工具，支持自定义标注界面，适合团队做RAG标注。

---

*下期预告：评估框架搭好了，标注工具也上线了。但这只是"一次性评估"，真正的挑战是"持续监控"。明天聊聊怎么在生产环境中运行evals，以及如何构建Evals飞轮。*
