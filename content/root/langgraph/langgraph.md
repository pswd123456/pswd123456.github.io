---
date created: 12月13日 , 23:29 , 2025
date modified: 12月14日 , 1:13 , 2025
---

LangGraph 是一个用于构建有状态、多角​​色应用程序的库，其核心深受 Google Pregel 图计算模型的启发。

## 第一阶段：[[掌握核心构建块]] (Graph Construction)

这是绝大多数用户与 LangGraph 交互的入口。你需要理解如何定义状态、节点和边。

1. **StateGraph (状态图)**
    
    - **目标**: 理解如何初始化图、添加节点和边、以及编译图。
        
    - **核心文件**: `langgraph/graph/state.py`
        
    - **关键点**:
        
        - `StateGraph` 类是构建图的主要入口，它接受 `state_schema`。
            
        - `add_node` 方法用于注册可运行对象（Runnable）或函数。
            
        - `add_edge` 和 `add_conditional_edges` 定义控制流。
            
        - `compile()` 方法将图转换为可执行的 `CompiledStateGraph` (它是 `Pregel` 的子类)。
            
2. **常用常量**
    
    - **目标**: 知道如何定义图的起点和终点。
        
    - **核心文件**: `langgraph/constants.py`
        
    - **关键点**: `START` 和 `END` 节点是图结构的基础常量。
        
3. **消息图与消息状态 (Chatbots 专用)**
    
    - **目标**: 学习处理对话历史的专用工具。
        
    - **核心文件**: `langgraph/graph/message.py`
        
    - **关键点**:
        
        - `add_messages` 是一个 reducer 函数，用于合并消息列表（支持追加和根据 ID 更新）。
            
        - `MessagesState` 是一个预定义的 TypedDict，包含带 `add_messages` 注解的 `messages` 键。

---

## 第二阶段：[[理解数据流与状态管理]] (Channels)

LangGraph 的状态管理依赖于“通道（Channels）”概念。节点不直接通信，而是通过写入通道来更新共享状态。

1. **通道基类与机制**
    
    - **目标**: 理解通道的生命周期（Update -> Checkpoint -> Get）。
        
    - **核心文件**: `langgraph/channels/base.py`
        
    - **关键点**: `BaseChannel` 定义了 `update`（接收新值）和 `get`（获取当前值）等抽象方法。
        
2. **核心通道类型**
    
    - **LastValue**: 默认行为，保留最后一个更新的值（通常用于替换状态）。
        
    - **BinaryOperatorAggregate**: 用于 `Annotated[list, operator.add]` 等场景，通过 reducer 函数聚合更新。
        
    - **Topic**: 用于发布-订阅模式，通常用于处理 `Send` 等动态任务。
        
    - **EphemeralValue**: 瞬态值，仅在当前步骤有效，用于传递临时信号。

---

## 第三阶段：[[深入运行时引擎]] (Pregel Engine)

这是 LangGraph 的心脏。理解这里能帮你明白图是如何通过“超步（Super-steps）”运行的。

1. **Pregel 主逻辑**
    
    - **目标**: 理解图的编译结果和执行入口。
        
    - **核心文件**: `langgraph/pregel/main.py`
        
    - **关键点**: `Pregel` 类实现了 `stream`、`invoke` 等方法。它负责协调检查点保存器（Checkpointer）和运行循环。
        
2. **执行循环 (The Loop)**
    
    - **目标**: 理解 Plan-Execute-Update 周期。
        
    - **核心文件**: `langgraph/pregel/_loop.py`
        
    - **关键点**: `PregelLoop.tick` 方法是核心迭代器。它负责准备下一个任务 (`prepare_next_tasks`)，执行任务，应用写入 (`apply_writes`)，并管理检查点。
        
3. **读取与写入 (IO)**
    
    - **核心文件**: `langgraph/pregel/_read.py` 和 `langgraph/pregel/_write.py`
        
    - **关键点**:
        
        - `ChannelRead` 负责从状态中提取数据传给节点。
            
        - `ChannelWrite` 负责将节点返回的数据转换成通道更新操作。

---

## 第四阶段：[[高级控制流与类型]] (Advanced Control)

学习如何控制复杂的图行为，如动态路由、人工介入和重试。

1. **控制原语**
    
    - **核心文件**: `langgraph/types.py`
        
    - **关键点**:
        
        - `Command`: 用于从节点内部动态更新状态、跳转节点或恢复中断。
            
        - `Send`: 用于 Map-Reduce 模式，将输入发送到特定节点（通常用于并行处理）。
            
        - `interrupt`: 用于暂停图的执行以等待人工输入。
            
        - `RetryPolicy`: 定义节点失败时的重试策略。
            
2. **配置与运行时访问**
    
    - **核心文件**: `langgraph/config.py` 和 `langgraph/runtime.py`
        
    - **关键点**: 了解如何在节点内部通过 `get_config()` 获取配置，或通过 `Runtime` 对象访问 Store 和上下文,。

---

## 第五阶段：[[函数式 API]] (Functional API)

这是 LangGraph 新引入的一种更 Pythonic 的定义方式（类似 @task 装饰器）。

1. **Task 和 Entrypoint**
    
    - **核心文件**: `langgraph/func/__init__.py`
        
    - **关键点**: 学习如何使用 `@task` 定义原子操作，使用 `@entrypoint` 定义工作流，而不必显式定义 `StateGraph` 类。

