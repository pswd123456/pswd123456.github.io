---
date created: 12月4日 , 19:47 , 2025
date modified: 12月5日 , 22:48 , 2025
---

在`base`类中定义了`chunk_size`和`chunk_overlap`, 这通常也是我们从`RecursiveCharacterTextSplitter`开始时最关心的两个参数, 

`_merge_splits`负责将切碎的小块粘合至快要超过`chunk_size`为止

## `RecursiveCharacterTextSplitter`

```python
def _split_text(self, text: str, separators: list[str]) -> list[str]:
        """Split incoming text and return chunks."""
        final_chunks = []
        # Get appropriate separator to use
        separator = separators[-1]
        new_separators = []
        
        # ... (寻找当前最合适的分隔符) ...
        
        # 使用当前找到的分隔符切分文本
        splits = _split_text_with_regex(text, separator_, keep_separator=self._keep_separator)

        # Now go merging things, recursively splitting longer texts.
        # 下面是关键逻辑：
        good_splits = []
        for s in splits: 
            if self._length_function(s) < self._chunk_size:
                # 如果切出来的块够小，先攒着（后面会合并）
                good_splits.append(s)
            else:
                # 如果这个块还是太大！
                if good_splits:
                    # 先把之前攒好的小块合并并存好
                    merged_text = self._merge_splits(good_splits, separator_)
                    final_chunks.extend(merged_text)
                    good_splits = []
                if not new_separators:
                    final_chunks.append(s)
                else:
                    # 关键点：递归调用自己！
                    # 使用剩下的分隔符（new_separators）继续切分这个过大的块
                    other_info = self._split_text(s, new_separators)
                    final_chunks.extend(other_info)
        # ...
        return final_chunks
```

会遍历 `separators` 直到 `' '`或者这个单词太大只能切断为止
