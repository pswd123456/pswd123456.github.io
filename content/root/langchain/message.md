---
date created: 12月5日 , 10:37 , 2025
date modified: 12月5日 , 11:28 , 2025
---

## (1) `SystemMessage` (系统消息)

- **定义**：幕后导演的指令。
    
- **作用**：设定 AI 的人设、行为准则、回答风格。它通常不在对话流中显示给用户，但对 AI 的影响最大。
    
- **格式**：`role="system", content="..."`
    

## (2) `HumanMessage` (人类消息)

- **定义**：用户的输入。
    
- **作用**：代表提问者、操作者。AI 需要针对这些消息进行回复。
    
- **格式**：`role="user", content="..."`
    

## (3) `AIMessage` (AI 消息)

- **定义**：AI 的输出。
    
- **作用**：代表模型生成的回复。在“历史记录”中，它帮助模型“回忆”起自己刚才说了什么，从而保持上下文连贯。
    
- **格式**：`role="assistant", content="..."`
    

## (4) `ChatMessage` (通用消息 - 较少用)

- **定义**：自定义角色的消息。
    
- **作用**：当你需要这就叫 "Tool" 或者其他自定义角色时使用。
    
- **代码中的体现**： 在 `prompts/chat.py` 中，`ChatMessagePromptTemplate` 就允许你手动指定 `role`。

## 如果传入list of str?

```python
def _convert_to_message_template(message: MessageLikeRepresentation, ...):
    # ...
    # 如果你只传了一个字符串 (比如 "今天天气如何")
    elif isinstance(message, str):
        # 它默认把你当成 "human" (用户)
        message_ = _create_template_from_message_type("human", message, ...)
    # ...
    # 如果你传了元组 ("system", "你是AI")
    elif isinstance(message, (tuple, dict)):
        # 它会根据第一个元素 "system" 创建 SystemMessage
        # ...
```
