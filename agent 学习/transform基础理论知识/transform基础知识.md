# 🔧 Transformer 理论知识汇总（自学笔记）

---

## 1. Transformer 模型核心原理与自注意力机制

Transformer 使用**自注意力机制（Self-Attention）**对序列中所有位置的 token 进行加权建模，无需递归或卷积结构，解决了长距离依赖难题。

- 通过自注意力，模型可在每一层学习序列中各位置之间的关系
- 可并行计算，适合大规模训练任务

---

## 2. 多头注意力机制的动机与优势

- **动机**：让模型从多个角度捕捉不同子空间的特征表示
- **优势**：
  - 每个注意力头关注不同的模式（如词法、语义、位置）
  - 提升表达能力，减少信息混叠
- **例子**：
  - 一个头学习主语与动词之间的关系
  - 一个头学习句子中时间、位置词之间的联系

---

## 3. 为什么 Q、K 使用不同权重生成？打破对称性提升表达力

若 Query 和 Key 使用相同权重，注意力分数可能产生退化（自我强化），不同权重矩阵可：

- 提供不同投影空间，增强表达多样性
- 打破对称性，提高模型学习非对称关系（如顺序偏好）

---

## 4. 点积 vs 加法注意力：为何选点积？

- **点积**：更高效（可并行向量运算），表达向量间相似度
- **加法注意力（Additive）**：用于RNN中，不适合并行
- **优势**：
  - 计算复杂度更低
  - 向量越相似（方向相近），点积越大，符合语义相似性直觉

---

## 5. 为什么需要Scaled Dot-Product Attention（缩放机制）

- **问题**：点积值过大导致 softmax 梯度过小（梯度消失）
- **缩放方式**：  
  \[
  \text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^\top}{\sqrt{d_k}}\right)V
  \]
- 缩放因子 \(\sqrt{d_k}\) 降低分数波动，防止梯度问题

---

## 6. 如何处理变长序列中的 Padding？Mask 的实现方法

- **问题**：Padding token 会干扰注意力计算
- **解决**：
  - 添加 Mask，使 Padding 位的注意力分数为 -∞
  - 使用 `attention_mask` 或 `key_padding_mask` 屏蔽无效位

---

## 7. Positional Encoding 的作用与数学意义

- **原因**：Transformer不具备处理顺序的能力
- **方法**：添加位置编码，使模型感知相对/绝对位置
- **正余弦位置编码**公式：
  \[
  PE_{(pos,2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right),\quad
  PE_{(pos,2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)
  \]

---

## 8. 残差连接（Residual Connection）作用与实现

- 作用：
  - 防止梯度消失
  - 加快收敛速度
  - 保留原始输入信息
- 实现方式：  
  \[
  \text{Output} = \text{LayerNorm}(x + \text{Sublayer}(x))
  \]

---

## 9. LayerNorm vs BatchNorm：为何选择 LayerNorm？

- LayerNorm 对每个样本独立归一化，适合变长序列和并行处理
- BatchNorm 对 batch 归一化，依赖统计量，不适合自注意力中的token级操作

---

## 10. 前馈神经网络 FNN 的设计与激活函数选择

- 通常结构：两层线性变换 + 激活函数  
  \[
  \text{FFN}(x) = \max(0, xW_1 + b_1)W_2 + b_2
  \]
- 激活函数选择：
  - ReLU：计算高效，但可能死神经
  - GELU：更平滑，BERT/GPT中默认使用

---

## 11. Encoder-Decoder 的交互机制

- Decoder 每一层通过 **Cross-Attention** 与 Encoder 输出交互
- Cross-Attention 使 Decoder 能感知 Encoder 的上下文表示，实现条件生成

---

## 12. 自注意力区别：Encoder vs Decoder

| 模块 | 注意力类型 | Mask 使用情况 |
|------|------------|----------------|
| Encoder | 全序列自注意力 | 无 Mask |
| Decoder | 自注意力（只看左边） | Causal Mask（防未来信息泄露） |
| Decoder-Encoder Attention | Cross-Attention | 无需 Mask |

---

## 13. Transformer 的并行化机制与解码端加速

- Encoder 结构天然并行
- Decoder 为自回归生成，难并行 → 可使用：
  - **KV Cache** 缓存前序状态
  - **Speculative Decoding / Parallel Decoding** 提前预测多步

---

## 14. WordPiece vs BPE 分词算法在 Transformer 中的应用

| 方法 | 特点 | 应用 |
|------|------|------|
| **WordPiece** | 通过词频概率合并 | BERT / T5 |
| **BPE（Byte Pair Encoding）** | 基于最频繁的字符对合并 | GPT / RoBERTa |

---

## 15. 为何 Embedding 乘以 \(\sqrt{d_{model}}\)

- 避免初始化 embedding 时方差过小，放大输入幅度
- 与 Attention 缩放保持一致性，提升稳定性

---

## 16. 学习率预热（Warmup）的策略与作用

- 初始阶段先缓慢升高学习率，再按计划衰减
- 避免训练初期不稳定、梯度爆炸
- 通常形式（Transformer 原始论文）：
  \[
  lr = d_{model}^{-0.5} \cdot \min(step^{-0.5}, step \cdot warmup^{-1.5})
  \]

---

## 17. Dropout 在 Transformer 中的作用与位置

- 防止过拟合，提高泛化能力
- 应用位置：
  - Attention 分数后
  - FFN 层之间
  - Embedding 后

> 测试阶段应关闭 Dropout，确保结果稳定

---

## 18. KV Cache 技术优化推理显存与效率

- 保存之前每一步生成的 Key/Value 张量
- 避免每次重复计算所有历史序列
- 显著减少推理时间和显存占用，尤其适用于长文本生成

---

## 19. 输入长度限制与解决方案

- 原始 Transformer 限制通常为 512 或 2048 token
- 解决方法：
  - Chunking（分块）
  - Sliding Window / 活动窗口
  - Sparse Attention（如 Longformer）
  - RoPE / ALiBi 等改进位置编码

---

## 20. FlashAttention 优化长文本处理性能

- 将 Attention 计算重写为高效内存访问的 CUDA 算子
- 优化显存访问，减少碎片和中间缓冲
- 提高长序列推理速度达数倍

---

## 21. PageAttention 与显存碎片问题

- PageAttention 将Attention计算按页分配显存
- 缓解显存碎片，提升大batch长文本处理能力
- 结合FlashAttention用于推理优化效果更佳

---

## 22. 动态量化 vs 静态量化

| 类型 | 特点 | 适用场景 |
|------|------|----------|
| **动态量化** | 推理时临时量化权重 | 小模型推理部署 |
| **静态量化** | 提前量化并校准范围 | 较大模型部署/硬件加速 |

---

## 23. 模型蒸馏原理与压缩方式

- **原理**：小模型模仿大模型行为（Soft Label学习）
- 压缩方式：
  - 结构精简
  - 层数减少
  - 注意力头精简
  - 参数共享

> 可在保留性能的同时降低模型规模与部署成本

---

## 24. 混合精度训练 AMP 的实现与加速作用

- AMP（Automatic Mixed Precision）通过 float16 + float32 组合训练
- 提升显存利用效率，缩短训练时间
- 框架支持：PyTorch AMP、NVIDIA Apex

---

## 25. 使用 DeepSpeed 的 ZeRO 优化器加速分布式训练

- **ZeRO**（Zero Redundancy Optimizer）将参数、梯度、优化器状态切分存储
- 优化方式：
  - **ZeRO-1**：优化器状态切分
  - **ZeRO-2**：再切分梯度
  - **ZeRO-3**：连参数也切分，最彻底压缩内存
- 支持千亿参数训练与推理

---
