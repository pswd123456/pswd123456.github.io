---
date created: 12月14日 , 0:29 , 2025
date modified: 12月14日 , 1:5 , 2025
---

## 核心痛点：普通的图做不到什么？

想象你是一个主厨（节点 A），你手上有 **3 个订单**：`["牛排", "意面", "沙拉"]`。

- 普通的边 (add_edge)：
    

    就像是你把这 3 个订单打包成一张大单子，递给一个厨师（节点 B）。

    
    - 结果：厨师 B 拿到单子，只能**按顺序**做：先做牛排，再做意面，最后做沙拉。效率低。
        
- 动态并行 (Send)：
    

    就像是你把这 3 个订单撕开，分发给 3 个不同的厨师（都是节点 B 的副本）。

    
    - 结果：厨师 B1 做牛排，厨师 B2 做意面，厨师 B3 做沙拉。他们**同时**开工，效率极高。
        

这就是 `Send` 的作用：**把一个大任务“撕开”，分发给同一个节点的多个副本同时处理。**

---

## 这里的“动态”是什么意思？

“动态”意味着：你在写代码的时候，根本不知道会有几个订单。

可能是 1 个，可能是 100 个。只有代码运行起来，拿到数据了，主厨才知道要分派几个厨师。这就是为什么普通的静态边（编译时确定的）无法满足需求，我们需要在运行时“动态”生成任务。

---

## 代码实战：从“大单子”到“小任务”

让我们把这个厨房场景变成代码。

### 1. 定义两份数据结构（关键点！）

这里的关键是：**主厨看的全貌** 和 **厨师看的局部** 是不一样的。

```Python
import operator
from typing import TypedDict, Annotated
from langgraph.types import Send
from langgraph.graph import StateGraph, START, END

# --- 1. 全局状态 (主厨看的账本) ---
class OverallState(TypedDict):
    # 待处理的订单列表
    orders: list[str] 
    # 做好的菜 (结果)，需要累加
    finished_dishes: Annotated[list[str], operator.add]

# --- 2. 子任务状态 (每个厨师拿到的小票) ---
# 注意：这跟 OverallState 结构完全不同！
class OrderState(TypedDict):
    dish_name: str
```

### 2. 定义主厨 (分发者)

主厨的工作是把 `orders` 列表拆开，变成一个个 `Send` 指令。

```Python
def head_chef(state: OverallState):
    # state["orders"] 是 ["牛排", "意面", "沙拉"]
    print(f"主厨: 收到了 {len(state['orders'])} 个订单，开始分发...")
    
    # 动态生成任务列表
    tasks = []
    for order in state["orders"]:
        # 语法: Send("目标节点名", {给该节点的私有输入})
        # 这里把 "牛排" 塞给了 "dish_name"
        task = Send("cook_node", {"dish_name": order})
        tasks.append(task)
        
    # 返回任务列表，引擎会并行执行它们
    return tasks
```

### 3. 定义厨师 (执行者)

厨师只关心他拿到的那张小票 (`OrderState`)。

```Python
def cook_node(state: OrderState):
    # 这里的 state 只有 {"dish_name": "牛排"}
    dish = state["dish_name"]
    print(f"  厨师: 正在做 {dish}...")
    
    # 返回结果。这个结果会被“汇聚”回全局状态的 finished_dishes
    # 因为全局状态里 finished_dishes 定义了 operator.add
    return {"finished_dishes": [f"热乎的{dish}"]}
```

### 4. 组装图

这里有一个特殊的连线方式：使用 `add_conditional_edges` 并返回 `path_map=["cook_node"]`。

```Python
builder = StateGraph(OverallState)
builder.add_node("head_chef", head_chef)
builder.add_node("cook_node", cook_node) # 注册厨师节点

# 关键连线：
# 告诉引擎：head_chef 会返回 Send 对象，目标是 cook_node
builder.add_conditional_edges(START, head_chef, ["cook_node"])

# 厨师做完就结束（其实是汇聚回主状态）
builder.add_edge("cook_node", END)

graph = builder.compile()
```

### 5. 运行结果

```Python
# 客户下单
input_data = {"orders": ["牛排", "意面", "沙拉"]}
result = graph.invoke(input_data)

print("\n最终结果:", result["finished_dishes"])
```

**运行过程模拟：**

1. **Start -> Head Chef**:
    
    - 主厨拿到 3 个订单。
        
    - 主厨发出 3 个 `Send` 指令。
        
2. **Engine (Map 阶段)**:
    
    - 引擎发现有 3 个 `Send`，于是启动了 3 个 `cook_node` 的副本。
        
    - 副本 1 输入: `{"dish_name": "牛排"}`
        
    - 副本 2 输入: `{"dish_name": "意面"}`
        
    - 副本 3 输入: `{"dish_name": "沙拉"}`
        
    - **这三个是同时跑的！**
        
3. **Engine (Reduce 阶段)**:
    
    - 三个厨师都做完了。
        
    - 引擎把他们返回的 `finished_dishes` 收集起来。
        
    - 根据 `OverallState` 里的 `operator.add` 规则，把三个结果拼在一起。
        
4. **End**:
    
    - 返回 `["热乎的牛排", "热乎的意面", "热乎的沙拉"]`。

---

## 总结

**动态并行 (`Send`)** 就是：

1. **Map (分发)**: 一个节点（主厨）遍历列表，为每一项创建一个 `Send("目标节点", 小数据)`。
    
2. **Execute (并行)**: 引擎启动 N 个目标节点副本，每个副本只处理那一份“小数据”。
    
3. **Reduce (汇聚)**: 所有副本的结果自动写回全局状态（Shared State），利用 `Annotated[list, operator.add]` 自动合并结果。
    

这下清楚了吗？它就是为了解决“如何高效处理一个未知长度的列表”而生的。