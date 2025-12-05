---
date created: 12月5日 , 15:43 , 2025
date modified: 12月5日 , 16:15 , 2025
---

[[#我的理解]]

## [[RunnableConfig]]
### `CallbackManager` & `RunManager` 🤵

请看 **`callbacks/manager.py`**。

这是 LangChain 的神经系统。

- **`CallbackManager`**：是总管。它管理着一堆 `Handlers`（处理程序）。当你想开始一个任务时，你找它。
    
- **`RunManager`** (例如 `CallbackManagerForChainRun`)：是分管。当你调用 `CallbackManager.on_chain_start()` 后，它会返回一个专门负责**当前这一次运行**的 `RunManager`。
    

**管家的工作流程：**

1. **开始 (`on_..._start`)**：通知所有人“我要开始了！”
    
2. **过程 (`on_..._new_token`)**：对于 LLM，每生成一个字，都要通知“又来了一个字！”
    
3. **结束/错误 (`on_..._end` / `error`)**：通知“我搞定了！”或者“我挂了！”


### 它们(config&&manager)是如何配合的？(The Workflow) 🔄

这是最精彩的部分。让我们回到 **`runnables/base.py`** 的 `_call_with_config` 方法（它是 `invoke` 的幕后实现）来看这个过程：

```python
def _call_with_config(
        self,
        func: ...,
        input_: Input,
        config: RunnableConfig | None,
        # ...
    ) -> Output:
        # 1. 整理背包：确保 config 是一个合法的字典
        config = ensure_config(config)
        
        # 2. 召唤总管：从 config['callbacks'] 里把 CallbackManager 找出来
        #    如果没有，就创建一个新的。
        callback_manager = get_callback_manager_for_config(config)
        
        # 3. 开始任务：告诉总管“我（这个Runnable）要开始了”
        #    总管会触发所有 Handler 的 on_chain_start
        #    并返回一个专门负责“这一次运行”的分管：run_manager
        run_manager = callback_manager.on_chain_start(
            serialized,
            input_,
            name=config.get("run_name") or self.get_name(),
            run_id=config.pop("run_id", None),
        )
        
        try:
            # 4. 修补背包：这是关键！
            #    我们创建了一个 child_config。
            #    为什么？因为下一步（子步骤）需要向“当前步骤”汇报。
            #    所以把 run_manager.get_child() 放进了背包的 callbacks 里。
            child_config = patch_config(config, callbacks=run_manager.get_child())
            
            # 5. 执行任务：带着这个新的、修补过的背包去执行具体的函数
            #    context.run 是为了处理 ContextVar（上下文变量），保证异步/线程安全
            with set_config_context(child_config) as context:
                output = context.run(
                    call_func_with_variable_args, # 那个“适配器”函数
                    func,
                    input_,
                    config,      # 原始配置（有时候函数需要）
                    run_manager, # 当前运行的分管（有时候函数需要手动触发事件）
                    # ...
                )
        except BaseException as e:
            # 6a. 报错：通知分管
            run_manager.on_chain_error(e)
            raise
        else:
            # 6b. 成功：通知分管
            run_manager.on_chain_end(output)
            return output
```

### 总结图示

假设你有一个链：`A | B`。

1. **用户调用** `A.invoke(input, config)`。
    
2. **A 启动**：
    
    - 从 `config` 创建 `Manager_A`。
        
    - `Manager_A` 喊了一嗓子：“A 开始了！” (Log: A Started)
        
3. **传递给 B**：
    
    - A 准备传给 B。A 会把 `config` 修改一下（patch）。
    -> 代码:

```python
class ParentRunManager(RunManager):
    def get_child(self, tag: str | None = None) -> CallbackManager:
        # 关键点！！！
        # 创建一个新的 Manager，并明确告诉它：“你的爸爸是我（self.run_id）”
        manager = CallbackManager(handlers=[], parent_run_id=self.run_id)
        
        # 顺便把家族传承的东西（Handler, Tags, Metadata）也给孩子一份
        manager.set_handlers(self.inheritable_handlers)
        manager.add_tags(self.inheritable_tags)
        manager.add_metadata(self.inheritable_metadata)
        
        return manager
```

```python
child_config = patch_config(config, callbacks=run_manager.get_child())
```

1. **B 启动**：
    
    - B 从 `config_for_B` 拿到 `Manager_B`（它是 A 的孩子）。
        
    - `Manager_B` 喊一嗓子：“B 开始了！”。
        
    - 监控系统（如 LangSmith）看到 B 是 A 的孩子，就会画出一个**树状图**。
        
2. **B 结束** -> `Manager_B` 汇报“B 结束”。
    
3. **A 结束** -> `Manager_A` 汇报“A 结束”。


## 我的理解

- `config` 是 `runnable` 的参数存储的地方, 
	- 比如 `runnable` 的 `metadata`, 比如从 `retriever` 上传下来一个 `source`, `下游的runnable` 比如 `prompt` 会接受这个 `source` 并且打在 `prompt` 的 `source` 里, `还有比如session_id` 之类的
- `CallbackManager` 就类似于封装了一堆管理程序, 
	- 具体比如: 我的项目中用了 `langfuse` 的 `handler`, 将执行的 `latency` 和 `token` `传给langfuse` 后端, 
- `RunManager` 就类似于处理当前任务的 `CallbackManager`, 负责跟踪当前任务的 `lifecycle`, 并且向上下游负责传递执行信息
	- 具体来说, `child_config` 会存有一个 `parent_run_id` 