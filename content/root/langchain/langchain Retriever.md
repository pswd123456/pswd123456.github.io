---
date created: 12月4日 , 19:47 , 2025
date modified: 12月4日 , 20:20 , 2025
---

## 检索器

### 向量

[[langchain vector store]]

#### `BaseRetriever`

```python

@override

    def __init_subclass__(cls, **kwargs: Any) -> None:

        super().__init_subclass__(**kwargs)

        parameters = signature(cls._get_relevant_documents).parameters
        
    """
    When implementing a custom retriever, the class should implement the

    `_get_relevant_documents` method to define the logic for retrieving documents.
    """
        
@override

    def invoke(

        self, input: str, config: RunnableConfig | None = None, **kwargs: Any

    ) -> list[Document]:

        """Invoke the retriever to get relevant documents.

        Main entry point for synchronous retriever invocations.

        Args:

            input: The query string.

            config: Configuration for the retriever.

            **kwargs: Additional arguments to pass to the retriever.

        Returns:

            List of relevant documents.

        Examples:

        ```python

        retriever.invoke("query")

        ```

        """
        
        ...
        
        result = self._get_relevant_documents(input, **kwargs_)
        
        return result
        
```

可以看到, `BaseRetriever`规定了customRetriever的子类必须有_get_relevant_documents这个方法

例如->

#### `VectorStoreRetriever` 

```python
 def _get_relevant_documents(

        self, query: str, *, run_manager: CallbackManagerForRetrieverRun, **kwargs: Any

    ) -> list[Document]:

        kwargs_ = self.search_kwargs | kwargs

        if self.search_type == "similarity":

            docs = self.vectorstore.similarity_search(query, **kwargs_)
            
            .....
            
            return docs
```

将 [[langchain vector store]] 提供的相似度搜索返回docs