# D7: 生产监控与Evals体系化

## 📍 开篇引子

课程最后一天，咱们聊个很多人忽视的问题：**你的评估是一次性的，还是持续运转的？**

见过太多团队上线前突击做评估，测完就扔到角落里。三个月后，产品迭代了好几轮，评估结果早就过时了。更可怕的是，你根本不知道产品现在"好不好"。

今天的重点是：**把Evals变成一个持续运转的系统**，而不是一次性的项目。

---

## 🎯 核心概念：两种评估场景，两套方法

在聊持续监控之前，先要区分两种评估场景：

### CI/CD中的Evals

在**发布前**做评估，类似传统软件的CI/CD流水线：

```text
代码变更 → 构建 → 单元测试 → Evals → 部署

```

这个阶段的Evals有**ground truth**（你知道正确答案），核心是：

- **回归测试**：确保改动没有破坏已有功能
- **断言检查**：满足预设标准才能发布
- **快反馈**：几分钟内知道结果

```Python
# CI中的Eval示例
def test_product_info_query():
    """测试产品查询场景"""
    test_cases = [
        {
            "query": "你们的高端套餐多少钱？",
            "expected_keywords": ["价格", "高端", "套餐"],
            "forbidden_patterns": ["不支持", "无法"],
        },
        {
            "query": "怎么申请退货？",
            "expected_keywords": ["退货", "申请", "流程"],
            "forbidden_patterns": ["请联系"],
        },
    ]
    
    for case in test_cases:
        response = call_product_ai(case["query"])
        
        # 断言检查
        for keyword in case["expected_keywords"]:
            assert keyword in response, f"缺少关键词: {keyword}"
        
        for pattern in case["forbidden_patterns"]:
            assert pattern not in response, f"包含禁止模式: {pattern}"

```

### 生产监控中的Evals

在**发布后**持续监控，核心挑战是：**没有ground truth**。

你不知道用户的问题"应该"怎么回答，你只能靠LLM-as-Judge或用户反馈来判断。

```text
用户Query → 系统回答 → LLM-as-Judge评估 → 监控Dashboard

```

这个阶段的Evals更侧重：

- **趋势追踪**：指标是变好还是变坏
- **异常检测**：有没有突然的性能下降
- **归因分析**：问题出在哪里

---

## 🛠️ 实操方法：构建Evals飞轮

Evals飞轮是持续改进的核心机制：

```text
错误分析 → 发现失败模式 → 编写Evals → 修复问题 → 生产监控 → 回到错误分析

```

### 飞轮 Step 1：错误分析（持续进行）

不要认为错误分析是一次性的活动，它是产品的"日常体检"。

```Python
import json
from datetime import datetime, timedelta
from collections import Counter

class ContinuousErrorAnalyzer:
    """持续错误分析"""
    
    def __init__(self, traces_store: str):
        self.traces_store = traces_store
        self.failure_patterns = Counter()
        self.failure_by_intent = {}
    
    def analyze_recent_traces(self, days: int = 7):
        """分析最近N天的失败案例"""
        cutoff = datetime.now() - timedelta(days=days)
        recent_traces = self.load_traces_since(cutoff)
        
        # 用LLM-as-Judge快速筛选失败case
        failures = []
        for trace in recent_traces:
            if self.is_failure(trace):
                failures.append(trace)
        
        # 按失败模式聚类
        patterns = self.cluster_failures(failures)
        
        # 按意图分析失败分布
        by_intent = self.analyze_by_intent(failures)
        
        # 生成报告
        return {
            "period": f"最近{days}天",
            "total_traces": len(recent_traces),
            "failure_count": len(failures),
            "failure_rate": len(failures) / len(recent_traces),
            "top_patterns": patterns.most_common(10),
            "failure_by_intent": by_intent,
        }
    
    def is_failure(self, trace: dict) -> bool:
        """判断是否是失败case"""
        # 用轻量级check快速筛选
        # 1. 用户有没有负反馈
        if trace.get("feedback") == "negative":
            return True
        
        # 2. LLM-as-Judge是否判定fail
        judge_result = self.quick_judge(trace)
        if judge_result == "fail":
            return True
        
        return False
    
    def cluster_failures(self, failures: list) -> Counter:
        """聚类失败模式"""
        patterns = Counter()
        for failure in failures:
            pattern = self.classify_failure(failure)
            patterns[pattern] += 1
        return patterns
    
    def classify_failure(self, failure: dict) -> str:
        """分类失败模式"""
        prompt = f"""
用户问题：{failure['query']}
系统回答：{failure['response']}

请识别失败的主要原因（选一个）：
- 意图理解错误
- 信息不完整
- 事实错误/幻觉
- 回答不相关
- 格式/输出问题
- 其他

输出：原因标签
"""
        result = call_llm(prompt)
        return parse_label(result)

```

### 飞轮 Step 2：监控与告警

```Python
from typing import Callable

class EvalsMonitor:
    """Evals监控器"""
    
    def __init__(self, eval_functions: dict):
        self.eval_functions = eval_functions
        self.baseline = None
        self.alert_callbacks = []
    
    def set_baseline(self, traces: list):
        """设置基准线"""
        results = self.run_evals(traces)
        self.baseline = {
            metric: results[metric]["mean"]
            for metric in results
        }
        print(f"基准线已设置：{self.baseline}")
    
    def check_for_drift(self, traces: list, threshold: float = 0.1):
        """检测指标漂移"""
        current = self.run_evals(traces)
        
        alerts = []
        for metric in self.baseline:
            if metric not in current:
                continue
            
            baseline_value = self.baseline[metric]
            current_value = current[metric]["mean"]
            
            # 计算相对变化
            if baseline_value == 0:
                change = abs(current_value)
            else:
                change = (current_value - baseline_value) / baseline_value
            
            if abs(change) > threshold:
                alerts.append({
                    "metric": metric,
                    "baseline": baseline_value,
                    "current": current_value,
                    "change": f"{change:.1%}",
                    "severity": "high" if abs(change) > threshold * 2 else "medium"
                })
        
        # 触发告警
        if alerts:
            for callback in self.alert_callbacks:
                callback(alerts)
        
        return alerts
    
    def run_evals(self, traces: list) -> dict:
        """运行所有evals"""
        results = {
            name: {"values": [], "sum": 0}
            for name in self.eval_functions
        }
        
        for trace in traces:
            for name, eval_fn in self.eval_functions.items():
                score = eval_fn(trace)
                results[name]["values"].append(score)
                results[name]["sum"] += score
        
        return {
            name: {
                "mean": r["sum"] / len(r["values"]) if r["values"] else 0,
                "count": len(r["values"]),
            }
            for name, r in results.items()
        }
    
    def on_alert(self, callback: Callable):
        """注册告警回调"""
        self.alert_callbacks.append(callback)

```

### 飞轮 Step 3：Guardrails vs Evaluators

这是个重要的概念区分：

|  | Guardrails | Evaluators |
|-|-|-|
| **时机** | Inline（同步） | 事后（异步） |
| **位置** | 在输出返回给用户之前 | 用户看到输出之后 |
| **目的** | 防止有害内容输出 | 评估质量、收集数据 |
| **速度要求** | 必须快（毫秒级） | 可以慢（秒/分钟级） |
| **典型应用** | 过滤敏感词、拦截幻觉 | 打分、归因、趋势分析 |

```Python
# Guardrails：同步拦截
def guardrail_response(query: str, response: str) -> tuple:
    """同步检查，必要时拦截或修改"""
    
    # 检查1：敏感内容过滤
    if contains_sensitive_content(response):
        return "", "该内容无法回答"
    
    # 检查2：格式验证
    if query.requires_json() and not is_valid_json(response):
        return "", "抱歉，格式出错"
    
    # 检查3：置信度检查
    confidence = estimate_confidence(response)
    if confidence < 0.5:
        return "", "这个问题我不确定..."
    
    return response, None


# Evaluators：异步评估
async def evaluate_response_async(trace_id: str):
    """异步评估，不阻塞返回"""
    trace = get_trace(trace_id)
    
    # LLM-as-Judge打分
    eval_result = judge_response(trace)
    
    # 存储评估结果
    store_eval_result(trace_id, eval_result)
    
    # 如果是失败case，加入分析队列
    if eval_result["pass"] == False:
        add_to_failure_analysis(trace)

```

---

## 📊 Agentic工作流评估的特殊性

如果你做的是Agentic产品（LLM调用工具、多步推理），评估要更复杂：

### 三层评估架构

```Python
class AgenticEvaluator:
    """Agentic工作流评估"""
    
    def evaluate(self, trace: dict) -> dict:
        # Layer 1: 端到端成功率
        overall_success = self.check_overall_success(trace)
        
        # Layer 2: 步骤级诊断
        step_results = self.diagnose_steps(trace)
        
        # Layer 3: 转移失败矩阵
        transfer_failures = self.analyze_transfer_failures(trace)
        
        return {
            "overall": overall_success,
            "steps": step_results,
            "transfer_failures": transfer_failures,
        }
    
    def check_overall_success(self, trace: dict) -> bool:
        """检查端到端是否完成目标"""
        # 检查最终输出是否解决了用户问题
        # 或者用户是否没有继续投诉
        return trace.get("outcome") == "success"
    
    def diagnose_steps(self, trace: dict) -> list:
        """诊断每一步的问题"""
        steps = trace.get("steps", [])
        diagnoses = []
        
        for i, step in enumerate(steps):
            diagnosis = {
                "step": i,
                "action": step.get("action"),
                "success": step.get("success", True),
                "issue": None
            }
            
            if not step.get("success"):
                diagnosis["issue"] = self.classify_step_failure(step)
            
            diagnoses.append(diagnosis)
        
        return diagnoses
    
    def analyze_transfer_failures(self, trace: dict) -> dict:
        """分析步骤之间的转移"""
        steps = trace.get("steps", [])
        transitions = []
        
        for i in range(len(steps) - 1):
            from_step = steps[i]["action"]
            to_step = steps[i+1]["action"]
            
            # 检查转移是否合理
            is_valid = self.is_valid_transition(from_step, to_step)
            
            transitions.append({
                "from": from_step,
                "to": to_step,
                "valid": is_valid
            })
        
        return {
            "transitions": transitions,
            "failure_rate": sum(1 for t in transitions if not t["valid"]) / len(transitions)
            if transitions else 0
        }

```

---

## 🚀 体系化落地建议

### 1. 从小开始

不要一上来就搞100个evals。先从3-5个核心指标开始，确保它们真的在跑、真的有人看。

### 2. 自动化一切能自动的

```YAML
# 每日自动化任务示例
cron: "0 9 * * *"  # 每天早上9点
tasks:
  - name: "run_regression_evals"
    command: "python eval_runner.py --suite regression"
    notify_on_fail: true
  
  - name: "check_drift"
    command: "python monitor.py --check-drift"
    alert_threshold: 0.1  # 10%漂移触发告警
  
  - name: "failure_analysis"
    command: "python analyzer.py --recent-days 1"
    output: "daily_failure_report.md"

```

### 3. 让数据流动起来

```text
用户行为 → Trace → LLM-as-Judge → 评估结果 → Dashboard
                ↓
          失败case → 人工Review → 新Evals → 改进

```

### 4. 定期Review评估本身

每季度问自己：

- 这套evals还在测"对的东西"吗？
- 评估结果和用户反馈一致吗？
- 有没有"测了但没人管"的evals？

---

## ⚠️ 避坑指南

**❌ 反模式1：指标太多没人看**你建了50个evals，但每个都只有你自己知道在干嘛。选3-5个核心指标，让团队都理解、都关注。

**❌ 反模式2：监控但不告警**你有了监控数据，但"飘了10%也没人管"。**没有告警的监控等于没监控**。

**❌ 反模式3：Evals和开发脱节**Evals团队和开发团队各干各的。Evals发现的问题，开发不知道；开发改了什么，Evals没测到。

---

## 🤔 今日一问

回顾你现在的AI产品：

1. 你的团队现在有"持续运行的evals"吗？还是只有上线前的一次性评估？
2. 如果要选3个核心指标来监控，你会选哪3个？
3. 你的错误分析流程是什么？是"出了问题再分析"还是"定期review"？

---

## 📚 延伸阅读

1. **Aporia的LLM监控指南** — 从产品视角讨论LLM监控，包括漂移检测、异常告警等。
2. **Evals飞轮实践（OpenAI Cookbook）** — 官方收录的Evals飞轮案例，看完会对持续改进有更清晰的认识。
3. **LangSmith的评估功能** — 如果你用LangChain，LangSmith的评估功能值得研究。

---

## 🎓 课程总结

到这里，7天的AI Evals课程就全部结束了。

我们从"什么是Evals"开始，经历了：

- **D1**: 建立评估的基本认知
- **D2**: 学会错误分析，找到真正的问题
- **D3**: 设计LLM-as-Judge
- **D4**: 校准你的裁判
- **D5**: 用合成数据突破数据瓶颈
- **D6**: 搞定RAG评估
- **D7**: 把一切体系化、持续化

核心记住一句话：**Evals不是一次性的项目，是持续运转的系统**。找到问题 → 量化问题 → 修复问题 → 监控问题，这个飞轮转起来，你的AI产品才会越来越好。

祝你的产品Eval都能Pass。🚀
