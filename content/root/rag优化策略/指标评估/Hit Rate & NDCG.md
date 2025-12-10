---
date created: 11月25日 , 23:38 , 2025
date modified: 12月10日 , 16:5 , 2025
---

## Hit Rate & NDCG

### 1. 核心概念简述

- **Hit Rate (Recall@K)**: 简单的二元判断。
    
    - _含义_：在前 `K` 个检索结果中，是否包含正确的那个文档块？
        
    - _公式_：命中次数 / 总查询次数。
        
- **NDCG@K**: 考虑排名的权重。
    
    - _含义_：正确的文档块排得越靠前，分数越高。如果正确文档在第 1 位，得分 1.0；在第 10 位，得分就很低。
        
    - _价值_：对于 Rerank 模型，NDCG 至关重要，因为 Rerank 的目的就是把正确答案推到最前面。

---

### 2. 准备工作：构建“带 ID 的”测试集

数据结构要求：

如下结构的 JSON 或 DataFrame：

JSON

```json
[
  {
    "query": "招标文件的保证金是多少？",
    "expected_doc_id": "chunk_uuid_12345",  // 关键：必须指向数据库中真实的 ID
    "expected_parent_id": "parent_uuid_abcde" // 如果是用 Parent-Child 策略
  },
  {
    "query": "服务器的CPU规格要求？",
    "expected_doc_id": "chunk_uuid_67890",
    "expected_parent_id": "parent_uuid_fghij"
  }
]
```

**如何获取这个 ID？**

- **方法 A（人工标注）**：在人工构建测试集时，手动查看知识库，记录下包含答案的那一段的 ID。
    
- **方法 B（逆向查找）**：如果您已经有了 `ground_truth_context` 文本，写一个脚本去数据库里模糊匹配，找到对应的 ID。
    
- **方法 C（生成时记录）**：如果您是使用 LLM 自动生成 QA 对，确保生成脚本在读取文档块生成问题时，直接把该块的 ID 记录到 JSON 中。

---

### 3. 代码实现 (Python)

不需要引入复杂的库，手写一个简单的评估脚本最灵活。

Python

```python
import numpy as np
import math

class RetrieverEvaluator:
    def __init__(self, retriever_func):
        """
        retriever_func: 一个函数，输入 query，返回一个 list[doc_id]
        """
        self.retriever = retriever_func

    def calculate_metrics(self, test_dataset, top_k=5):
        hits = 0
        ndcg_scores = []
        
        print(f"开始评估: Top-K = {top_k}")

        for item in test_dataset:
            query = item['query']
            # 这里注意：如果是 Parent-Child 策略，我们通常关心是否召回了正确的 Parent
            target_id = item.get('expected_parent_id') or item.get('expected_doc_id')
            
            # 1. 执行检索，获取返回的 ID 列表
            # 这里的 retrieved_ids 应该是经过 Rerank 后的最终排序列表
            retrieved_ids = self.retriever(query, top_k=top_k)
            
            # 2. 计算 Hit Rate
            if target_id in retrieved_ids:
                hits += 1
                
                # 3. 计算 NDCG (Binary Relevance)
                # 找到 target_id 在列表中的索引 (0-based)
                rank = retrieved_ids.index(target_id)
                # NDCG 公式: rel / log2(rank + 2). 假设相关性 rel=1
                ndcg = 1.0 / math.log2(rank + 2)
            else:
                ndcg = 0.0
            
            ndcg_scores.append(ndcg)

        # 汇总结果
        hit_rate = hits / len(test_dataset)
        avg_ndcg = np.mean(ndcg_scores)
        
        return {
            "hit_rate": hit_rate,
            "ndcg": avg_ndcg
        }

# --- 模拟使用示例 ---

# 1. 定义您的检索包装函数
def my_rag_retrieval(query, top_k):
    # 这里调用您的真实业务代码
    # results = vector_store.similarity_search(query, k=top_k)
    # 或者 results = hybrid_retriever.invoke(...)
    # 或者 results = reranker.rerank(...)
    
    # 假设返回了对象，提取 ID
    # return [doc.metadata['doc_id'] for doc in results]
    pass 

# 2. 准备数据
dataset = [
    {"query": "测试问题1", "expected_doc_id": "id_100"},
    {"query": "测试问题2", "expected_doc_id": "id_200"},
    # ...
]

# 3. 运行评估
# evaluator = RetrieverEvaluator(my_rag_retrieval)
# metrics = evaluator.calculate_metrics(dataset, top_k=10)
# print(metrics)
```

---