---
date created: 12月4日 , 19:47 , 2025
date modified: 12月5日 , 12:58 , 2025
---

## langchain

### 一个标准的 RAG 链条就是这样闭环的

[[Prompts]] (组装 Message 对象) -> LCLE chain [[Runnables]] (吃 Message，吐 AIMessage) -> [[TextSplitter]] -> [[Retriever]] -> [[OutputParser]] (吃 AIMessage，吐 String)

> [!NOTE]
> 仅包含以下文件的内容
> FROM DIR: langchain_core, langchain, langchain_textsplitter

```
├─retrievers.py
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
