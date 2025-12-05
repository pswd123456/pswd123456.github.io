---
date created: 12月5日 , 15:43 , 2025
date modified: 12月5日 , 16:39 , 2025
---

## 背包：`RunnableConfig` 🎒

请看 **`runnables/config.py`**。

它本质上就是一个 Python 的 `TypedDict`（带类型提示的字典），规定了这个“背包”里能装什么：

```python

class RunnableConfig(TypedDict, total=False):
    tags: list[str]             # 标签，例如 ["production", "chatbot"]
    metadata: dict[str, Any]    # 元数据，例如 {"user_id": "123", "session_id": "abc"}
    callbacks: Callbacks        # 装着各种 Handler（比如 LangSmith Tracer, ConsoleLogger）
    run_name: str               # 当前这一步的名字
    max_concurrency: int | None # 并发限制
    recursion_limit: int        # 递归深度限制
    configurable: dict[str, Any]# 运行时动态配置参数
    run_id: uuid.UUID | None    # 唯一运行 ID
```

**它的作用：** 在链条（Chain）中透传。就像接力赛的接力棒，每一棒跑完后，会把这个背包（可能经过修补）传给下一棒。
