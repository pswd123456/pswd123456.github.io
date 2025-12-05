---
date created: 12月5日 , 9:50 , 2025
date modified: 12月5日 , 10:48 , 2025
---

## ChatPromptTemplate

```python
def format_messages(self, **kwargs: Any) -> list[BaseMessage]:

        """Format the chat template into a list of finalized messages.
        
        Args:

            **kwargs: keyword arguments to use for filling in template variables

                in all the template messages in this chat template
        Raises:

            ValueError: if messages are of unexpected types.

        Returns:

            list of formatted messages.

        """

        kwargs = self._merge_partial_and_user_variables(**kwargs)

        result = []

        for message_template in self.messages:

            if isinstance(message_template, BaseMessage):

                result.extend([message_template])

            elif isinstance(

                message_template, (BaseMessagePromptTemplate, BaseChatPromptTemplate)

            ):

                message = message_template.format_messages(**kwargs)

                result.extend(message)

            else:

                msg = f"Unexpected input: {message_template}"

                raise ValueError(msg)  # noqa: TRY004

        return result
```

总之把所有[[message]]和`message_template`拼到`result`里, 如果是`MessagesPlaceholder` 也先拼起来等待后续传入->

### MessagesPlaceholder

```python
def format_messages(self, **kwargs: Any) -> list[BaseMessage]:
        # ... 
        value = (
            kwargs.get(self.variable_name, []) # 情况 A: 如果 optional=True
            if self.optional
            else kwargs[self.variable_name]    # 情况 B: 如果 optional=False (默认)
        )
        # ...
        if not isinstance(value, list):
            msg = (
                f"variable {self.variable_name} should be a list of base messages, "
                f"got {value} of type {type(value)}"
            )
            raise ValueError(msg)
            #必须是一个列表
```

比如 `MessagesPlaceholder(variable_name="history")` 

```python
# 定义占位符 
prompt = ChatPromptTemplate.from_messages([ ("system", "你是AI"), MessagesPlaceholder(variable_name="history"), # <--- 占位 
("human", "{question}") ])
```

`ChatPromptTemplate`

- 会先把它拼进result, 
- 插在 system和human的中间, 
- 等待传入的字典中有一个key叫 `"history"`

等待调用时:

```python
prompt.invoke({ 
	"history": ["must be a list", AImessage(content="AI message")], # 填空 
	"question": "今天天气如何" })
```

`prompt`:

```python
[
	SystemMessage(content="你是AI"),
	HumanMessage(content="must be a list"), #<-- history被展开了, 为什么是HumanMessage? 参见: message#如果传入list of str?]]
	AImessage(content="AI message"),
	HumanMessage(content="今天天气如何")
]
```

- *[[message#如果传入list of str?]]

### 如何在LCEL中传入MessagePlaceholder?

使用 [[RunnablePassthrough]]
