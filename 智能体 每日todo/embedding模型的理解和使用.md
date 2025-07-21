Embedding是一个非常常见的任务，一般是指把文本、图像、视频、时间序列或者语音之类的数据转化为特定的向量表示并用于下游任务。Embedding有着丰富的应用场景，检索、召回、分类、聚类、图像或者文本相似度比较等任务。特别是随着RAG类任务的兴起，Embedding的意义在变得更加重要。

常见的Embedding模型应用的任务主要有四类：检索，分类，聚类，相似匹配。按查询和文档的长短分可以划分为短-长匹配、长-短匹配、短-短匹配和长-长匹配。按模态的不同就是图文，文本与文本，图片与图片等。

对比表示学习，目标是学习一个Embedding空间，其中相似的样本对彼此保持接近，而不同的样本对则相距较远。对比学习可以应用于监督和无监督训练。当处理无监督和弱监督数据时，对比学习是非常好用的方法之一。对比学习也可以认为是无监督或者弱监督下的度量学习。有文章认为对比学习是在学习一个Alignment（一致性）和 Uniformity（均匀性）都适宜的表示空间

常用的对比学习的loss。

1. Contrastive Loss 正例拉近，负例拉远
2. Triplet Loss 同时对正例和负例分别靠近和远离
3. InfoNCE 包括有交叉熵与无交叉熵版本
4. Consent loss 适合有一些有排序标签数据
![[Pasted image 20250708000030.png]]

## **训练经验与流程**

![](https://pic2.zhimg.com/v2-b04ab6a14c4590b880a4dc038e9d4f11_1440w.jpg)

Conan embedding的训练流程，也是常见的二阶段训练流程

目前常见的训练流程一般分为二阶段。

1. 一阶段（Contrastive Pre-Training）：大批量弱监督数据预训练。寻找大量弱相关性的pair，利用训练后的模型过滤掉最差的数据。
2. 二阶段：精细化多任务微调，加入更多负例数据，进行不同的任务共同训练。如使用CoSENT Loss，Contrast Loss，Info NCE loss等多种任务共同学习。

## 如何训练一个更优质的Embedding模型

### **数据工程**

**困难负样本挖掘 Hard Negative Mining**

困难负样本指的是应具有与锚样本不同的标签，但具有非常接近锚样本embedding的特征。

部分拥有标签的有监督数据集里可以很容易地获得特定任务的困难负样本。但是无监督或者弱监督训练的时候，**获取困难负样本**相对就会比较困难。增加训练Batch Size大小或Memoy Bank大小会隐式地引入更多困难负样本，但副作用是导致大量显存和内存占用的负担。

负例的难度在采样时非常重要。 选择一些困难的例子，但不要太难。

**BM25 Negatives/Approximate Nearest Neighbors**：通过BM25和向量混合召回拥有一定的相似度的负例，但是仍然语义上存在偏差的数据作为困难负例加入到Batch内。

**合成数据，利用LLM构造正例与困难负例**：传统的原生无监督情况下，一般是利用EDA，回译之类的手段，对原始的句子表示获取更多的正样本增强。后来会采用Dropout （SimCSE）和 Cut off tokens作为扰动的手段。或者对某些单词随机重复。

在ChatGPT后很多工作开始尝试利用LLM来合成正例和困难负例。大多数的工作认为人造数据可以很好的帮助提高性能，但是需要用正例doc和负例doc来作为条件来生成。

**动态困难负样本挖掘**：Conan提出了一个新的范式，对于具有给定权重集的embedding模型，预先处理的困难负样例是固定的。然而，随着训练进程和模型权重的更新，与当前权重相对应的困难负样本也会发生变化。在预处理阶段挖掘的困难负样本可能在几次训练迭代后变得不那么具有挑战性。

对于每个数据点，记录相对于query的困难负样本的当前平均得分。每100次step会重算一次负样本查询集合，判断负样本是否足够困难，并继续进行新一轮的==硬负挖掘==。

### 数据集混合和采样 **Dataset Mix And Sampling**

由于不同数据集的大小、一致性、困难程度和学习动态不同，简单地将所有可用数据集合并在一起被证明是一种次优策略，特别是在微调阶段。

**多类型数据对比：**根据来源领域与分布的不同，构造同源的数据集，每个mini batch都是基于domain粒度的采样，batch内随机负例来自相同的数据集比来自整个数据集更好。

### **模型架构**

当下大多数的Embedding模型还是以Bert为基础架构，或者类似NV-Embed以decode的模型为架构。

一般来说，文本的embedding模型。通常采用先取decode模型生成的最后一个<EOS> token， 或者encode结构最后一层hidden state，结合attention mask后mean pooling。再过一层线性层作为embedding表示。

**_Nomic Embed_** 在Bert架构的基础上

- Rotary位置嵌入替代绝对位置编码。
- 使用SwiGLU激活代替GeLU。
- 使用Flash Attention。
- 删除Dropout。
- 词表大小为64的倍数。

### **训练策略**

### **Matryoshka Representation Learning（俄罗斯套娃表征学习）**

根据指定维度[64,128,...,2048,3072]的向量来计算多个loss。使得用户在推理时，可以根据自己的实际需求，输入维度参数，来得到指定维度的向量。![[Pasted image 20250708000640.png]]

### **多任务混合损失训练**

同时训练多种embedding的下游任务数据，对于不同的任务采用不同的Loss

1. 检索排序：InfoNCE Loss / 批内负例
2. Semantic Textual Similarity and PairClassification：cosent loss  
    log⁡(1+∑sim⁡(i,j)>sim⁡(k,l)eλ(cos⁡(uk,ul)−cos⁡(ui,uj)))  
    对于有标签排序关系的样本对，如正负标签，正样本对的相似度大于负样本对的相似度
3. Classification and Clustering：三元组对比损失

### **其他**
**长文检索中的Embedding技巧**

**[迟分](https://link.zhihu.com/?target=https%3A//arxiv.org/pdf/2409.04701)：**Jina Embedding分享的长文embedding技巧。核心思想是先过 Embedding 模型再分块，先将 Embedding 模型的 transformer 层应用到整个文本或尽可能多的连续文本，为每个 token 生成一个包含丰富上下文信息的向量表示序列。随后，再对这些 token 向量序列进行平均池化，进而得到考虑了整个文本上下文的块 Embedding。![[Pasted image 20250708000727.png]]
**上下文检索：**anthropic发布的技巧，把每个文本块和整个文档一起输入到LLM中。LLM 给每个文本块加上相关的上下文信息。输出信息更丰富的向量。
![[Pasted image 20250708000736.png]]
### **评估**

- 分类任务，一段文本判断是否分类正确，指标：准确率（多分类）或精确率（二分类）
- 聚类，多段文本，指标：V-Measure
- 查询类，一段文本查多段文本

- 在有限集合中查询 Re-Ranking，指标：MAP（mean average precision）

- 在无限集合中查询

- Retrival，指标：NDCG@10 NDCG@k=DCG@kIDCG@kIDCG@k=Σi=1k2relideal,i−1log2⁡(i+1)DCG@k=∑i=1k2reli−1log2⁡(i+1) 真实序列收益值比上理论序列收益值

- 一段文本与另一段文本的关系（这类需要一群样本）

- 相似性，STS，指标：Spearman秩相关性系数
- 判别性，Pair-classification，指标：AP（average precision）