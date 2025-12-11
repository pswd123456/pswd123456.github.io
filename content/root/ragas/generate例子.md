---
date created: 12月11日 , 23:20 , 2025
date modified: 12月11日 , 23:26 , 2025
---

这个问题的核心在于理解 Ragas 是如何把“死”的文档变成“活”的测试题的。

为了让你直观理解，我们用一个**“老师出考卷”**的例子来比喻整个过程。

## 1. 核心概念比喻

- **Documents (文档)**: 教科书（原始素材）。
    
- **Transforms (转换)**: 老师备课做的**笔记和思维导图**。老师不能只盯着书看，得先把书里的重点（关键词）、章节之间的联系（相似度）整理出来。
    
- **Synthesizer (合成器)**: **出题老师**。有负责出“单选题”的老师，也有负责出“综合大题”的老师。
    
- **Scenario (场景)**: **出题大纲/蓝图**。比如：“这道题要考第一章和第二章的对比，难度要高，模拟‘好奇的学生’的语气”。（注意：此时还没写出具体的题目字句，只是确定了考点）。
    
- **Sample (样本)**: **最终的考题和答案**。具体的“题目：请分析 A 和 B 的区别？”以及对应的“标准答案”。

---

## 2. 具体例子演示 (Step-by-Step)

假设你上传了两段文本作为 `Documents`：

- **文档 A**: “iPhone 15 使用了 A16 芯片，续航时间为 20 小时。”
    
- **文档 B**: “iPhone 15 Pro 使用了 A17 Pro 芯片，支持光线追踪。”
    

### **第一步：Transforms (转换) —— 整理笔记**

Transforms 的工作是丰富知识图谱。

原始图谱里只有两个孤立的点：[Node A] 和 [Node B]。

Transforms 干了什么？

1. **提取摘要 (SummaryExtractor)**: 发现 A 讲标准版，B 讲 Pro 版。
    
2. **提取实体 (NERExtractor)**: 提取出 "A16", "A17 Pro", "iPhone 15"。
    
3. **建立关系 (RelationshipBuilder)**: 发现 A 和 B 都在讲 "iPhone 15" 系列的芯片，于是建立一条**关联线**（Similarity/Overlap）。
    
    - _结果_：现在的图谱里，Node A 和 Node B 连起来了。
        

### **第二步：Synthesizer (合成器) —— 老师介入**

假设我们用的是 MultiHopSpecificQuerySynthesizer（多跳查询合成器，专门出综合题）。

Synthesizer 干了什么？

1. 它去扫描图谱，发现 Node A 和 Node B 有关联。
    
2. 它决定：“这里可以出一个跨文档的对比题”。
    

### **第三步：Scenario (场景) —— 制定大纲**

合成器生成了一个 Scenario 对象。这还不是具体的文字，而是一个配置单。

Scenario 长什么样？ (参考 BaseScenario 源码)

```Python
Scenario(
    nodes=[Node_A, Node_B],        # 考点涉及这两个节点
    style="Web Search Like",       # 风格：像搜索引擎的简短提问
    length="Medium",               # 长度：中等
    persona="Tech Reviewer"        # 角色：科技博主
)
```

### **第四步：Sample (样本) —— 生成题目**

合成器把上面的 Scenario 扔给 LLM，说：“请你作为一个科技博主，根据Node A 和 Node B 的内容，写一个中等长度的问题。”

Sample (最终结果):

- **User Input (Question)**: "对比 iPhone 15 和 iPhone 15 Pro 在芯片性能上的主要区别是什么？"
    
- **Reference (Answer)**: "iPhone 15 使用 A16 芯片，而 Pro 版本使用 A17 Pro 芯片并支持光追。"

---

## 3. 源码层面的深度解释

### **Transforms (转换)**

它是数据预处理的流水线。

在 ragas/testset/transforms/default.py 中，你可以看到它包含了一系列步骤：

- `HeadlineSplitter`: 把文章按标题切开。
    
- `EmbeddingExtractor`: 把文本转成向量。
    
- `CosineSimilarityBuilder`: 这一步最关键。它计算不同节点摘要的相似度，如果相似度高（比如 >0.7），就在两个节点间加一条边。**没有这一步，Synthesizer 就找不到跨文档的关联，也就生成不了复杂的推理题**。
    

### **Synthesizer (合成器)**

它是生成逻辑的执行者。

不同的 Synthesizer 负责生成不同类型的题目。

- `SingleHopSpecificQuerySynthesizer`: 专门找单个节点里的实体（entities）来提问。源码中 `_generate_scenarios` 方法通过 `get_node_clusters` 找到包含特定属性的单个节点。
    
- `MultiHopSpecificQuerySynthesizer`: 专门找**三元组（Triplets）**或**重叠（Overlap）**关系。源码中它使用 `find_two_nodes_single_rel` 寻找两个连接的节点，从而生成需要结合两处信息才能回答的问题。
    

### **Scenario (场景) vs Sample (样本)**

这两个类在源码中是严格区分的。

- **Scenario (`BaseScenario`)**: 是**中间产物**。它是一个 Python 对象（Pydantic Model），包含了生成问题所需的所有元数据（节点、风格、角色）。生成 Scenario 的过程通常不涉及大规模的文本生成，主要是图谱搜索和逻辑组合。
    
- **Sample (`BaseSample`)**: 是**最终产物**。它是调用 LLM 对 `Scenario` 进行“渲染”后的结果，包含具体的字符串文本。生成 Sample 的过程就是 LLM 根据 Scenario 的指令写作文的过程。
    

## 总结

- **Transforms**: 造桥修路（构建知识图谱中的连接）。
    
- **Synthesizer**: 调度员（决定走哪条路，用什么车）。
    
- **Scenario**: 行程单（计划书：从A到B，用跑车，走高速）。
    
- **Sample**: 实际的旅行记录（具体的问答文本）。