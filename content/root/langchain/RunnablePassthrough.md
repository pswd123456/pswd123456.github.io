---
date created: 12月4日 , 21:29 , 2025
date modified: 12月5日 , 10:45 , 2025
---

```python
    @override

    def invoke(

        self, input: Other, config: RunnableConfig | None = None, **kwargs: Any

    ) -> Other:

        if self.func is not None:

            call_func_with_variable_args(

                self.func, input, ensure_config(config), **kwargs

            )

        return self._call_with_config(identity, input, config)
```

无论input是什么返回的都是input <- identity返回自身

func做什么用的? -> 让input跑一遍, 无论是打印日志, 还是记录, 还是需要取出input

```
Input (数据)
         │
         ▼
+-------------------------+
|   RunnablePassthrough   |
|                         |
|   1. 把 Input 给 func   | ---> func(Input) 执行 (比如打印日志)
|      (结果被丢弃)       |
|                         |
|   2. 返回 Input         |
+-------------------------+
         │
         ▼
   Input (数据原样输出)
```

`call_func_with_variable_args`总之是一个工具函数, 确保下游获得所需要的内容:

```
 Call function that may optionally accept a run_manager and/or config.
 
 Returns:

        The output of the function.
```

```python
def call_func_with_variable_args(

    func: Callable[[Input], Output]

    | Callable[[Input, RunnableConfig], Output]

    | Callable[[Input, CallbackManagerForChainRun], Output]

    | Callable[[Input, CallbackManagerForChainRun, RunnableConfig], Output],

    input: Input,

    config: RunnableConfig,

    run_manager: CallbackManagerForChainRun | None = None,

    **kwargs: Any,

) -> Output:

    """Call function that may optionally accept a run_manager and/or config.

  

    Args:

        func: The function to call.

        input: The input to the function.

        config: The config to pass to the function.

        run_manager: The run manager to pass to the function.

        **kwargs: The keyword arguments to pass to the function.

  

    Returns:

        The output of the function.

    """

    if accepts_config(func):

        if run_manager is not None:

            kwargs["config"] = patch_config(config, callbacks=run_manager.get_child())

        else:

            kwargs["config"] = config

    if run_manager is not None and accepts_run_manager(func):

        kwargs["run_manager"] = run_manager

    return func(input, **kwargs)  # type: ignore[call-arg]
```

下游可能需要的:

```
config -> 需要metadata\tags\recursion_limit
run_manager -> 需要流式输出token\中间步骤
```

`RunnablePassthrough` 的`self.func`接受Callable, 也就是任何函数
