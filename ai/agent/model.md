# M — Model

> **Model 决定能力上限。**

Model 关注的是：

```text
模型本身是什么
模型如何处理输入
模型如何训练
模型如何生成输出
模型能力从哪里来
```

核心主线：

```text
神经网络基础
   ↓
RNN / LSTM
   ↓
Transformer
   ↓
Attention
   ↓
现代 LLM 架构
   ↓
预训练 / 后训练
   ↓
推理与生成
```

---

# 大语言模型基础

## 模型能力从哪里来

大语言模型能力主要由以下因素共同决定：

```text
模型结构
训练数据
参数规模
训练计算量
训练目标
后训练方法
```

可以简单理解：

```text
模型结构
→ 决定信息如何流动

数据
→ 决定模型学到什么

参数量
→ 决定模型容量

计算量
→ 决定训练是否充分

后训练
→ 决定模型是否会遵循指令、偏好和任务要求
```

---

## Scaling Laws

Scaling Laws 描述模型性能与：

```text
参数量 N
训练数据量 D
训练计算量 C
```

之间存在较稳定的经验关系。

可以粗略理解为：

```text
N ↑
D ↑
C ↑
   ↓
Loss 通常下降
模型性能通常提升
```

核心意义：

```text
性能具有一定可预测性
资源之间存在瓶颈效应
模型、数据、计算需要合理配比
```

---

## 能力涌现

随着模型规模和训练水平提升，模型可能表现出更复杂的能力，例如：

```text
指令遵循
上下文学习
多步推理
代码生成
复杂任务泛化
```

核心理解：

> 模型能力并不一定随着规模简单线性增长。

---

## 模型幻觉

模型幻觉可以理解为：

```text
模型生成了看起来合理
但实际上不正确或不忠实的信息
```

常见类型：

### 事实性幻觉

```text
生成内容
≠
现实世界事实
```

### 忠实性幻觉

常见于：

```text
摘要
翻译
改写
```

生成内容没有忠实反映源文本。

### 内在幻觉

```text
生成内容
与输入信息直接矛盾
```

常见缓解方法：

```text
高质量数据
RAG
外部工具
结果验证
多步检查
```

---

# 神经网络基础

## 梯度消失与梯度爆炸

深层神经网络反向传播时：

```text
梯度不断连乘
```

可能出现：

```text
梯度越来越小
→ 梯度消失

梯度越来越大
→ 梯度爆炸
```

常见解决方法：

```text
合理激活函数
合理权重初始化
归一化
残差连接
梯度裁剪
LSTM / GRU
合适的优化器
```

---

## 激活函数

### Sigmoid

\[
\sigma(x)=\frac{1}{1+e^{-x}}
\]

特点：

```text
输出范围：(0,1)
```

常见用途：

```text
概率输出
门控结构
```

缺点：

```text
容易饱和
容易造成梯度消失
输出不是零中心
```

---

### Tanh

\[
\tanh(x)
\]

特点：

```text
输出范围：(-1,1)
零中心
```

常见于：

```text
RNN
LSTM
```

但仍可能发生梯度消失。

---

### ReLU

\[
f(x)=\max(0,x)
\]

特点：

```text
x > 0 → x
x <= 0 → 0
```

优点：

```text
计算简单
缓解梯度消失
```

缺点：

```text
神经元死亡
```

---

### Leaky ReLU

负半轴保留一个较小斜率：

\[
f(x)=
\begin{cases}
x,&x>0\\
\alpha x,&x\le0
\end{cases}
\]

用于缓解：

```text
ReLU 神经元死亡
```

---

### GELU

\[
GELU(x)=x\Phi(x)
\]

特点：

```text
平滑
梯度稳定
Transformer 中常见
```

---

### SwiGLU

核心：

```text
门控
+
非线性激活
```

可以理解为：

```text
一部分特征提供内容
另一部分特征作为门
```

常用于现代大语言模型的 FFN。

---

## 正则化

### L1

惩罚项：

```text
权重绝对值之和
```

特点：

```text
容易产生稀疏权重
```

---

### L2

惩罚项：

```text
权重平方和
```

特点：

```text
让权重趋向较小
提高泛化能力
```

---

## 残差连接

标准残差连接：

\[
y=x+F(x)
\]

结构：

```text
x ──────────────┐
                ↓
                +
                ↑
x → F(x) ───────┘
```

作用：

```text
给信息提供直接传播路径
给梯度提供直接传播路径
缓解深层网络训练困难
```

注意：

```text
标准 RNN
→ 没有残差连接

标准 LSTM
→ 也不是 ResNet 式残差连接
```

LSTM 的 Cell State 有：

```text
门控
+
加法更新
```

因此具有类似“高速通道”的效果，但不等同于标准残差连接。

---

## 归一化

常见：

```text
BatchNorm
LayerNorm
```

Transformer 更常使用：

```text
LayerNorm
```

目的：

```text
稳定中间表示
改善训练稳定性
```

---

# Transformer

## RNN → LSTM → Transformer

这一演进可以记成：

```text
RNN
→ 解决序列怎么记

LSTM
→ 解决 RNN 记不住太久

Transformer
→ 解决为什么必须一步一步记
```

---

## RNN

RNN：

```text
当前输入 x_t
+
上一时刻隐藏状态 h_(t-1)
      ↓
得到 h_t
```

公式可以粗略写为：

\[
h_t=f(x_t,h_{t-1})
\]

结构：

```text
x1 → RNN → h1
           ↓
x2 → RNN → h2
           ↓
x3 → RNN → h3
```

特点：

```text
前面的信息
通过 hidden state
逐步向后传递
```

问题：

```text
长距离依赖弱
容易梯度消失 / 爆炸
必须串行计算
```

---

## LSTM

LSTM：

```text
Long Short-Term Memory
长短期记忆网络
```

本质上仍属于：

```text
RNN
```

但增加：

```text
Cell State
+
输入门
遗忘门
输出门
```

### 三个门

输入门：

```text
新信息写入多少
```

遗忘门：

```text
旧信息保留多少
```

输出门：

```text
当前输出多少
```

一句话：

> **遗忘门决定“忘多少”，输入门决定“记多少”，输出门决定“说多少”。**

---

### Cell State

核心更新：

\[
c_t=f_t\odot c_{t-1}+i_t\odot\tilde c_t
\]

隐藏状态：

\[
h_t=o_t\odot\tanh(c_t)
\]

可以理解：

```text
旧 Cell State c(t-1)
        │
        × 遗忘门
        │
        ├───────────┐
                    +
                    ↑
              新信息 × 输入门
                    │
                    ↓
               新 Cell State c(t)
                    │
                  tanh
                    │
                 × 输出门
                    │
                    ↓
                   h(t)
```

LSTM 改善：

```text
长期依赖
梯度传播
```

但仍然存在：

```text
按时间步串行计算
并行能力差
```

---

## Transformer 为什么出现

RNN / LSTM 都有：

```text
h_t
依赖
h_(t-1)
```

因此：

```text
第 t 步
必须等第 t-1 步算完
```

结果：

```text
无法充分并行
长序列训练慢
模型规模受限制
```

Transformer：

```text
去掉循环结构
主要依赖 Attention
```

从：

```text
逐步传递信息
```

变成：

```text
任意 token
直接关注其他 token
```

---

## Encoder / Decoder

原始 Transformer 是：

```text
Encoder
+
Decoder
```

### Encoder

主要作用：

```text
理解输入
```

每层主要包含：

```text
Self-Attention
+
FFN
```

---

### Decoder

主要作用：

```text
生成输出
```

每层主要包含：

```text
Masked Self-Attention
+
Cross-Attention
+
FFN
```

Cross-Attention：

```text
Q
→ Decoder 表示

K / V
→ Encoder 输出
```

---

## Self-Attention

每个 token 会产生：

```text
Q — Query
K — Key
V — Value
```

由输入矩阵：

\[
X
\]

经过三个可学习矩阵得到：

\[
Q=XW_Q
\]

\[
K=XW_K
\]

\[
V=XW_V
\]

可以理解：

```text
Q
→ 我想找什么

K
→ 我有什么特征

V
→ 我真正携带什么信息
```

---

## Scaled Dot-Product Attention

核心公式：

\[
\operatorname{Attention}(Q,K,V)
=
\operatorname{softmax}
\left(
\frac{QK^T}{\sqrt{d_k}}
\right)V
\]

如果计算词 A 的新表示：

### 1. 生成 Q、K、V

```text
所有 token
   ↓
W_Q / W_K / W_V
   ↓
Q / K / V
```

### 2. 计算相关性

用：

```text
词 A 的 q_A
```

和所有 Key 点积：

\[
q_AK^T
\]

得到：

```text
A 对每个 token 的相关性分数
```

### 3. 缩放

\[
\frac{q_AK^T}{\sqrt{d_k}}
\]

其中：

```text
d_k
→ Key 向量维度
```

作用：

```text
防止维度较大时点积绝对值过大
避免 Softmax 过度饱和
```

### 4. Softmax

\[
\alpha=
\operatorname{softmax}
\left(
\frac{q_AK^T}{\sqrt{d_k}}
\right)
\]

得到：

```text
注意力权重
```

例如：

```text
A → 0.1
B → 0.6
C → 0.2
D → 0.1
```

### 5. 对 V 加权求和

\[
z_A=\sum_j\alpha_{Aj}v_j
\]

最终：

```text
z_A
→ 词 A 的新表示
```

记忆：

> **Q 和 K 决定“关注谁”，Softmax 决定“关注多少”，V 决定“拿什么信息回来”。**

---

## Multi-Head Attention

单头 Attention：

```text
只在一个表示空间里计算关系
```

多头 Attention：

```text
多个 Head
分别学习不同关系
```

流程：

```text
Q / K / V
   ↓
多个 Head
   ↓
各自 Attention
   ↓
Concat
   ↓
线性映射
   ↓
输出
```

经典表达：

\[
MultiHead(Q,K,V)
=
Concat(head_1,\ldots,head_h)W^O
\]

---

## FFN

Transformer 每层还包含：

```text
Position-wise Feed Forward Network
```

经典结构：

\[
FFN(x)=\max(0,xW_1+b_1)W_2+b_2
\]

过程：

```text
d_model
   ↓
扩维 d_ff
   ↓
激活函数
   ↓
压回 d_model
```

Attention：

```text
聚合不同 token 的信息
```

FFN：

```text
进一步加工每个 token 的特征
```

特点：

```text
每个位置独立计算
所有位置共享同一组 FFN 参数
```

---

## Residual + LayerNorm

Transformer 子层通常配合：

```text
Residual Connection
+
LayerNorm
```

残差：

\[
y=x+Sublayer(x)
\]

作用：

```text
改善梯度传播
提高深层网络稳定性
```

LayerNorm：

```text
对单个 token 的特征维度归一化
```

作用：

```text
稳定训练
```

---

## Position Encoding

Self-Attention 本身：

```text
不天然包含顺序信息
```

所以必须加入位置信息。

常见方式：

```text
正弦 / 余弦位置编码
可学习绝对位置编码
RoPE
```

---

### 正弦 / 余弦位置编码

特点：

```text
无可训练参数
通过固定函数编码位置
```

---

### 可学习绝对位置编码

特点：

```text
位置向量由模型学习
```

限制：

```text
通常受训练时最大序列长度约束
```

---

### RoPE

RoPE：

```text
Rotary Positional Embedding
```

核心：

```text
对 Q / K 做位置相关旋转
```

从而：

```text
Attention 分数
自然带有相对位置信息
```

流程：

```text
位置
 ↓
构造旋转
 ↓
旋转 Q / K
 ↓
计算 Attention
```

---

# Transformer 演进

## Encoder-Only

代表：

```text
BERT
RoBERTa
```

特点：

```text
双向 Self-Attention
```

擅长：

```text
文本分类
情感分析
NER
NLI
表征学习
```

核心：

```text
理解
```

---

## Decoder-Only

代表：

```text
GPT
Llama
Qwen
```

特点：

```text
Causal Self-Attention
```

生成第 t 个 token 时：

```text
只能看到当前位置之前的信息
```

擅长：

```text
文本生成
对话
代码生成
上下文学习
```

核心：

```text
生成
```

---

## Encoder-Decoder

代表：

```text
T5
BART
原始 Transformer
```

结构：

```text
Encoder
   ↓
理解输入

Decoder
   ↓
Cross-Attention
   ↓
条件生成
```

适合：

```text
翻译
摘要
问答
Seq2Seq
```

---

## MHA、MQA、GQA

### MHA

```text
N 个 Q Head
N 个 K Head
N 个 V Head
```

特点：

```text
表达能力强
KV Cache 大
```

---

### MQA

```text
N 个 Q Head
1 个 K Head
1 个 V Head
```

特点：

```text
KV Cache 小
推理效率高
表达能力可能有所下降
```

---

### GQA

```text
N 个 Q Head
G 个 K Head
G 个 V Head
```

作用：

```text
在 MHA 和 MQA 之间折中
```

对比：

| 对比 | MHA | MQA | GQA |
| --- | --- | --- | --- |
| Q Head | N | N | N |
| K/V Head | N | 1 | G |
| KV Cache | 大 | 最小 | 中等 |
| 表达能力 | 强 | 相对弱 | 接近 MHA |
| 推理效率 | 较低 | 高 | 较高 |

---

## MoE

MoE：

```text
Mixture of Experts
混合专家模型
```

核心：

```text
大量 Expert
+
每个 token 只激活少数 Expert
```

流程：

```text
Token
 ↓
Router
 ↓
给专家打分
 ↓
Top-K Expert
 ↓
专家分别计算
 ↓
加权汇总
```

Expert 本质通常是：

```text
FFN
```

优势：

```text
总参数量很大
但单次只使用部分参数
```

因此可以一定程度上实现：

```text
模型容量
与
单次计算量
解耦
```

---

# Tokenization

## Token

模型不能直接处理文本字符串。

流程：

```text
原始文本
   ↓
Tokenizer
   ↓
Token
   ↓
Token ID
   ↓
Embedding
```

---

## Subword

子词切分：

```text
词
↕
字符
```

之间的折中。

优点：

```text
缓解 OOV
控制词表大小
控制序列长度
保留部分词形信息
```

---

## BPE 与 WordPiece

| 对比 | BPE | WordPiece |
| --- | --- | --- |
| 核心 | 高频相邻单元合并 | 基于语言建模目标选择子词 |
| 特点 | 简单、高效 | 更偏概率建模 |
| 常见代表 | GPT、Llama、RoBERTa | BERT 等 |

---

# 推理与生成

## Greedy Search

每一步：

```text
直接选择概率最高 token
```

优点：

```text
快
稳定
```

缺点：

```text
多样性低
容易局部最优
```

---

## Beam Search

每一步：

```text
保留多个高概率候选序列
```

再继续扩展。

特点：

```text
搜索更充分
计算开销更高
```

---

## Top-K Sampling

流程：

```text
选概率最高 K 个 token
   ↓
重新归一化
   ↓
随机采样
```

特点：

```text
增加随机性
过滤低概率 token
```

---

## Top-P Sampling

流程：

```text
按概率排序
   ↓
选择累计概率达到 p 的最小候选集
   ↓
重新归一化
   ↓
随机采样
```

特点：

```text
候选集合大小动态变化
```

---

## KV Cache

自回归生成时：

```text
前面 token 的 K / V
已经计算过
```

所以可以缓存：

```text
K
V
```

后续生成时直接复用。

作用：

```text
减少重复计算
提高推理速度
```

但代价：

```text
占用显存
```

这也是：

```text
MHA → MQA → GQA
```

不断优化的重要原因之一。

---

# LLM 训练

## Pretraining

预训练目标：

```text
学习语言规律
学习通用知识
获得基础生成能力
```

典型训练方式：

```text
自监督学习
Next Token Prediction
```

特点：

```text
数据量大
计算成本高
训练时间长
```

---

## Post-training

后训练目标：

```text
学会遵循指令
学会对话格式
提升回答质量
对齐偏好
```

常见流程：

```text
SFT
 ↓
Preference / Reward Modeling
 ↓
RL / Preference Optimization
```

---

## SFT

SFT：

```text
Supervised Fine-Tuning
监督微调
```

训练数据通常：

```text
Prompt
+
Completion
```

目标：

```text
学习指令遵循
学习任务输出格式
```

---

## Reward Model

奖励模型使用偏好数据：

```text
同一个 Prompt
├── chosen
└── rejected
```

目标：

```text
学习哪个回答更符合偏好
```

---

## 强化学习微调

经典方式之一：

```text
PPO
```

核心：

```text
模型生成回答
 ↓
获得 Reward
 ↓
根据 Reward 优化策略
```

---

## 大模型训练挑战

### 显存

常见技术：

```text
数据并行
张量并行
流水线并行
ZeRO
```

---

### 通信

大规模分布式训练需要关注：

```text
设备间通信
通信带宽
通信开销
```

---

### 训练稳定性

常见方法：

```text
合适的数值精度
稳定模型结构
梯度裁剪
学习率调度
Warmup
```

---

# 速记

```text
Model
│
├── 神经网络基础
│   ├── 激活函数
│   ├── 梯度
│   ├── 正则化
│   └── 残差连接
│
├── Transformer
│   ├── RNN → LSTM → Transformer
│   ├── Attention
│   ├── MHA
│   ├── FFN
│   ├── Residual + LayerNorm
│   └── Position Encoding
│
├── Transformer 演进
│   ├── Encoder-Only
│   ├── Decoder-Only
│   ├── Encoder-Decoder
│   ├── MHA / MQA / GQA
│   └── MoE
│
├── Tokenization
│   ├── Token
│   ├── Subword
│   ├── BPE
│   └── WordPiece
│
├── 推理与生成
│   ├── Greedy
│   ├── Beam Search
│   ├── Top-K
│   ├── Top-P
│   └── KV Cache
│
└── LLM 训练
    ├── Pretraining
    ├── Post-training
    ├── SFT
    ├── Reward Model
    └── RL
```

核心关系：

```text
RNN
→ 顺序传递 hidden state

LSTM
→ 加 Cell State + 门控
→ 更好处理长期依赖

Transformer
→ 去掉循环
→ Self-Attention 直接建立 token 间关系
→ 支持大规模并行训练
```

Attention 核心公式：

\[
\boxed{
\operatorname{Attention}(Q,K,V)
=
\operatorname{softmax}
\left(
\frac{QK^T}{\sqrt{d_k}}
\right)V
}
\]

一句话：

> **Model 这一层，重点就是理解模型如何表示信息、如何建模关系、如何训练，以及如何从概率分布中生成输出。**
