# 07 · 从零训练 64M 小模型（预训练 → 微调 → 部署 → API 化）

基于开源项目 minimind 从零完成一次完整的 LLM 训练闭环：Mac M1 8GB 本地小样本试跑、云端 GPU 全量训练（约 10 元）、Ollama 本地部署、OpenAI 规范 API 化并接入真实 IDE 客户端。

> 这份材料想证明的是：理解 LLM 不需要百万级算力，一个 63.91M 参数的小模型（fp16 权重约 128MB）就能把预训练、SFT、模式坍缩、格式转换、适配层工程这些机制全部亲手踩一遍。

**1 篇 · 约 5,200 字**

| 篇目 | 字数 |
|---|---:|
| [从零训练一个 63.91M 参数的中文小模型：预训练、微调与本地部署实录](00-from-zero-to-api.md) | ~5,200 |

## 这篇里有什么

- **预训练**：交叉熵任务本质、loss 的两个锚点读数（ln(6400)≈8.76 起点锚、PPL 质量锚）、本地 vs 云端全量对照
- **微调 SFT**：loss mask 与"换脑"、模式坍缩最贵一课（loss 降了模型反而更差）、10 元对照实验证伪"小模型装不下知识"
- **本地部署**：.pth → safetensors → GGUF 三次格式转换各在转换什么、BPE pre-tokenizer 补丁、推理模板逐字节对齐
- **API 化**：FastAPI 鉴权代理、接入真实客户端后的十轮排障实录（UTF-8 劈字符、分段 content、Agent 注入模板、复读熔断）、温度策略 A/B 裁决（temp 0 复读率 95% vs temp 0.7 归零）

## 一句话心得

**模型的表现 = 模型 × 服务配置 × 输入卫生。** 同一个权重，任何一条拉胯都表现为"模型变笨了"——这句话是整篇实训的压缩包。

---

> 训练代码与数据处理基于 [jingyaogong/minimind](https://github.com/jingyaogong/minimind)（Apache-2.0），部署依赖 [llama.cpp](https://github.com/ggml-org/llama.cpp) 与 [Ollama](https://ollama.com)。
