---
date created: 12月7日 , 17:6 , 2025
date modified: 12月7日 , 18:8 , 2025
---

`HybridChunker` 的工作机制非常依赖 `tokenizer` 来确保切分出的文本块（Chunk）能够被你的 Embedding 模型（即 `paraphrase-multilingual-MiniLM-L12-v2`）完美消化。

简单来说，`HybridChunker` 并不是在“分词”阶段去切分文档，而是用 Tokenizer 作为**“尺子”**来衡量和修剪文档结构。

以下是具体的**三阶段工作流程**，以及针对你使用的多语言模型的特殊优势：

## 1. 工作流程：Tokenizer 如何参与分块？

`HybridChunker` 遵循“先结构，后 Token”的策略：

- 第一步：结构化提取 (Structural Extraction)
    

    它首先从 DoclingDocument 中提取自然的文档结构（如：标题、段落、列表项、表格）。此时，Tokenzier 尚未介入。它得到的是一个个完整的语义单元。

    
- 第二步：Token 计数与校验 (Measurement)
    

    对于提取出的每一个结构单元（例如一个长段落），HybridChunker 会调用你传入的 tokenizer 进行编码（Encode）。

    
    - **操作**：调用 `tokenizer.tokenize(text)` 或 `len(tokenizer.encode(text))`。
        
    - **目的**：获取该段落精确的 Token 数量。
        
    - **例子**：对于 `paraphrase-multilingual-MiniLM-L12-v2`，中文字符串 `"人工智能"` 可能会被分解为 2-3 个 token（取决于词表），而不仅仅是 4 个字符。
        
- **第三步：细化与合并 (Refinement)**
    
    - **如果块太长 (> max_tokens)**：Chunker 发现某一段落的 token 数超过了你设定的限制（例如 512）。它会利用 Tokenizer 将其强制切分。它会尝试在句号、换行符等自然边界切分，切分后再次用 Tokenizer 检查，确保每一小块都小于 `max_tokens`。
        
    - **如果块太短 (且 merge_peers=True)**：它会尝试将相邻的短段落（如连续的短句）合并。在合并前，它会先计算 `len(tokens_A + tokens_B)`，只有当总和小于限制时才执行合并。
        

## 2. 针对 `paraphrase-multilingual-MiniLM-L12-v2` 的优势

你选择的这个模型是**多语言 (Multilingual)** 模型，通常使用 **SentencePiece** 分词算法。在 `HybridChunker` 中使用它有巨大的优势：

1. **解决“中文计数”难题**：
    
    - **普通 Splitter**：通常按字符数切分（例如 `chunk_size=500` 指 500 个字符）。但对于中文，500 个字符可能对应 700-800 个 token（因为某些生僻字或词组会占多个 token），这会导致送入模型时被截断，丢失信息。
        
    - **HybridChunker + 你的 Tokenizer**：它直接使用模型的词表来计数。它知道对于这个特定的模型，这段中文到底占多少容量。这保证了**生成的 Chunk 永远不会因为超长而被模型截断**。
        
2. **保留完整语义**：
    
    - 由于这个 tokenizer 懂多语言，它能更准确地识别词的边界，避免在一个完整的词（token）中间强行切断。
        

## 3. 代码实现

要在代码中正确“激活”这种能力，你需要显式加载这个 Tokenizer 并传给 `HybridChunker`。

```Python
from docling.chunking import HybridChunker
from transformers import AutoTokenizer

# 1. 加载你的特定模型的 Tokenizer
# 这一步非常重要，确保 Chunker 的"尺子"和你后续用的 Embedding 模型是同一把
MODEL_ID = "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
tokenizer = AutoTokenizer.from_pretrained(MODEL_ID)

# 2. 初始化 HybridChunker
chunker = HybridChunker(
    tokenizer=tokenizer,      # 传入你的 tokenizer
    max_tokens=512,           # 设定该模型的最大上下文窗口 (通常 MiniLM 是 512)
    merge_peers=True          # 推荐开启：允许合并相邻短句，填满窗口
)

# 3. 开始分块
# doc 是之前通过 DocumentConverter 转换得到的 DoclingDocument 对象
chunk_iter = chunker.chunk(dl_doc=doc)

for chunk in chunk_iter:
    # 这里的 chunk.text 长度是经过你的 tokenizer 精确计算过的
    print(f"Token count: {len(tokenizer.encode(chunk.text))}") 
    print(chunk.text)
```

## 总结

`HybridChunker` 使用 tokenizer **不是为了把文本变成 ID 给模型用**（这步是 Embedding 模型做的事），而是把它当作一个**精确的计数器**。

当你传入 `paraphrase-multilingual-MiniLM-L12-v2` 的 tokenizer 时，你就赋予了 Chunker **“多语言感知能力”**，让它能精准地切分中文、日文或混合文本，不再受制于简单的字符长度统计。