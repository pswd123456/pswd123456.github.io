---
date created: 12月4日 , 21:54 , 2025
date modified: 12月5日 , 22:48 , 2025
---

## `RunnableLambda` 和 `@chain` 🛠️

可以理解为“将普通函数转化为 Runnable”。

它们的作用就是把一个普通的 Python 函数“升级”成 LangChain 的组件，让这个函数拥有 `invoke`、`stream`、`batch` 以及最重要的 `|`（管道连接）能力。

- **`RunnableLambda`**：是一个**类**。你手动把函数传给它。
    

    Python

    ```
    # 你的普通函数
    def add_one(x): return x + 1
    
    # 变成了 Runnable
    runnable = RunnableLambda(add_one)
    ```

- **`@chain`**：是一个**装饰器**（Syntactic Sugar，语法糖）。它本质上就是在你的函数外面包了一层 `RunnableLambda`。
    

    Python

    ```
    from langchain_core.runnables import chain
    
    @chain
    def add_one(x):
        return x + 1
    # add_one 现在直接就是一个 Runnable 了
    ```

**为什么要这么做？** 为了**“标准化”**。一旦变成了 `Runnable`，你的函数就可以无缝地插入到 LangChain 的任何管道中，和 OpenAI 的模型、向量数据库检索器等组件平起平坐，像乐高积木一样拼在一起。