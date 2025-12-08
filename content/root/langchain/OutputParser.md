---
date created: 12月5日 , 10:46 , 2025
date modified: 12月5日 , 22:48 , 2025
---

## OutputParser

LLM返回的会是一个 `AImessage` 对象, 需要将它转化回 `str`

```python
class StrOutputParser(BaseTransformOutputParser[str]):
    # ...
    @override
    def parse(self, text: str) -> str:
        """Returns the input text with no changes."""
        return text
```

- LLM 输出的是 `AIMessage(content="你好")`。
    
- `StrOutputParser` 会自动提取其中的 `.content` 属性。
    
- 然后调用这里的 `parse` 方法，把提取出来的 `"你好"` 字符串返回给你