---
date created: 12月13日 , 3:24 , 2025
date modified: 12月13日 , 4:25 , 2025
---

这是 FastAPI 最迷人、也是最复杂的章节。

许多开发者认为 `Depends` 只是简单的函数调用，但实际上，FastAPI 在底层实现了一个**支持异步、基于图（Graph）解析、且具备上下文管理功能的依赖注入容器**。

如果不读源码，你很难理解为什么：

1. **Request Scoped Caching**: 同一个请求中，多次引用同一个 `Depends`，它只执行一次？
    
2. **Yield Support**: 为什么函数 `yield` 后面的代码能在请求结束后执行？
    

我们将深入 `fastapi/dependencies/utils.py` 和 `fastapi/routing.py`。

---

## 1. 静态构建：依赖图谱的生成 (Startup Phase)

FastAPI 不是在每次请求进来时才去分析你的函数参数，而是在**应用启动时**就做好了。

**源码位置：** `fastapi/dependencies/utils.py` -> `get_dependant`

### 📝 核心逻辑解析

当你写下 `@app.get("/")` 时，FastAPI 会调用 `get_dependant`。它是一个递归函数，负责"扫描"你的代码。

```Python
# 伪代码逻辑简化
def get_dependant(path: str, call: Callable) -> Dependant:
    # 1. 获取函数签名 (Signature)
    signature = inspect.signature(call)
    
    # 2. 遍历参数
    for param_name, param in signature.parameters.items():
        # 如果参数默认值是 Depends(...)
        if isinstance(param.default, params.Depends):
            # 3. 递归！解析这个依赖自己的依赖
            # 这就构建了一个树状结构 (DAG)
            sub_dependant = get_dependant(
                path=path, 
                call=param.default.dependency
            )
            dependant.dependencies.append(sub_dependant)
            
    return dependant
```

**🔍 深度洞察：**

- FastAPI 将你的路由函数和它所有的依赖（以及依赖的依赖）转换成了一个 `Dependant` 对象树。
    
- 这就是为什么启动时如果依赖写错了会报错，而不是等到请求进来才报错。

---

## 2. 运行时执行：`solve_dependencies` (Runtime Phase)

这是整个系统的心脏。每当一个 HTTP 请求到达，FastAPI 会调用这个函数来填充参数。

**源码位置：** `fastapi/dependencies/utils.py` -> `solve_dependencies`

### 📝 源码级逻辑 (缓存与递归)

```Python
async def solve_dependencies(
    *,
    request: Request,
    dependant: Dependant,
    dependency_cache: dict[tuple, Any], # ⭐ 关键：请求级缓存
    # ... 其他参数
) -> tuple[dict, list, Any, Any]:

    values: dict[str, Any] = {}
    
    # 1. 遍历当前层级的所有子依赖
    for sub_dependant in dependant.dependencies:
        
        # 2. ⭐ 缓存检查机制
        # call 是依赖函数的引用。如果这个函数在这个请求中已经被执行过
        # 直接从缓存拿结果，不再执行！
        if sub_dependant.call in dependency_cache:
            values[sub_dependant.name] = dependency_cache[sub_dependant.call]
            continue

        # 3. 递归调用：先解决子依赖的子依赖
        sub_values = await solve_dependencies(
            request=request,
            dependant=sub_dependant,
            dependency_cache=dependency_cache
        )

        # 4. 执行真正的依赖函数
        # run_in_threadpool 确保同步的依赖不会阻塞 Loop
        if is_coroutine_callable(sub_dependant.call):
            solved = await sub_dependant.call(**sub_values)
        else:
            # 如果依赖是同步函数（def），在这里被丢进线程池
            solved = await run_in_threadpool(sub_dependant.call, **sub_values)

        # 5. 存入缓存
        dependency_cache[sub_dependant.call] = solved
        values[sub_dependant.name] = solved

    return values
```

💡 实战启示 (Singleton 模式)：

你在 Depends(get_current_user) 中查了数据库。如果你的 Service 层也依赖 get_current_user，而在 Controller 层也依赖它。

由于 dependency_cache 的存在，数据库只会查询一次。这就是 FastAPI 的默认行为（除非你设置 use_cache=False）。

---

## 3. "Yield" 的魔法：`AsyncExitStack`

这是处理数据库 Session 最关键的部分。为什么 `yield` 能够工作？

**源码位置：** `fastapi/dependencies/utils.py` -> `solve_generator`

FastAPI 利用了 Python `contextlib` 中的 `AsyncExitStack`。这是一个栈结构，专门用来管理多个上下文管理器。

### 📝 源码逻辑

当 `solve_dependencies` 发现你的依赖是一个 Generator (带 `yield`) 时：

```Python
# 伪代码：如何处理 yield 依赖
async def solve_generator(..., stack: AsyncExitStack):
    # 1. 启动生成器 (相当于 __aenter__)
    cm = contextmanager_in_threadpool(call(**values))
    
    # 2. 将这个上下文管理器推入栈中
    # stack 会记住它的 __aexit__ 方法，留待以后调用
    value = await stack.enter_async_context(cm)
    
    # 3. 返回 yield 出出来的值给视图函数
    return value
    
    # 注意：这里没有调用 exit！代码暂停在了 yield 处
```

然后，在 `fastapi/routing.py` 的请求处理尾部：

```Python
async def app(request):
    # 创建一个栈，用于收集所有 yield 依赖的清理函数
    async with AsyncExitStack() as stack:
        # 解析依赖，把 stack 传进去
        values = await solve_dependencies(..., dependency_overrides_provider=stack)
        
        # 执行用户的视图函数
        response = await run_endpoint_function(..., values)
        
        return response
    # ⭐ 离开 async with 块时：
    # AsyncExitStack 会自动按 "后进先出" (LIFO) 的顺序
    # 调用所有依赖的 __aexit__ (即 yield 后面的代码)
```

🔍 深度解析：

这解释了为什么你不需要手动关闭数据库连接。

1. 请求开始 -> 创建 `AsyncExitStack`。
    
2. 依赖注入 -> `get_db` 运行到 `yield`，Session 开启。`session.close()` 被注册到 Stack 中。
    
3. 业务逻辑 -> 运行你的代码。
    
4. 发送响应 -> 返回 JSON 给用户。
    
5. **请求结束** -> Stack 触发，执行 `session.close()`。

---

## 4. 总结：对学习路线的影响

读完这部分源码，对你的阶段二和阶段三（Web开发）有极大的指导意义：

1. 混合开发的安全性：
    

    源码显示 solve_dependencies 内部会判断 is_coroutine_callable。所以，在 async def 的视图函数中，混用同步的 def 依赖是完全安全的（FastAPI 会自动切线程）。这对于复用旧代码库非常有价值。

    
2. Scope 的生命周期：
    

    Depends 的生命周期严格绑定于 Single Request。如果你需要在多个 Request 间共享数据（比如全局数据库连接池），不能依赖 Depends 的缓存机制，而应该将其存储在 app.state 或全局变量中，通过 Depends 去取。

    
3. Yield 的坑：
    

    由于 BackgroundTasks（后台任务）是在 Response 发送后才运行的，而 AsyncExitStack 也是在 Response 发送后关闭的。

    

    竞态风险： 如果你的后台任务依赖于一个 yield 出来的数据库 Session，可能会出现任务还没跑完，Session 已经被 Stack 关闭了。

    
    - **解决：** 对于后台任务，必须在该任务内部单独创建 Session，而不要复用请求级的 `Depends` Session。

---
