---
date created: 12月11日 , 23:25 , 2025
date modified: 12月11日 , 23:26 , 2025
---

Ragas 目前主要包含 **3 种** 核心的 Synthesizer（合成器）。它们分别针对不同的查询场景（单跳/多跳）和内容粒度（具体实体/抽象主题）进行设计。

以下是具体的分类及其功能解释：

## 1. SingleHopSpecificQuerySynthesizer (单跳具体查询合成器)

- **用途**: 生成基于**单个文档节点**中**特定术语或实体**的简单查询。
    
- **工作原理**: 它会在知识图谱中寻找包含特定实体（Entities）的单个节点（Chunk 或 Document），结合生成的 Persona（角色）对该实体进行提问。
    
- **典型问题**: "iPhone 15 的电池容量是多少？"（直接针对某个具体参数提问）。
    

## 2. MultiHopSpecificQuerySynthesizer (多跳具体查询合成器)

- **用途**: 生成需要**跨越两个或多个节点**，且基于**共同实体或重叠信息**的复杂查询。
    
- **工作原理**: 它会寻找知识图谱中具有重叠实体（Entities Overlap）的节点对。例如，如果节点 A 提到了 "Elon Musk"，节点 B 也提到了 "Elon Musk"，该合成器就会生成一个结合两处信息的问题。
    
- **典型问题**: "Elon Musk 在创立 SpaceX 之前，在 PayPal 担任什么具体职务？"（需要结合两段不同时期的经历）。
    

## 3. MultiHopAbstractQuerySynthesizer (多跳抽象查询合成器)

- **用途**: 生成基于**高层主题或摘要**的跨文档查询，通常涉及比较、分析或总结。
    
- **工作原理**: 它利用节点间的**摘要相似度**（Summary Similarity）来寻找关联。它不依赖完全相同的关键词，而是依赖语义上的相关性（Themes），生成更宏观的问题。
    
- **典型问题**: "比较这两篇文章中对于气候变化应对策略的主要异同点。"（不针对具体实体，而是针对抽象的策略主题）。
    

## 总结

在 `default_query_distribution` 函数中可以看到，Ragas 默认会同时使用这三种合成器来保证测试集的多样性：

```Python
default_queries = [
    SingleHopSpecificQuerySynthesizer(llm=llm),
    MultiHopAbstractQuerySynthesizer(llm=llm),
    MultiHopSpecificQuerySynthesizer(llm=llm),
]
```