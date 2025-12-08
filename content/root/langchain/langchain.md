---
date created: 12月4日 , 19:47 , 2025
date modified: 12月7日 , 17:0 , 2025
---

## langchain

### 一个标准的 RAG 链条

[[Prompts]] (组装 [[message]] 对象) -> LCLE chain [[Runnables]] (吃 Message，吐 AIMessage) -> [[TextSplitter]] -> [[Retriever]] -> [[OutputParser]] (吃 AIMessage，吐 String)

全程负责对上下游进行监控\传递信息\建立chain的树结构的 -> [[RunManager]]

> [!NOTE]
> 仅可能涉及以下目录的内容
> 
> langchain git repository: langchain_core, langchain, langchain_textsplitter

```
├─retrievers.py
│  
├─embeddings.py
│
├─callbacks
│      manager.py
│      
├─out_parser
│      string.py
│      
├─prompts
│      chat.py
│      
├─runnables
│      base.py
│      config.py
│      passthrough.py
│      
├─text_splitters
│      base.py.py
│      character.py
│      
└─vectorstores
        base.py
        in_memory.py
        utils.py
        __init__.py
```