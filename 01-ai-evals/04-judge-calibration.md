# D4: 评估器校准——裁判也需要考核

## 📍 开篇引子

你有没有遇到过这种情况：LLM裁判告诉你"这条回答fail"，你看了眼case，觉得明明挺好啊。然后你让裁判解释，它支支吾吾说了堆"因为整体质量不够高"之类的废话。

**LLM裁判不是圣人，它的判断也需要被考核。** 今天聊聊怎么校准你的AI裁判。

---

## 🎯 核心概念：什么是"校准"？

先解释一下术语。**校准（Calibration）** 是指让你的模型的输出概率（或判断）与实际情况保持一致。

比如：

- 一个完美的校准模型：它说"我有80%把握这条回答是good"，实际确实80%的时候是good
- 一个失准的模型：它说"我有80%把握"，但实际只有50%的时候是good

对于LLM-as-Judge，校准的目标是：**让AI的判断与人（尤其是领域专家）的判断尽可能一致**。

### 为什么要校准？

原因很现实：**不校准的LLM-as-Judge，数据参考价值有限。**

假设你跑了一周evals，通过率从30%变成了50%。你是高兴还是怀疑？

- 如果裁判没校准：可能是你真的改进了，也可能是裁判这周心情好
- 如果裁判校准过：你可以更有信心地说"通过率提升了20个百分点"

更重要的是：**校准能帮你发现裁判的偏见**。

常见的偏见包括：

- **宽容偏见**：几乎所有回答都pass，因为它是"友好的AI"
- **严厉偏见**：总想挑毛病，动不动就fail
- **位置偏见**：A在B前面就给高分，后面就给低分
- **长度偏见**：长的回答更容易得高分

校准能帮你量化这些偏见，然后针对性地修正。

---

## 🛠️ 实操方法：校准你的LLM裁判

### 方法一：真值数据集对比

最直接的方法：**准备一批人工标注的数据，用它来测试你的裁判**。

```Python
import json

# 假设你有人工标注的ground truth
CALIBRATION_DATASET = [
    {
        "query": "你们支持7天退货吗？",
        "response": "支持的，7天内可以申请退货，需要保持商品完好。",
        "human_label": "pass",
    },
    {
        "query": "你们支持7天退货吗？",
        "response": "支持的。",
        "human_label": "fail",
    },
    {
        "query": "这个套餐包含什么服务？",
        "response": "包含基础客服、工单系统和知识库。",
        "human_label": "pass",
    },
    # ... 至少50条
]

def calibrate_judge(judge_prompt: str, calibration_data: list):
    """校准裁判"""
    results = []
    
    for item in calibration_data:
        # 获取裁判判断
        judge_output = call_judge(
            judge_prompt,
            query=item["query"],
            response=item["response"]
        )
        judge_label = parse_judge_output(judge_output)  # 解析为 pass/fail
        
        results.append({
            "human_label": item["human_label"],
            "judge_label": judge_label,
            "match": human_label == judge_label
        })
    
    # 计算准确率
    accuracy = sum(r["match"] for r in results) / len(results)
    
    # 计算TPR和TNR
    true_positives = sum(1 for r in results if r["human_label"] == "pass" and r["judge_label"] == "pass")
    false_negatives = sum(1 for r in results if r["human_label"] == "pass" and r["judge_label"] == "fail")
    true_negatives = sum(1 for r in results if r["human_label"] == "fail" and r["judge_label"] == "fail")
    false_positives = sum(1 for r in results if r["human_label"] == "fail" and r["judge_label"] == "pass")
    
    tpr = true_positives / (true_positives + false_negatives)  # 真阳性率
    tnr = true_negatives / (true_negatives + false_positives)  # 真阴性率
    
    return {
        "accuracy": accuracy,
        "tpr": tpr,  # 裁判识别"好回答"的能力
        "tnr": tnr,  # 裁判识别"坏回答"的能力
        "total": len(results)
    }

```

### 方法二：理解TPR和TNR

TPR（True Positive Rate）和TNR（True Negative Rate）是校准的核心指标：

| 指标 | 含义 | 理想值 |
|-|-|-|
| TPR（真阳性率） | 在所有"真正是好回答"的case中，裁判正确识别了多少 | 接近1.0 |
| TNR（真阴性率） | 在所有"真正是坏回答"的case中，裁判正确识别了多少 | 接近1.0 |

**TPR和TNR都很重要**。一个"老好人"裁判TPR很高（比如0.95）但TNR很低（0.4），它会把所有坏回答都误判成好的；一个"杠精"裁判则相反。

### 方法三：用偏差修正预测结果

如果你的裁判有系统性偏差，你可以通过数学方法修正结果：

```Python
def adjust_with_calibration(
    raw_pass_rate: float,
    tpr: float,
    tnr: float
) -> float:
    """
    根据校准数据修正通过率
    
    原理：如果裁判的TPR是0.8，TNR是0.9
    说明它识别"好"的本事差一些（漏掉了20%）
    我们可以反推真实的通过率
    """
    # 简化模型下的真实通过率估算
    # 这是一个近似解，实际更复杂
    
    # 假设原始通过率是 P(judge=pass)
    # judge=pass 的概率 = P(human=pass) * TPR + P(human=fail) * (1-TNR)
    # 反推 P(human=pass)
    
    # 用近似公式
    adjusted_rate = (raw_pass_rate - (1 - tnr)) / tpr if tpr > 0 else raw_pass_rate
    
    return max(0, min(1, adjusted_rate))  # 限制在[0,1]范围

# 使用示例
raw_rate = 0.65  # 裁判说通过率65%
calibration_result = calibrate_judge(judge_prompt, calibration_data)

adjusted_rate = adjust_with_calibration(
    raw_rate,
    calibration_result["tpr"],
    calibration_result["tnr"]
)

print(f"原始通过率: {raw_rate:.1%}")
print(f"校准后: {adjusted_rate:.1%}")

```

**注意**：这个修正公式是简化的，实际应用中要根据具体情况调整。复杂场景建议用贝叶斯方法。

### 方法四：选择标注策略——"仁慈独裁者"还是"民主投票"？

如果你需要人工标注ground truth，有个决策点：**一个人标还是多个人投票？**

我的建议：**一个人标注，但选对人**。

原因：

- 多人投票会稀释专家判断（少数服从多数，专家意见被平均掉了）
- LLM-as-Judge本身就是"一个人的判断"，你用多人投票校准它，逻辑上不匹配
- 一致性比正确性重要——你需要一个"稳定的标准"，而不是"每个人都觉得差不多"

选谁？

- **领域专家**（比如你们的客服主管、或者资深PM）
- 这个人要有**明确的判断标准**，不是凭感觉
- 这个人要**愿意写理由**，不能只打标签

---

## 📊 校准目标：80%一致性

什么时候可以信任你的裁判？

**80%的人机一致性是一个重要里程碑。** 达到这个水平后：

- 裁判的判断可以作为快速筛选工具
- 人工可以聚焦在边缘case和校准检查
- 迭代效率大幅提升

```Python
def should_trust_judge(calibration_result: dict) -> bool:
    """判断裁判是否可信"""
    accuracy = calibration_result["accuracy"]
    tpr = calibration_result["tpr"]
    tnr = calibration_result["tnr"]
    
    # 基本条件：准确率 > 80%
    if accuracy < 0.80:
        return False
    
    # 平衡性：TPR和TNR差距不能太大
    # 一个0.95、一个0.5，说明有严重偏见
    if abs(tpr - tnr) > 0.3:
        return False
    
    # 两者都不能太低
    if tpr < 0.70 or tnr < 0.70:
        return False
    
    return True

```

---

## ⚠️ 避坑指南

**❌ 反模式1：拿训练数据当测试数据**你用同一批数据既训练prompt又测试效果，那准确率肯定是虚高的。必须留出独立的测试集。

**❌ 反模式2：校准一次就完事了**LLM是"非稳定性"系统，你的prompt改一点、模型版本换一下、温度调一下，都可能影响判断。建议每次重大变更后重新校准。

**❌ 反模式3：忽视模型本身的影响**你用的是GPT-4o校准，但生产环境是GPT-4o-mini——两者的判断能力差异很大。建议**生产用什么，校准就用什么**。

---

## 🤔 今日一问

假设你的客服AI上线了，你用它来判断"用户问题是否被解决"：

1. 你会用什么标准来定义"问题被解决"？
2. 你打算请谁来标注ground truth？
3. 如果裁判的TPR很高但TNR很低，说明了什么？你会怎么应对？

---

## 📚 延伸阅读

1. **Scale AI的校准指南** — 做了大量人工标注的团队，他们的校准经验很实用，特别是关于标注员选择的部分。
2. **AI评估中的常见偏见（Anthropic）** — 官方出品，列举了LLM-as-Judge的各种偏见来源，读完你会对"什么时候不能相信裁判"有清晰认知。
3. **置信度校准的机器学习方法** — 偏学术，但如果你想深入理解校准背后的统计学原理，这篇论文是好起点。

---

*下期预告：校准好了裁判，你还需要高质量的数据来喂它。但真实数据不够、敏感数据不能用怎么办？明天聊聊合成数据——这是没有真实数据时的突围之道。*
