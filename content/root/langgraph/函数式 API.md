---
date created: 12月14日 , 1:6 , 2025
date modified: 12月14日 , 1:13 , 2025
---

欢迎来到 **第五阶段：函数式 API (Functional API)**。

这是 LangGraph 最新引入的一种构建工作流的方式。如果你觉得 `StateGraph` 的“定义节点 -> 定义边 -> 编译”过程有点繁琐，或者你更习惯写普通的 Python 函数，那么函数式 API 会让你感觉非常亲切。

它的核心理念是：**用写代码的方式写图，而不是“画”图。**

核心文件是 `langgraph/func/__init__.py`，它引入了两个关键装饰器：`@task` 和 `@entrypoint`。

## 1. `@task`: 原子操作

`@task` 用于装饰一个函数，将其变成图中的一个**节点**。

- **特点**:
    
    - 调用被装饰的函数时，它不会立即执行，而是返回一个 **Future** 对象（类似于 `asyncio.Future` 或 `concurrent.futures.Future`）。
        
    - 这意味着你可以轻松实现并行：先发起多个任务，稍后统一获取结果。
        
    - 它支持重试策略 (`retry_policy`) 和缓存策略 (`cache_policy`)。
        

**代码示例:**

```Python
from langgraph.func import task

@task(retry_policy=...) 
def generate_joke(subject: str) -> str:
    return f"为什么 {subject} 过马路？为了去对面！"
```

## 2. `@entrypoint`: 定义工作流

`@entrypoint` 用于装饰主函数，将其变成一个 **Pregel 图**（相当于 `StateGraph` 编译后的结果）。

- **特点**:
    
    - 它负责编排 `@task` 的执行。
        
    - 它管理检查点（Checkpointer）和持久化。
        
    - **函数签名**: 必须接受一个参数作为输入。
        

**代码示例:**

```Python
from langgraph.func import entrypoint

@entrypoint()
def workflow(subjects: list[str]):
    # 1. 启动并行任务 (Map)
    futures = [generate_joke(s) for s in subjects]
    
    # 2. 等待结果 (Reduce)
    results = [f.result() for f in futures]
    
    return results
```

## 3. 为什么说它实现了“动态并行”？

回想一下上一阶段的 `Send`。在函数式 API 中，你不需要特殊的 `Send` 指令。你只需要利用 Python 原生的**列表推导式**和 `@task` 返回的 Future 特性。

在上面的 `workflow` 中：

1. `[generate_joke(s) for s in subjects]` 会瞬间创建 N 个任务（Future）。
    
2. LangGraph 引擎会并行调度这些任务。
    
3. `f.result()` 会阻塞直到该任务完成。
    

这比 `StateGraph` + `Send` 写起来要直观得多，因为它符合 Python 程序员的直觉。

## 4. 状态管理：`previous` 参数

在函数式 API 中，我们不再显式定义一个 `TypedDict` 来作为全局状态。相反，状态管理变得更加隐式，或者通过 `checkpointer` 持久化。

如果启用了 `checkpointer`，你的入口函数可以接收一个名为 `previous` 的特殊参数。

- **`previous`**: 它是上一次运行结束时的返回值（或者显式保存的状态）。
    
- **用途**: 实现长短期记忆、累加历史等。
    

**代码示例：带记忆的累加器**

```Python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver

# 启用检查点
@entrypoint(checkpointer=InMemorySaver())
def accumulator(number: int, *, previous: int = 0) -> int:
    # previous 会自动注入上一次的结果
    # 如果是第一次运行，previous 使用默认值 0
    current_sum = previous + number
    return current_sum

# 运行
config = {"configurable": {"thread_id": "1"}}

print(accumulator.invoke(10, config)) # 输出 10 (0 + 10)
print(accumulator.invoke(5, config))  # 输出 15 (10 + 5)
```

## 5. `entrypoint.final`: 分离“返回值”与“存储值”

有时你想返回给用户看的是 "任务完成"，但在数据库（Checkpoint）里存的应该是 "任务结果数据"。

`entrypoint.final` 允许你区分这两者。

```python
from langgraph.func import entrypoint

@entrypoint(checkpointer=InMemorySaver())
def process_data(data: str, *, previous: list = None):
    history = previous or []
    history.append(data)
    
    # value: 返回给 invoke() 调用者的值
    # save: 存入 Checkpoint 供下一次 previous 使用的值
    return entrypoint.final(
        value=f"Processed {data}", 
        save=history
    )
```

## 总结：StateGraph vs Functional API

|**特性**|**StateGraph API (类定义)**|**Functional API (装饰器)**|
|---|---|---|
|**思维模型**|**图论**：显式定义节点和边|**函数式**：定义函数调用栈|
|**状态定义**|**显式**：必须定义 Schema (TypedDict)|**隐式**：通过函数参数和返回值传递|
|**控制流**|边 (Edges), 条件边, Command|Python 原生控制流 (if/for/while)|
|**并行**|需要 `Send` 或并行分支|列表推导式 + Futures|
|**适用场景**|复杂的、非线性的、高度结构化的状态机|快速原型、数据处理管道、简单的 Agent|

## 学习路径结语

恭喜你！你已经了解了 LangGraph 的全貌：

1. **Phase 1**: 用 `StateGraph` 画出流程骨架。
    
2. **Phase 2**: 用 `Channels` (`Annotated`) 管理数据血液。
    
3. **Phase 3**: 理解 `Pregel` 引擎的心跳机制（快照与写入）。
    
4. **Phase 4**: 用 `Command` 和 `Send` 指挥复杂的动态行为。
    
5. **Phase 5**: 用 `Functional API` 回归 Python 原生体验。
    

下一步建议：

选择一种风格（通常推荐从 StateGraph 开始，因为它更直观地对应“图”的概念，且功能最全），找一个具体的业务场景（比如“RAG 问答”或“多步骤工具调用 Agent”），开始写你的第一个应用吧！

如果你在实战中遇到报错，随时把代码发给我，我帮你 Debug！