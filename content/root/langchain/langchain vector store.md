---
date created: 12月4日 , 19:47 , 2025
date modified: 12月4日 , 20:11 , 2025
---

## langchain vector store

### `InMemoryVectorStore`

#### `__init__`

```python
self.store: dict[str, dict[str, Any]] = {}
```

模拟了 `索引, vector` 的结构, 每一条包含了在向量空间的`dense`权重

#### `add_documents`

```python
self.store[doc_id_] = {

                "id": doc_id_,

                "vector": vector,

                "text": doc.page_content,

                "metadata": doc.metadata,

            }
```

存进了id和chunk原文以及metadata, 方便retrieval给出doc_id返回text和metadata

metadata可能存了什么信息? 可以是doc_source, author, 或者created_date等等, 方便检索时过滤信息

### `_similarity_search_with_score_by_vector`

如何计算similarity的? ->

```python
similarity = cosine_similarity([embedding], [doc["vector"] for doc in docs])[0]
```

计算query的embedding和doc的向量进行cosine相似度比较

```python
top_k_idx = similarity.argsort()[::-1][:k]
```

返回topk

cosine比较就类似于二维向量计算cosine角的算法, 只不过拓展到指定维度, 这个值通常在 -1, 1之间, 越大表示向量越相关

可以参见 ->  `utils.py` -> `_cosine_similarity`

```python
 similarity = np.dot(x, y.T) / np.outer(x_norm, y_norm)
```

- **分子**：`np.dot(x, y.T)` 是向量的点积（Dot Product）。
    
- **分母**：`np.outer(x_norm, y_norm)` 是两个向量模长（Norm）的乘积。

