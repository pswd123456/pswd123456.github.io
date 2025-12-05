---
date created: 12月4日 , 21:51 , 2025
date modified: 12月5日 , 15:36 , 2025
---

```python
def invoke(
        self, input: Input, config: RunnableConfig | None = None, **kwargs: Any
    ) -> Output:
        # ... (省略了一些配置初始化的代码) ...
        
        # input_ 就像接力棒，一开始是用户传进来的 input
        input_ = input

        # invoke all steps in sequence
        try:
            # 遍历所有的步骤 (steps 就是你用 | 连起来的那些组件)
            for i, step in enumerate(self.steps):
                # ... (省略了为每一步创建子配置的代码) ...
                
                # 关键点来了！
                if i == 0:
                    # 第一步：吃进用户的 input，吐出结果给 input_
                    input_ = context.run(step.invoke, input_, config, **kwargs)
                else:
                    # 后续步骤：吃进上一步的 input_，再吐出新的结果给 input_
                    input_ = context.run(step.invoke, input_, config)
                    
        # ... (省略异常处理) ...
        
        # 最后，把最后一棒的结果作为整个链的输出返回
        return cast("Output", input_)
```

## 🏃‍♂️ 数据流动的过程

这就好比一场接力赛：

1. **初始化**：变量 `input_` 拿着第一棒（用户的输入）。
    
2. **循环 (`for` loop)**：
    
    - **第一棒** (`PromptTemplate`)：拿到 `input_` ("讲个笑话") -> 变成 `PromptValue` -> 更新 `input_`。
        
    - **第二棒** (`ChatModel`)：拿到 `input_` (`PromptValue`) -> 生成 `AIMessage` -> 更新 `input_`。
        
    - **第三棒** (`StrOutputParser`)：拿到 `input_` (`AIMessage`) -> 提取字符串文本 -> 更新 `input_`。
        
3. **终点**：循环结束，返回最后的 `input_`