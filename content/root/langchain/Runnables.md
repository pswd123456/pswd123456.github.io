---
date created: 12月4日 , 21:10 , 2025
date modified: 12月5日 , 18:8 , 2025
---

## Runnables

Runnable就是LangChain的基础组件, 

包含基础的invoke\stream\batch

以及可以使用 `"|"`进行管道连接

```python
"""
**`RunnableSequence`** invokes a series of runnables sequentially, with

    one Runnable's output serving as the next's input. Construct using

    the `|` operator or by passing a list of runnables to `RunnableSequence`.
    
    
**`RunnableParallel`** invokes runnables concurrently, providing the same input

to each. Construct it using a dict literal within a sequence or by passing a

dict to `RunnableParallel`.
    
"""
```

Docstring 对 runnable 包括 `"|"`符号的解释

`"|"` 是` __or__` ([[魔术方法]]) 实现的 ->

```python
@override
    def __or__(

        self,

        other: Runnable[Any, Other]

        | Callable[[Iterator[Any]], Iterator[Other]]

        | Callable[[AsyncIterator[Any]], AsyncIterator[Other]]

        | Callable[[Any], Other]

        | Mapping[str, Runnable[Any, Other] | Callable[[Any], Other] | Any],

    ) -> RunnableSerializable[Input, Other]:

        if isinstance(other, RunnableSequence):

            return RunnableSequence(

                self.first,

                *self.middle,

                self.last,

                other.first,

                *other.middle,

                other.last,

                name=self.name or other.name,

            )

        return RunnableSequence(

            self.first,

            *self.middle,

            self.last,

            coerce_to_runnable(other),

            name=self.name,

        )
```

返回一个 [[RunnableSequence]]对象, docstring也写了:

```
    A `RunnableSequence` can be instantiated directly or more commonly by using the

    `|` operator where either the left or right operands (or both) must be a

    `Runnable`.
```

 where either the left or right operands (or both) must be a **Runnable:** [[RunnableLambda]] 将函数wrap成runnable<- 使用 `@chain` 装饰器

Batch会按顺序的运行

```
    Batching is implemented by invoking the batch method on each component of the

    `RunnableSequence` in order.
```

基类的 `batch` **并不是**简单的 `for` 循环调用 `invoke`。为了提高 IO 密集型任务（如调 LLM API）的效率，它默认使用了 `ThreadPoolExecutor`（线程池）。Batch会根据 `config` 中的 `max_concurrency` 参数决定开启多少个线程