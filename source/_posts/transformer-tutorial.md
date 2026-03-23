---
title: 图解 Transformer：从零理解"注意力就是一切"
date: 2024-01-15
tags:
  - 深度学习
  - NLP
  - Transformer
  - 注意力机制
categories:
  - 论文
cover: /images/transformer-architecture.svg
---

> 2017 年，Google 的一篇论文《Attention Is All You Need》彻底改变了自然语言处理的格局。Transformer 架构不仅横扫了机器翻译任务，更成为 BERT、GPT、T5 等一系列大模型的基石。本文将用图文并茂的方式，带你从零理解 Transformer 的每一个核心组件。

<!-- more -->

---

## 一、为什么需要 Transformer？

在 Transformer 出现之前，序列建模的主流方案是 **RNN（循环神经网络）** 和 **LSTM**。它们的工作方式是逐步处理序列中的每个词，就像人类一个字一个字地阅读。

这种方式有两个致命缺陷：

**缺陷一：长距离依赖问题。** 当句子很长时，早期词的信息在传递过程中会逐渐"遗忘"。比如"The cat that ate the fish was full"，要理解"was"的主语是"cat"而不是"fish"，RNN 需要跨越很长的距离。

**缺陷二：无法并行计算。** RNN 必须按顺序处理，第 t 步依赖第 t-1 步的结果，导致训练速度极慢，无法充分利用现代 GPU 的并行能力。

Transformer 的核心思想是：**用注意力机制直接建模序列中任意两个位置之间的关系，完全抛弃循环结构**，从而同时解决了这两个问题。

---

## 二、整体架构一览

Transformer 采用经典的**编码器-解码器（Encoder-Decoder）**结构，以机器翻译为例：编码器负责"理解"源语言句子，解码器负责"生成"目标语言句子。

![Transformer 整体架构](/images/transformer-architecture.svg)

整个模型由以下几个核心部分组成：

- **输入嵌入（Input Embedding）**：将词转换为向量
- **位置编码（Positional Encoding）**：注入位置信息
- **编码器（Encoder）**：由 N 个相同的层堆叠而成（原论文 N=6）
- **解码器（Decoder）**：同样由 N 个层堆叠，但结构略有不同
- **输出层（Linear + Softmax）**：将解码器输出映射为词汇表上的概率分布

下面我们来看一个直观的翻译示例：

![翻译示例](/images/transformer-seq2seq.svg)

---

## 三、输入处理：词嵌入与位置编码

### 3.1 词嵌入（Word Embedding）

首先，每个词被映射为一个 $d_{model}$ 维的向量（原论文中 $d_{model} = 512$）。这一步与传统 NLP 模型相同。

```python
import torch
import torch.nn as nn

class TokenEmbedding(nn.Module):
    def __init__(self, vocab_size: int, d_model: int):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, d_model)
        self.d_model = d_model

    def forward(self, tokens):
        # tokens: (batch_size, seq_len)
        # 乘以 sqrt(d_model) 是为了让嵌入向量的尺度与位置编码匹配
        return self.embedding(tokens) * (self.d_model ** 0.5)
```

### 3.2 位置编码（Positional Encoding）

由于 Transformer 没有循环结构，它天然不知道词的顺序。位置编码就是为了解决这个问题——给每个位置注入一个独特的"坐标信号"。

原论文使用正弦/余弦函数生成位置编码：

$$PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$

$$PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$

其中 $pos$ 是位置索引，$i$ 是维度索引。

![位置编码可视化](/images/transformer-positional-encoding.svg)

这种设计的精妙之处在于：不同频率的正弦波可以让模型学习到相对位置关系，而且可以推广到训练时未见过的序列长度。

```python
import math
import torch
import torch.nn as nn

class PositionalEncoding(nn.Module):
    def __init__(self, d_model: int, max_len: int = 5000, dropout: float = 0.1):
        super().__init__()
        self.dropout = nn.Dropout(p=dropout)

        # 创建位置编码矩阵 (max_len, d_model)
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len).unsqueeze(1).float()  # (max_len, 1)
        
        # 计算分母：10000^(2i/d_model)
        div_term = torch.exp(
            torch.arange(0, d_model, 2).float() * (-math.log(10000.0) / d_model)
        )
        
        # 偶数维度用 sin，奇数维度用 cos
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        
        # 增加 batch 维度并注册为 buffer（不参与梯度更新）
        pe = pe.unsqueeze(0)  # (1, max_len, d_model)
        self.register_buffer('pe', pe)

    def forward(self, x):
        # x: (batch_size, seq_len, d_model)
        # 将位置编码加到词嵌入上
        x = x + self.pe[:, :x.size(1), :]
        return self.dropout(x)
```

---

## 四、核心机制：注意力（Attention）

注意力机制是 Transformer 的灵魂。它的核心思想是：**在处理每个词时，让模型动态地决定应该"关注"序列中的哪些其他词**。

### 4.1 缩放点积注意力（Scaled Dot-Product Attention）

注意力机制的输入是三个矩阵：**Q（Query，查询）、K（Key，键）、V（Value，值）**。

可以用一个图书馆检索的比喻来理解：
- **Q（查询）**：你想找的书的关键词
- **K（键）**：图书馆里每本书的标签
- **V（值）**：每本书的实际内容
- **注意力权重**：你的查询与每本书标签的匹配程度

计算公式为：

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

除以 $\sqrt{d_k}$ 是为了防止点积结果过大导致 softmax 梯度消失。

![缩放点积注意力](/images/transformer-attention.svg)

```python
import torch
import torch.nn.functional as F
import math

def scaled_dot_product_attention(
    query: torch.Tensor,   # (batch, heads, seq_q, d_k)
    key: torch.Tensor,     # (batch, heads, seq_k, d_k)
    value: torch.Tensor,   # (batch, heads, seq_k, d_v)
    mask: torch.Tensor = None  # 可选的掩码矩阵
):
    d_k = query.size(-1)
    
    # Step 1: 计算注意力分数 QK^T / sqrt(d_k)
    # scores shape: (batch, heads, seq_q, seq_k)
    scores = torch.matmul(query, key.transpose(-2, -1)) / math.sqrt(d_k)
    
    # Step 2: 应用掩码（解码器中用于屏蔽未来位置）
    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))
    
    # Step 3: Softmax 得到注意力权重
    attention_weights = F.softmax(scores, dim=-1)
    
    # Step 4: 加权求和 Value
    output = torch.matmul(attention_weights, value)
    
    return output, attention_weights
```

### 4.2 多头注意力（Multi-Head Attention）

单个注意力头只能关注一种类型的关系。多头注意力的思想是：**用多个注意力头并行地从不同角度理解序列**，就像用多个视角同时观察同一件事。

$$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, \ldots, \text{head}_h) W^O$$

$$\text{head}_i = \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)$$

原论文使用 $h=8$ 个头，每个头的维度 $d_k = d_v = d_{model}/h = 64$。

![多头注意力机制](/images/transformer-multihead.svg)

```python
import torch
import torch.nn as nn
import math

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model: int, num_heads: int, dropout: float = 0.1):
        super().__init__()
        assert d_model % num_heads == 0, "d_model 必须能被 num_heads 整除"
        
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads  # 每个头的维度
        
        # Q、K、V 的线性投影层
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        
        # 输出投影层
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)

    def split_heads(self, x: torch.Tensor) -> torch.Tensor:
        """将最后一维拆分为 (num_heads, d_k)，并转置"""
        batch_size, seq_len, d_model = x.size()
        # (batch, seq, d_model) -> (batch, seq, heads, d_k) -> (batch, heads, seq, d_k)
        return x.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)

    def forward(
        self,
        query: torch.Tensor,  # (batch, seq_q, d_model)
        key: torch.Tensor,    # (batch, seq_k, d_model)
        value: torch.Tensor,  # (batch, seq_k, d_model)
        mask: torch.Tensor = None
    ):
        batch_size = query.size(0)
        
        # Step 1: 线性投影
        Q = self.split_heads(self.W_q(query))  # (batch, heads, seq_q, d_k)
        K = self.split_heads(self.W_k(key))    # (batch, heads, seq_k, d_k)
        V = self.split_heads(self.W_v(value))  # (batch, heads, seq_k, d_k)
        
        # Step 2: 缩放点积注意力
        d_k = Q.size(-1)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attn_weights = torch.softmax(scores, dim=-1)
        attn_weights = self.dropout(attn_weights)
        
        # Step 3: 加权求和
        context = torch.matmul(attn_weights, V)  # (batch, heads, seq_q, d_k)
        
        # Step 4: 拼接所有头并输出投影
        # (batch, heads, seq_q, d_k) -> (batch, seq_q, d_model)
        context = context.transpose(1, 2).contiguous().view(
            batch_size, -1, self.d_model
        )
        output = self.W_o(context)
        
        return output, attn_weights
```

---

## 五、编码器层详解

每个编码器层由两个子层组成，每个子层都使用**残差连接（Residual Connection）**和**层归一化（Layer Normalization）**。

$$\text{LayerNorm}(x + \text{Sublayer}(x))$$

![编码器单层结构](/images/transformer-encoder-layer.svg)

### 5.1 前馈神经网络（Feed-Forward Network）

注意力层之后是一个简单的两层全连接网络，对每个位置独立地进行非线性变换：

$$\text{FFN}(x) = \max(0, xW_1 + b_1)W_2 + b_2$$

中间层维度为 $d_{ff} = 2048$（是 $d_{model}$ 的 4 倍）。

```python
class FeedForward(nn.Module):
    def __init__(self, d_model: int, d_ff: int, dropout: float = 0.1):
        super().__init__()
        self.linear1 = nn.Linear(d_model, d_ff)
        self.linear2 = nn.Linear(d_ff, d_model)
        self.dropout = nn.Dropout(dropout)
        self.relu = nn.ReLU()

    def forward(self, x):
        # x: (batch, seq, d_model)
        return self.linear2(self.dropout(self.relu(self.linear1(x))))
```

### 5.2 完整的编码器层

```python
class EncoderLayer(nn.Module):
    def __init__(self, d_model: int, num_heads: int, d_ff: int, dropout: float = 0.1):
        super().__init__()
        self.self_attention = MultiHeadAttention(d_model, num_heads, dropout)
        self.feed_forward = FeedForward(d_model, d_ff, dropout)
        
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)

    def forward(self, x: torch.Tensor, mask: torch.Tensor = None):
        # 子层 1：多头自注意力 + 残差连接 + 层归一化
        attn_output, _ = self.self_attention(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # 子层 2：前馈网络 + 残差连接 + 层归一化
        ff_output = self.feed_forward(x)
        x = self.norm2(x + self.dropout(ff_output))
        
        return x


class Encoder(nn.Module):
    def __init__(self, num_layers: int, d_model: int, num_heads: int, d_ff: int, dropout: float = 0.1):
        super().__init__()
        self.layers = nn.ModuleList([
            EncoderLayer(d_model, num_heads, d_ff, dropout)
            for _ in range(num_layers)
        ])
        self.norm = nn.LayerNorm(d_model)

    def forward(self, x: torch.Tensor, mask: torch.Tensor = None):
        for layer in self.layers:
            x = layer(x, mask)
        return self.norm(x)
```

---

## 六、解码器层详解

解码器比编码器多了一个**交叉注意力（Cross-Attention）**子层，用于关注编码器的输出。解码器的三个子层分别是：

**子层 1：掩码多头自注意力（Masked Multi-Head Self-Attention）**

解码器在生成第 $t$ 个词时，只能看到前 $t-1$ 个已生成的词，不能"偷看"未来。这通过一个**因果掩码（Causal Mask）**实现：

```python
def create_causal_mask(seq_len: int) -> torch.Tensor:
    """创建下三角掩码，屏蔽未来位置"""
    # 生成下三角矩阵（包含对角线）
    mask = torch.tril(torch.ones(seq_len, seq_len))
    return mask  # 1 表示可见，0 表示屏蔽
```

**子层 2：交叉注意力（Cross-Attention）**

Q 来自解码器自身，K 和 V 来自编码器的输出。这让解码器在生成每个词时，都能"查阅"整个源序列。

**子层 3：前馈神经网络**

与编码器相同。

```python
class DecoderLayer(nn.Module):
    def __init__(self, d_model: int, num_heads: int, d_ff: int, dropout: float = 0.1):
        super().__init__()
        # 子层 1：掩码自注意力
        self.self_attention = MultiHeadAttention(d_model, num_heads, dropout)
        # 子层 2：交叉注意力
        self.cross_attention = MultiHeadAttention(d_model, num_heads, dropout)
        # 子层 3：前馈网络
        self.feed_forward = FeedForward(d_model, d_ff, dropout)
        
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.norm3 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)

    def forward(
        self,
        x: torch.Tensor,              # 解码器输入
        encoder_output: torch.Tensor, # 编码器输出
        src_mask: torch.Tensor = None,  # 源序列掩码（padding）
        tgt_mask: torch.Tensor = None   # 目标序列掩码（因果掩码）
    ):
        # 子层 1：掩码自注意力
        attn1, _ = self.self_attention(x, x, x, tgt_mask)
        x = self.norm1(x + self.dropout(attn1))
        
        # 子层 2：交叉注意力（Q 来自解码器，K/V 来自编码器）
        attn2, _ = self.cross_attention(x, encoder_output, encoder_output, src_mask)
        x = self.norm2(x + self.dropout(attn2))
        
        # 子层 3：前馈网络
        ff = self.feed_forward(x)
        x = self.norm3(x + self.dropout(ff))
        
        return x
```

---

## 七、完整的 Transformer 模型

将所有组件组合在一起：

```python
class Transformer(nn.Module):
    def __init__(
        self,
        src_vocab_size: int,
        tgt_vocab_size: int,
        d_model: int = 512,
        num_heads: int = 8,
        num_encoder_layers: int = 6,
        num_decoder_layers: int = 6,
        d_ff: int = 2048,
        max_len: int = 5000,
        dropout: float = 0.1
    ):
        super().__init__()
        
        # 嵌入层
        self.src_embedding = TokenEmbedding(src_vocab_size, d_model)
        self.tgt_embedding = TokenEmbedding(tgt_vocab_size, d_model)
        self.positional_encoding = PositionalEncoding(d_model, max_len, dropout)
        
        # 编码器
        self.encoder = Encoder(num_encoder_layers, d_model, num_heads, d_ff, dropout)
        
        # 解码器
        self.decoder_layers = nn.ModuleList([
            DecoderLayer(d_model, num_heads, d_ff, dropout)
            for _ in range(num_decoder_layers)
        ])
        self.decoder_norm = nn.LayerNorm(d_model)
        
        # 输出层
        self.output_projection = nn.Linear(d_model, tgt_vocab_size)
        
        # 参数初始化
        self._init_parameters()

    def _init_parameters(self):
        """Xavier 均匀初始化"""
        for p in self.parameters():
            if p.dim() > 1:
                nn.init.xavier_uniform_(p)

    def encode(self, src: torch.Tensor, src_mask: torch.Tensor = None):
        src_emb = self.positional_encoding(self.src_embedding(src))
        return self.encoder(src_emb, src_mask)

    def decode(
        self,
        tgt: torch.Tensor,
        encoder_output: torch.Tensor,
        src_mask: torch.Tensor = None,
        tgt_mask: torch.Tensor = None
    ):
        tgt_emb = self.positional_encoding(self.tgt_embedding(tgt))
        x = tgt_emb
        for layer in self.decoder_layers:
            x = layer(x, encoder_output, src_mask, tgt_mask)
        return self.decoder_norm(x)

    def forward(
        self,
        src: torch.Tensor,  # (batch, src_len)
        tgt: torch.Tensor,  # (batch, tgt_len)
        src_mask: torch.Tensor = None,
        tgt_mask: torch.Tensor = None
    ):
        # 编码
        encoder_output = self.encode(src, src_mask)
        
        # 解码
        decoder_output = self.decode(tgt, encoder_output, src_mask, tgt_mask)
        
        # 输出投影到词汇表
        logits = self.output_projection(decoder_output)  # (batch, tgt_len, vocab_size)
        
        return logits


# ===== 快速测试 =====
if __name__ == "__main__":
    model = Transformer(
        src_vocab_size=10000,
        tgt_vocab_size=8000,
        d_model=512,
        num_heads=8,
        num_encoder_layers=6,
        num_decoder_layers=6,
        d_ff=2048
    )
    
    batch_size, src_len, tgt_len = 2, 10, 8
    src = torch.randint(0, 10000, (batch_size, src_len))
    tgt = torch.randint(0, 8000, (batch_size, tgt_len))
    
    # 创建因果掩码
    tgt_mask = create_causal_mask(tgt_len).unsqueeze(0).unsqueeze(0)
    
    logits = model(src, tgt, tgt_mask=tgt_mask)
    print(f"输出形状: {logits.shape}")  # (2, 8, 8000)
    
    # 统计参数量
    total_params = sum(p.numel() for p in model.parameters())
    print(f"模型参数量: {total_params:,}")  # 约 44M
```

---

## 八、训练技巧

### 8.1 学习率调度（Warmup）

原论文使用了一个特殊的学习率调度策略：先线性增大，再按步数的平方根衰减。

```python
class TransformerLRScheduler:
    """
    lrate = d_model^(-0.5) * min(step^(-0.5), step * warmup_steps^(-1.5))
    """
    def __init__(self, optimizer, d_model: int, warmup_steps: int = 4000):
        self.optimizer = optimizer
        self.d_model = d_model
        self.warmup_steps = warmup_steps
        self.current_step = 0

    def step(self):
        self.current_step += 1
        lr = self._get_lr()
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr

    def _get_lr(self):
        step = self.current_step
        return (self.d_model ** -0.5) * min(
            step ** -0.5,
            step * self.warmup_steps ** -1.5
        )
```

### 8.2 标签平滑（Label Smoothing）

原论文使用 $\epsilon_{ls} = 0.1$ 的标签平滑，防止模型过于自信：

```python
class LabelSmoothingLoss(nn.Module):
    def __init__(self, vocab_size: int, smoothing: float = 0.1, ignore_index: int = 0):
        super().__init__()
        self.smoothing = smoothing
        self.vocab_size = vocab_size
        self.ignore_index = ignore_index

    def forward(self, pred: torch.Tensor, target: torch.Tensor):
        # pred: (N, vocab_size), target: (N,)
        confidence = 1.0 - self.smoothing
        smooth_val = self.smoothing / (self.vocab_size - 2)
        
        # 创建平滑后的目标分布
        true_dist = torch.full_like(pred, smooth_val)
        true_dist.scatter_(1, target.unsqueeze(1), confidence)
        true_dist[:, self.ignore_index] = 0
        
        # 对 padding 位置不计算损失
        mask = (target != self.ignore_index).float()
        
        loss = (-true_dist * torch.log_softmax(pred, dim=-1)).sum(dim=-1)
        return (loss * mask).sum() / mask.sum()
```

---

## 九、关键超参数总结

原论文（base 模型）的超参数配置如下：

| 超参数 | 值 | 含义 |
|--------|-----|------|
| $d_{model}$ | 512 | 模型维度 |
| $h$ | 8 | 注意力头数 |
| $d_k = d_v$ | 64 | 每个头的维度 |
| $d_{ff}$ | 2048 | FFN 中间层维度 |
| $N$ | 6 | 编码器/解码器层数 |
| $P_{drop}$ | 0.1 | Dropout 比例 |
| $\epsilon_{ls}$ | 0.1 | 标签平滑系数 |
| warmup_steps | 4000 | 学习率预热步数 |

---

## 十、Transformer 的影响与延伸

Transformer 的提出开启了 NLP 的"预训练大模型"时代：

**BERT（2018）**：只使用编码器，通过掩码语言模型预训练，在众多 NLP 任务上刷新了 SOTA。

**GPT 系列（2018-至今）**：只使用解码器，通过自回归语言模型预训练，GPT-3/4 展现了惊人的涌现能力。

**T5（2019）**：将所有 NLP 任务统一为文本到文本的格式，使用完整的编码器-解码器结构。

**Vision Transformer（ViT，2020）**：将 Transformer 引入计算机视觉，把图像切分为 patch 序列处理。

**ChatGPT / GPT-4（2022-2023）**：基于 GPT 架构，通过 RLHF 对齐，成为现象级产品。

---

## 十一、总结

Transformer 的成功源于几个关键设计决策：

**注意力机制**让模型能够直接建模任意距离的依赖关系，彻底解决了 RNN 的长距离依赖问题。

**多头注意力**让模型从多个角度理解序列，捕获不同类型的语义关系。

**残差连接与层归一化**使得深层网络的训练更加稳定，可以堆叠更多层。

**完全并行化**的架构设计使得训练效率大幅提升，能够在大规模数据上进行预训练。

理解 Transformer 是进入现代 AI 领域的必经之路。希望这篇文章能帮助你建立起对这一里程碑架构的直觉理解。如果你想深入学习，强烈推荐阅读原论文 [Attention Is All You Need](https://arxiv.org/abs/1706.03762)，以及 Andrej Karpathy 的 [nanoGPT](https://github.com/karpathy/nanoGPT) 项目——一个极简但完整的 GPT 实现。

---

*参考资料：Vaswani et al., "Attention Is All You Need", NeurIPS 2017*
