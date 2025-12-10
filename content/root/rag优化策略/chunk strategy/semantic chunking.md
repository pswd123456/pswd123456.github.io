---
date created: 12月5日 , 19:35 , 2025
date modified: 12月10日 , 16:5 , 2025
---

## 语义切分

按句子切分后对比前后句的embeddings score, 相似就保留, 不相似就切断

token消耗会是原始文本的二到四倍, 

主要原因是有

- 重叠计算(滑动窗口), <-探测阶段
- 二次计算: 探测阶段不会存储, 纯粹计算相似度, 存储仍然需要embedding 一次

感觉没什么道理, 

首先是这样, bi-encoder的相似度很多时候会失能, 而且embedding不到一块很多时候不意味着他们不相关, 

第一, 如果出现embedding分布很大, 但是事实上在说一件事情, 这种chunking就失效了, 

第二, 如果一个巨大的段落全都在说一件事情(假设在embeddings里都映射到一块了, 但是这可能吗?), 最后还是要按其他组件的token限制切

所以semantic chunking一般不会单独使用

而且如果用docling的视觉模型识别layout效果会比用sematic chunking来的好一些, 使用父子索引的情况下无需再二次使用语义切分, 因为一两百token的子块的量级怎么切其实影响都不太大