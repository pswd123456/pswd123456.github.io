---
date created: 12月11日 , 22:35 , 2025
date modified: 12月11日 , 22:56 , 2025
---

这是 Ragas 源码解读的第三部分：**LLM 配置与适配 (The Engine)**。

Ragas 的核心是一个“基于 LLM 的评估系统”。这部分源码解决了最关键的实操问题：**“如果我不用 OpenAI，怎么把我的模型（DeepSeek, 通义千问, 本地模型等）塞进去？”**

我们将重点解读 `ragas/llms/base.py`。

---

## 1. 统一接口：`BaseRagasLLM`

所有的 LLM 在 Ragas 内部都被视为 `BaseRagasLLM` 的子类。

```Python
# 位于 ragas/llms/base.py

@dataclass
class BaseRagasLLM(ABC):
    run_config: RunConfig = field(default_factory=RunConfig, repr=False)

    @abstractmethod
    def generate_text(self, prompt: PromptValue, n: int = 1, ...) -> LLMResult: ...

    @abstractmethod
    async def agenerate_text(self, prompt: PromptValue, n: int = 1, ...) -> LLMResult: ...
```

**源码解读**：

- **核心方法**：`generate_text` 和 `agenerate_text`。无论底层是哪个库，Ragas 的指标只会调用这两个方法。
    
- **参数 `n`**：这是为了支持“一次生成多个样本”（用于 `AnswerRelevancy` 等指标计算平均分）。如果你自定义的 LLM 不支持 `n>1`，Ragas 内部会通过简单的循环来模拟（参考 `LangchainLLMWrapper` 中的实现）。

---

## 2. 桥梁：`LangchainLLMWrapper` (最常用的适配器)

绝大多数用户通过 LangChain 连接模型。这个类负责把 LangChain 的 `BaseLanguageModel` 伪装成 Ragas 需要的样子。

```Python
# 位于 ragas/llms/base.py

class LangchainLLMWrapper(BaseRagasLLM):
    def __init__(self, langchain_llm: BaseLanguageModel, ...):
        self.langchain_llm = langchain_llm
        # ...

    def generate_text(self, prompt: PromptValue, n: int = 1, ...) -> LLMResult:
        # ...
        if is_multiple_completion_supported(self.langchain_llm):
            # 如果原生支持 n (如 OpenAI)，直接透传
            result = self.langchain_llm.generate_prompt(prompts=[prompt], n=n, ...)
        else:
            # 否则，把同一个 prompt 复制 n 遍发给 LLM
            result = self.langchain_llm.generate_prompt(prompts=[prompt] * n, ...)
            # ...
        return result
```

*注: 即使你使用了非OpenAI模型, 不支持n>1的模型, 比如deepseek, 仍然不会自动降级, 因为是维护了一个白名单, deepseek不在白名单上, 

```python
# 位于 ragas/llms/base.py

# 1. 这里定义了一个白名单
MULTIPLE_COMPLETION_SUPPORTED = [
    OpenAI,
    ChatOpenAI,
    AzureOpenAI,
    AzureChatOpenAI,
    ChatVertexAI,
    VertexAI,
]

# 2. 这个函数去检查你的 LLM 是否在白名单里
def is_multiple_completion_supported(llm: BaseLanguageModel) -> bool:
    """Return whether the given LLM supports n-completion."""
    for llm_type in MULTIPLE_COMPLETION_SUPPORTED:
        if isinstance(llm, llm_type):
            return True
    return False
```

*要改只能这么改:*

```python
from ragas.llms.base import MULTIPLE_COMPLETION_SUPPORTED
from my_custom_llm import MySuperFastLLM

# 强行把你的类加进白名单
MULTIPLE_COMPLETION_SUPPORTED.append(MySuperFastLLM)

# 然后再初始化 Wrapper
ragas_llm = LangchainLLMWrapper(MySuperFastLLM())
```

*或者用替代指标 比如用nv_accuracy 代替 answer_relevancy 这种需要n>1的指标*

**使用建议**： 如果你使用非 OpenAI 模型（例如通过 LangChain 的 `ChatOllama` 或 `ChatHuggingFace`），你只需要这样做：

```Python

from langchain_community.chat_models import ChatOllama
from ragas.llms import LangchainLLMWrapper

# 1. 初始化 LangChain 模型
my_lc_model = ChatOllama(model="llama3")
# 2. 包装成 Ragas 模型
ragas_llm = LangchainLLMWrapper(my_lc_model)
# 3. 传给 evaluate
evaluate(..., llm=ragas_llm)
```

这样你就绕过了 Ragas 默认的 OpenAI 依赖。

*注: 如果使用这个会跳一个即将deprecated的warning, 建议用llm_factory

*而且不知道是不是因为LangchainLLMWrapper不输出json的原因, 
**Qwen**系列(flash, plus, max)作为judge的时候有些指标会出现模型缺陷式的错误, 要么是无限重复自己, 要么是什么都不返回, 两者都会失败然后报错, 表现为分数是0

---

*需要确保provider已经支持, 至少在我用的时候还不支持dashscope和deepseek*

## 3. 现代化工厂：`llm_factory`

在 Ragas 2.0+ 中，官方更推荐使用工厂模式来创建模型，因为它能自动处理**结构化输出 (Structured Output)** 的适配。

```Python
# 位于 ragas/llms/base.py

def llm_factory(
    model: str,
    provider: str = "openai",
    client: t.Optional[t.Any] = None,
    adapter: str = "auto",
    # ...
) -> InstructorBaseRagasLLM:
    # ...
    # 自动探测最佳适配器 (Instructor 或 LiteLLM)
    if adapter == "auto":
        from ragas.llms.adapters import auto_detect_adapter
        adapter = auto_detect_adapter(client, provider_lower)
    
    # ...
    return llm
```

**源码解读**： Ragas 为了保证评估结果（通常是 JSON）的稳定性，重度使用了 `instructor` 库来强制 LLM 输出结构化数据。

- 如果你用 `llm_factory("gpt-4", client=openai_client)`，它会返回一个经过 `Instructor` 增强的模型。
    
- **为什么这很重要？** 很多指标（如 `Faithfulness`）需要 LLM 输出具体的 statements 和 verdict（true/false）。如果不用这种结构化生成，开源小模型经常输出乱码 JSON，导致指标计算失败（NaN）。

如果 `llm_factory` 不支持你的 Provider（比如某些国内的小众模型服务，或者你自己的私有 API），解决方案如下：

借力打力 —— 使用 `adapter="litellm"`

`llm_factory` 的默认行为是 `adapter="auto"`，它可能只认识主流厂商。但 Ragas 内部集成了 **LiteLLM** 适配器，而 LiteLLM 支持 100+ 种 Provider。

如果你的 Provider 被 LiteLLM 支持（大多数都支持），你可以强制指定 adapter：

```Python

from ragas.llms import llm_factory

# 假设你的 provider 是 "deepseek" (如果它被 LiteLLM 支持)
# 或者你可以直接用 LiteLLM 的 OpenAI 兼容模式连接任意 provider
from litellm import OpenAI as LiteLLMClient

# 初始化 LiteLLM 客户端 (它可以连接任何 OpenAI 兼容的 API)
client = LiteLLMClient(
    api_key="your-api-key", 
    base_url="https://api.your-provider.com/v1",
    model="your-model-name"
)

# 强制指定 adapter="litellm"
llm = llm_factory(
    model="your-model-name",
    client=client,
    adapter="litellm"  # <--- 关键在这里
)
```

**原理**：查看 `ragas/llms/base.py` 的源码，`llm_factory` 会动态加载 adapter。只要 `litellm` 能处理，Ragas 就能用。

*注: 没有验证过的解决方案*

---

## 4. 结构化输出的核心：`InstructorLLM`

这是 `llm_factory` 生产出来的产品，也是 Ragas 2.0 能够稳定运行的关键。

```Python
# 位于 ragas/llms/base.py

class InstructorLLM(InstructorBaseRagasLLM):
    def generate(self, prompt: str, response_model: t.Type[InstructorTypeVar]) -> InstructorTypeVar:
        # ...
        # 调用 client.chat.completions.create 并强制要求 response_model
        result = self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            response_model=response_model, # 关键：强制返回 Pydantic 对象
            **provider_kwargs,
        )
        return result
```

**使用建议**： 如果你在使用能力较弱的模型（如 7B 以下的开源模型），**强烈建议**不要直接用 `LangchainLLMWrapper`，而是尝试配置支持工具调用或 JSON Mode 的 `llm_factory` 方式，或者确保你的 LangChain 对象配置了良好的 Prompt 来引导 JSON 输出。Ragas 现在的趋势是强依赖 Pydantic Schema 来做输出解析。

---

## 5. 异常处理：`is_finished`

有时候模型因为 `max_tokens` 设置太小，话说了一半就被截断了。Ragas 在 `LangchainLLMWrapper` 里做了检查。

```Python
# 位于 ragas/llms/base.py (LangchainLLMWrapper内)

    def is_finished(self, response: LLMResult) -> bool:
        # ...
        # 检查 finish_reason 是否为 "stop", "STOP" 等正常结束标志
        # 如果是 "length" (也就是超长截断)，这里会返回 False
        return all(is_finished_list)
```

并在 `generate` 方法中抛出异常：

```Python
        # check there are no max_token issues
        if not self.is_finished(result):
            raise LLMDidNotFinishException()
```

**使用建议**： 如果你频繁遇到 `LLMDidNotFinishException`，这不是 Ragas 的 bug，而是你需要增加你的 LLM 配置里的 `max_tokens` (或 `max_output_tokens`) 参数。

*注: 使用可观测工具查看报错的trace到底输出了什么, 有时候不一定是max_token不够, 而是有的模型抽风了无限的重复自己*

---

## 总结：第三部分核心知识点

1. **Wrapper 模式**：要把任意 LangChain 模型给 Ragas 用，请使用 `LangchainLLMWrapper(your_model)`。
    
2. **Factory 模式**：`llm_factory` 是为了更好地支持结构化输出（JSON），这对指标计算的成功率至关重要。
    
3. **参数 n 的陷阱**：如果你发现某些指标跑得特别慢，可能是因为你的模型不支持 `n` 参数（并发生成），Ragas 在本地做了串行循环。
    
4. **截断报错**：遇到 `LLMDidNotFinishException` 时，去调大你的模型生成的 token 限制。