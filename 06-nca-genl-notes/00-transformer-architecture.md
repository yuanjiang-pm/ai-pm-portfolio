# D0：Transformer架构

## 今日主题

**从RNN到Transformer：一个"并行阅读"的架构如何颠覆了整个NLP——Attention is All You Need.**

---

## 一、概念讲解

### 先说清楚一件事：Transformer到底解决了什么问题？

想象你在读一份100页的合同：

**RNN的做法**：从第1页逐页读到第100页，读到第90页时已经忘了第3页写了什么——因为"记忆"一路传过来，越传越稀薄。这叫**长距离依赖问题**。

**Transformer的做法**：把100页全部铺在桌上，你想看哪页就看哪页，而且可以**同时看所有页**——不需要一页一页翻。这就是"并行处理"。

核心区别：

- RNN必须**顺序处理**（一个token接一个token），无法并行，且远距离信息会衰减
- Transformer通过**注意力机制**让每个token直接"看到"所有其他token，距离不再是问题

---

### Transformer整体架构：编码器-解码器

原版Transformer（Vaswani et al., 2017）是Encoder-Decoder结构，用于机器翻译：

```
输入序列 → [Encoder × N] → 编码表示 → [Decoder × N] → 输出序列
```

- **Encoder（编码器）**：理解输入——"读懂源语言"
- **Decoder（解码器）**：生成输出——"翻译成目标语言"

**关键：后来的BERT只用Encoder，GPT只用Decoder，T5两个都用。** 这就是D1要讲的内容，但先把零件搞清楚。

---

### 零件一：Token Embedding + 位置编码

Transformer的输入不是文字，是向量。两步走：

**第一步：Token Embedding（词嵌入）**

- 把每个token映射成一个d_model维的向量（原论文d_model=512）
- 本质：一个查找表，token ID → 向量

**第二步：Positional Encoding（位置编码）**

⚠️ **考试必考：为什么Transformer需要位置编码？**

因为Transformer是**并行处理**所有token的——不像RNN天生有顺序。如果没有位置信息，"猫吃鱼"和"鱼吃猫"对Transformer来说完全一样。

**原论文的正弦位置编码公式：**

```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```

- pos：token在序列中的位置（0, 1, 2, ...）
- i：维度索引
- 2i用sin，2i+1用cos——每个维度用不同频率的正弦/余弦函数

**为什么用三角函数？**

1. 每个位置有唯一的编码
2. 模型可以学到相对位置关系（因为sin(a+b)可以用sin(a)和cos(b)表示）
3. 可以外推到比训练时更长的序列

**其他位置编码方式**：学习式位置编码（Learned Positional Embedding）——直接把位置当参数训练，BERT和GPT用的是这种。

---

### 零件二：自注意力机制（Self-Attention）——Transformer的灵魂

这是考试最高频的知识点，没有之一。

#### 直觉理解

一句话："The animal didn't cross the street because **it** was too tired."

当模型处理"it"这个词时，注意力机制会让"it"去"看"句子中的其他词，发现"it"和"animal"关系最强——这就是注意力在做的事。

#### Q、K、V——三个核心矩阵

⚠️ **必须记住：**

| 矩阵 | 类比 | 作用 |
|-|-|-|
| **Query（Q）** | 搜索关键词 | "我想要什么信息？"——当前token在寻找什么 |
| **Key（K）** | 文档标题/标签 | "我有什么信息？"——每个token能提供什么 |
| **Value（V）** | 文档正文 | "我的实际内容"——每个token的实际信息 |

**工作流程**：

1. 每个token生成自己的Q、K、V（通过三个线性变换矩阵W_Q、W_K、W_V）
2. 用Q和所有K做点积，算出"我跟你有多相关"——这就是**注意力分数**
3. 用softmax归一化，得到**注意力权重**（0到1，加起来=1）
4. 用权重对V加权求和，得到该token的**新表示**

#### 缩放点积注意力公式

⚠️ **考试必背公式：**

```
Attention(Q, K, V) = softmax(QK^T / √d_k) · V
```

逐部分拆解：

1. **QK^T**：Q和K的点积，算相似度。shape = (seq_len, seq_len)
2. **÷ √d_k**：**缩放（Scaling）**——这是最容易被忽略但考试爱考的点
3. **softmax**：归一化为概率分布
4. **· V**：用注意力权重对V加权求和

**为什么要除以√d_k？**

⚠️ **考试陷阱：**

当d_k很大时，QK^T的值会很大（因为点积的方差随维度增长），导致softmax的输入落进饱和区，梯度变得极小——模型几乎学不动。除以√d_k把方差拉回1，让softmax工作在有效区间。

**记忆口诀**：点积太大→softmax饱和→梯度消失→学不动→除以√d_k救场。

---

### 零件三：多头注意力（Multi-Head Attention）

一个注意力头只学一种"关注模式"。多头 = 多个注意力头并行计算，让模型同时关注不同类型的关系。

**做法**：

- 把Q、K、V分别投影h次（原论文h=8），每次投影到d_k = d_model/h = 64维
- 每个头独立算注意力
- 拼接所有头的输出，再做一个线性变换

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) · W_O

其中 head_i = Attention(QW_i^Q, KW_i^K, VW_i^V)
```

**为什么多头比单头好？**

类比：读一句话时，你可能同时关注语法关系、语义关系、指代关系——单头只能学一种模式，多头可以同时学多种。

**参数量没有增加**：8个64维头 vs 1个512维头，总计算量一样，但表达能力更强。

---

### 零件四：Add & Norm（残差连接 + 层归一化）

每个子层（注意力层、FFN层）后面都跟一个"Add & Norm"：

```
output = LayerNorm(x + Sublayer(x))
```

- **Add（残差连接）**：把输入直接加到输出上——`x + Sublayer(x)`

  - 解决深层网络梯度消失问题
  - 让网络至少不比浅层差（最坏情况Sublayer学成恒等映射，等于跳过）
- **Norm（层归一化）**：对每个样本的特征维度做归一化

  - 公式：LayerNorm(x) = γ × (x - μ) / σ + β
  - μ和σ是当前样本当前层的均值和标准差
  - γ和β是可学习的缩放和偏移参数
  - 稳定训练，加速收敛

⚠️ **考试区分**：

- **Batch Norm**：跨样本归一化（同一特征在不同样本间归一化）——CV常用
- **Layer Norm**：跨特征归一化（同一样本的不同特征间归一化）——NLP/Transformer用这个

为什么Transformer用Layer Norm不用Batch Norm？因为序列长度不一，batch内样本对齐困难，Layer Norm不受batch大小和序列长度影响。

---

### 零件五：前馈神经网络（Feed-Forward Network, FFN）

每个Transformer子层有两个子模块：注意力 + FFN。

```
FFN(x) = max(0, xW_1 + b_1)W_2 + b_2
```

- 两层线性变换，中间是ReLU激活
- 第一层把维度从d_model扩展到d_ff（原论文d_ff=2048，4倍扩展）
- 第二层压缩回d_model
- **每个位置独立计算，不共享权重**（但不同位置用相同的W_1和W_2）

直觉：注意力负责"收集信息"，FFN负责"加工信息"。注意力是信息路由，FFN是信息处理。

---

### 零件六：Decoder的特殊设计

Decoder和Encoder的三个关键区别：

| 区别 | Encoder | Decoder |
|-|-|-|
| **注意力类型** | 只有自注意力（双向） | 有掩码自注意力（单向）+ 交叉注意力 |
| **掩码（Mask）** | 不需要掩码 | 训练时遮住未来token（防止偷看答案） |
| **交叉注意力** | 无 | Q来自Decoder，K和V来自Encoder输出 |

**掩码自注意力（Masked Self-Attention）**：

- 把未来位置的注意力分数设为-∞，softmax后变成0
- 保证生成第t个token时，只能看到1到t-1的token
- 这就是为什么GPT是"从左到右依次生成"

**交叉注意力（Cross-Attention）**：

- Q来自Decoder（"我现在要生成什么"）
- K和V来自Encoder（"源语言有什么信息"）
- 这是翻译任务的关键：Decoder边生成边参考Encoder的理解

---

### 完整数据流（背下来！）

⚠️ **考试必考：Transformer一层的完整数据流**

**Encoder一层：**

```
输入 → Multi-Head Attention → Add & Norm → FFN → Add & Norm → 输出
```

**Decoder一层：**

```
输入 → Masked Multi-Head Attention → Add & Norm → Cross-Attention → Add & Norm → FFN → Add & Norm → 输出
```

原论文Encoder和Decoder各6层（N=6），堆叠而成。

---

## 二、关键对比

### Transformer vs RNN vs CNN

| 维度 | RNN | CNN | Transformer |
|-|-|-|-|
| **顺序性** | 必须顺序 | 可并行（局部） | 完全并行 |
| **长距离依赖** | 差（梯度消失） | 中（需要多层堆叠） | 好（直接注意力） |
| **计算复杂度** | O(n·d²) | O(k·n·d²) | O(n²·d) |
| **并行度** | 低 | 中 | 高 |
| **位置信息** | 天然有序 | 局部有序 | 需要显式编码 |

**注意**：Transformer的注意力复杂度是O(n²)——序列越长，计算量平方级增长。这是Transformer的阿喀琉斯之踵，也是后来各种高效注意力（稀疏注意力、线性注意力等）要解决的问题。

---

### Encoder vs Decoder vs Encoder-Decoder

| 架构 | 注意力方向 | 代表模型 | 擅长 |
|-|-|-|-|
| **Encoder-only** | 双向自注意力 | BERT | 理解：分类、NER、语义匹配 |
| **Decoder-only** | 单向掩码注意力 | GPT | 生成：续写、对话、代码 |
| **Encoder-Decoder** | 双向+单向+交叉 | T5、BART | 转换：翻译、摘要 |

---

## 三、易错陷阱

### ❌ 陷阱1：以为"注意力权重高=这个词更重要"

**错！** 注意力权重是**相对于某个query token**的——"对当前这个词来说，哪些其他词最相关"。权重高说明"跟当前token关系大"，不说明这个词本身更重要。

### ❌ 陷阱2：混淆"Masked LM"和"Masked Attention"

- **Masked LM（BERT）**：训练目标——随机遮住15%的token让模型预测
- **Masked Attention（Decoder）**：架构设计——遮住未来token防止偷看

两个完全不同的东西，考试会在选项里混着出。

### ❌ 陷阱3：以为多头注意力参数量是单头的h倍

**错！** 每个头的维度是d_model/h，总参数量不变。8个64维头 ≈ 1个512维头的计算量。多头不是"更重"，是"更宽"。

### ❌ 陷阱4：以为Transformer没有位置信息也能工作

**大错！** 没有位置编码，Transformer是词袋模型——"我爱你"和"你爱我"完全一样。位置编码是Transformer感知顺序的唯一方式。

### ❌ 陷阱5：搞混Layer Norm和Batch Norm

|  | Layer Norm | Batch Norm |
|-|-|-|
| 归一化维度 | 每个样本的特征维度 | 同一特征跨样本 |
| 适用场景 | NLP/Transformer | CV/CNN |
| 对batch大小依赖 | 不依赖 | 依赖 |

---

## 四、练习题

**Q1（核心概念）** Transformer相比RNN的核心优势是什么？

> **答案**：并行计算能力 + 直接建模长距离依赖。RNN必须顺序处理token，远距离信息靠隐状态传递会衰减；Transformer通过自注意力让任意两个token直接交互，距离不影响信息获取，且所有位置可同时计算。

---

**Q2（公式理解）** 注意力公式中为什么要除以√d_k？如果不除会怎样？

> **答案**：当d_k较大时，QK^T点积的方差会随维度增大而增大，导致softmax输入值过大，进入饱和区，梯度极小（接近0），模型几乎无法学习。除以√d_k使方差归一化，让softmax工作在有效区间。

---

**Q3（QKV理解）** 在自注意力中，Q、K、V分别从哪里来？

> **答案**：在自注意力中，Q、K、V都来自同一个输入序列——通过三个不同的线性变换矩阵W_Q、W_K、W_V分别投影得到。这就是"自"注意力的含义：自己关注自己。而在交叉注意力中，Q来自Decoder，K和V来自Encoder。

---

**Q4（架构辨析）** Decoder中的掩码自注意力和Encoder中的自注意力有什么区别？

> **答案**：Encoder的自注意力是双向的——每个token可以看到所有其他token。Decoder的掩码自注意力是单向的——每个token只能看到自己和之前的token，未来位置的注意力分数被设为-∞（softmax后为0），防止生成时"偷看"答案。

---

**Q5（位置编码）** 为什么Transformer需要位置编码而RNN不需要？

> **答案**：RNN按顺序处理token，天然携带位置信息。Transformer并行处理所有token，对所有位置一视同仁——如果没有位置编码，模型无法区分"我爱你"和"你爱我"。位置编码是Transformer获取序列顺序信息的唯一方式。

---

**Q6（多头注意力）** 多头注意力相比单头注意力有什么优势？参数量会增加吗？

> **答案**：多头注意力可以同时学习多种不同的注意力模式（如语法关系、语义关系、指代关系），单头只能学一种。参数量不增加——每个头的维度是d_model/h，h个头拼接后维度仍是d_model，总参数量与单头相同。

---

**Q7（层归一化）** 为什么Transformer用Layer Norm而不是Batch Norm？

> **答案**：①NLP任务中序列长度不一，batch内样本对齐困难，Batch Norm统计不稳定；②Batch Norm依赖batch大小，小batch时统计量噪声大；③Layer Norm对每个样本独立归一化，不受batch大小和序列长度影响，更适合变长序列。

---

**Q8（综合理解）** 请描述Encoder一层Transformer的完整数据流。

> **答案**：输入 → Multi-Head Self-Attention → Add & Norm（残差连接 + Layer Norm）→ Feed-Forward Network → Add & Norm → 输出。其中"Add"是残差连接（输入直接加到子层输出上），"Norm"是层归一化。

---

## 五、考试速查卡

| 概念 | 一句话 |
|-|-|
| 自注意力 | 每个token直接跟所有token交互，距离不是问题 |
| Q/K/V | Q问"我要什么"，K答"我有什么"，V给"我的内容" |
| 缩放因子√d_k | 防止点积过大导致softmax饱和、梯度消失 |
| 多头注意力 | 多种关注模式并行，参数量不变 |
| 位置编码 | 让并行处理的Transformer感知序列顺序 |
| Add & Norm | 残差连接保梯度 + 层归一化稳训练 |
| FFN | 注意力路由信息，FFN加工信息 |
| 掩码注意力 | Decoder专用，遮住未来token防止偷看 |
| 交叉注意力 | Q来自Decoder，K/V来自Encoder，桥接理解与生成 |

---

## 六、与D1的衔接

D0讲了Transformer的**零件**——注意力、位置编码、编码器/解码器的内部结构。

D1将讲Transformer的**产品形态**——同样的零件，不同的组装方式：

- 只用Encoder → BERT（双向理解）
- 只用Decoder → GPT（单向生成）
- 两个都用 → T5（理解+生成）

架构类型是Transformer的不同应用模式，理解了零件，D1就水到渠成了。

---

> 📖 参考：Vaswani, A., et al. (2017). "Attention is All You Need." [NVIDIA官方考试蓝图](https://www.nvidia.com/en-us/learn/certification/generative-ai-llm-associate/)将Transformer架构列为Domain 1（Deep Learning Fundamentals）的核心考点。
