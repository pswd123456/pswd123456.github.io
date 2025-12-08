---
date created: 12月7日 , 17:1 , 2025
date modified: 12月7日 , 18:7 , 2025
---

`HybridChunker` 是 Docling 库中用于 **文档分块 (Chunking)** 的核心工具，特别为 RAG (检索增强生成) 场景设计。

它的名称“Hybrid（混合）”来源于它结合了两种分块策略的优势：

1. **基于文档结构的层级分块 (Hierarchical/Structural Chunking)**
    
2. **基于 Token 数量的线性分块 (Token-based Chunking)**
    

以下是关于 `HybridChunker` 的详细解析：

## 1. 核心设计理念

传统的文本分块器（如 LangChain 的 `RecursiveCharacterTextSplitter`）通常只把文档看作一长串纯文本，容易在不该切分的地方（如表格中间、句子中间）切断，导致语义丢失。

**`HybridChunker` 的工作方式：**

- **第一步（结构感知）：** 它首先利用 Docling 解析出的丰富文档结构（`DoclingDocument` 对象）。它知道哪里是标题、哪里是段落、哪里是表格。它会优先根据这些自然的语义边界（例如一个完整的段落作为一个块）来切分。
    
- **第二步（Token 细化）：** 它会检查生成的块是否超过了你设定的 Token 限制（例如 Embedding 模型的上下文窗口）。如果一个段落太长，它会再次进行更细粒度的切分（通常基于 [[Tokenizer]]），以确保块的大小适合模型处理。
    

## 2. 主要特点

- 语义完整性 (Semantic Coherence)：
    

    它尽量保持段落、列表项和表格的完整性，避免把一个完整的句子或表格行切碎，从而提高检索召回的准确率。

    
- 上下文感知 (Context Awareness)：
    

    生成的每个块（DocChunk）不仅包含文本，还包含元数据（Metadata）。例如，如果切分的是正文中的一段，它可能会自动附带该段落所属的 章节标题 作为上下文信息。

    
- Tokenizer 对齐：
    

    你可以传入特定的 Tokenizer（如 HuggingFace 的 AutoTokenizer 或 OpenAI 的 tokenizer）。HybridChunker 会使用这个 tokenizer 来精确计算长度，确保切分后的块 100% 符合你嵌入模型的输入限制，而不是通过字符数估算。

    

## 3. 代码示例

要在你的代码中使用 `HybridChunker`，通常需要结合 `DocumentConverter` 使用。

### 基础用法

```python
from docling.document_converter import DocumentConverter
from docling.chunking import HybridChunker

# 1. 转换文档
doc_converter = DocumentConverter()
conv_result = doc_converter.convert("https://arxiv.org/pdf/2408.09869") # 或本地文件路径
doc = conv_result.document  # 获取 DoclingDocument 对象

# 2. 初始化 HybridChunker
# 默认情况下，它使用简单的 tokenizer，但建议配置你实际使用的 embedding tokenizer
chunker = HybridChunker(
    tokenizer="sentence-transformers/all-MiniLM-L6-v2" # 指定 HuggingFace 模型 ID
)

# 3. 执行分块
chunk_iter = chunker.chunk(dl_doc=doc)

# 4. 查看结果
for i, chunk in enumerate(chunk_iter):
    print(f"Chunk {i}:")
    print(f"Text: {chunk.text[:100]}...") # 打印块内容的前100个字符
    print(f"Metadata: {chunk.meta}")       # 查看元数据（如标题路径、页码）
    print("-" * 20)
```

### 进阶配置

你可以自定义 Tokenizer 实例，而不是只传字符串 ID，这样控制力更强：

```python
from transformers import AutoTokenizer
from docling.chunking import HybridChunker

# 加载你的 Embedding 模型的 tokenizer
tokenizer = AutoTokenizer.from_pretrained("BAAI/bge-large-en-v1.5")

# 初始化 chunker
chunker = HybridChunker(
    tokenizer=tokenizer,
    max_tokens=512,        # 强制限制每个块的最大 token 数
    merge_peers=True       # 是否尝试合并相邻的小块（如连续的短句）以填充上下文窗口
)

chunks = list(chunker.chunk(doc))
```

## 4. 为什么它比普通 Splitter 好？

| **特性**   | **普通 Text Splitter** | **Docling HybridChunker** |
| -------- | -------------------- | ------------------------- |
| **输入**   | 纯文本字符串 (Plain Text)  | Docling 文档对象 (结构化数据)      |
| **表格处理** | 视为普通文本，容易切断行/列       | 理解表格结构，尽量保持表格完整           |
| **标题处理** | 只是普通的一行字             | 识别为标题，并可将其作为上下文附加到正文块中    |
| **切分边界** | 字符数/换行符              | 文档布局元素 (段落、列表、表格)         |
