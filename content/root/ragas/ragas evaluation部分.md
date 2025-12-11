---
date created: 12月11日 , 21:58 , 2025
date modified: 12月12日 , 0:13 , 2025
---

## 第一阶段：[[核心入口与数据契约]] (Getting Started)

这是最基础的部分，解决“我需要传什么格式的数据给它”以及“它是怎么跑起来的”这两个问题。

**1. 阅读文件：`ragas/dataset_schema`.py**

- **关注点**：
    
    - `SingleTurnSample` 类：这是 Ragas 2.0+ 核心的数据单元。你需要知道字段 `user_input`, `response`, `retrieved_contexts`, `reference` 分别代表什么。**这是正确构造测试集的关键**。
        
    - `EvaluationDataset` 类：了解 `from_list` 方法，这是将你的数据转换为 Ragas 内部格式的主要途径。
        
    - `EvaluationResult` 类：了解评估结束后你会得到什么（`scores` 和 `to_pandas()` 方法）。
        
- **为了使用**：确保你的输入数据（List of Dicts）的 key 能对应上 `SingleTurnSample` 的字段，否则会在校验阶段报错。
    

**2. 阅读文件：`ragas/evaluation.py`**

- **关注点**：
    
    - `evaluate` 函数：这是整个库的入口。
        
    - 参数列表：重点看 `dataset`, `metrics`, `llm`, `run_config`, `column_map`。
        
    - 逻辑流程：虽然你可以跳过复杂的异步逻辑，但要看它是如何调用 `convert_v1_to_v2_dataset` 和 `validate_required_columns` 的。这能帮你理解为什么有时候会报“缺少列”的错误。
        
- **为了使用**：理解 `evaluate` 是如何将 Dataset、Metrics 和 LLM 组装在一起的。

---

## 第二阶段：[[评估指标的运作机制]] (Metrics Logic)

为了避免“指标跑出来了但不知道它是怎么算的”或者“为什么这个指标报错说缺列”，你需要理解指标的基类。

**3. 阅读文件：`ragas/metrics/base.py`**

- **关注点**：
    
    - `Metric` 类：重点看 `required_columns` 属性。**这是调试 Ragas 最重要的地方**。如果你使用的指标需要 `reference` 但你没提供，这里会告诉你。
        
    - `SingleTurnMetric` vs `MultiTurnMetric`：了解你的场景是单轮对话还是多轮对话，选择对应的指标类型。
        
    - `MetricWithLLM`：了解大多数指标（如 Faithfulness, Answer Relevancy）都需要绑定一个 `llm` 对象。
        
- **为了使用**：当你自定义指标或者调试现成指标时，检查 `required_columns` 是第一步。

关于我在项目中使用的metrics: [[ragas metrics]]

---

## 第三阶段：[[LLM 配置与适配]] (The Engine)

Ragas 极度依赖 LLM 进行打分。如果你不想用默认的 OpenAI，或者你的网络环境特殊，这部分源码必须读。

**4. 阅读文件：`ragas/llms/base.py`**

- **关注点**：
    
    - `BaseRagasLLM`：所有 Ragas LLM 的基类。
        
    - `LangchainLLMWrapper`：**非常重要**。如果你已经在用 LangChain，你会经常通过这个包装器把你的 LangChain LLM 传给 Ragas。看它是如何处理 `generate_text` 的。
        
    - `llm_factory` 函数：Ragas 推荐的初始化 LLM 的方式。看它是如何根据 `provider`（如 openai, bedrock, google）自动选择适配器的。
        
- **为了使用**：学会如何将自定义的 LLM（例如通过 vLLM 部署的开源模型，或者 Azure OpenAI）正确封装传入 `evaluate` 函数，避免出现 `LLM provider not supported` 错误。

---

## 第四阶段：[[生产环境稳定性]] (Production Readiness)

在 Notebook 里跑 Demo 和在生产管线里跑评估是不一样的。这一阶段关注如何处理超时、重试和并发。

**5. 阅读文件：`ragas/run_config.py`**

- **关注点**：
    
    - `RunConfig` 数据类：关注 `timeout`, `max_retries`, `max_wait`, `max_workers`。
        
- **为了使用**：当你的评估任务因为 Rate Limit 失败，或者模型响应太慢导致 Timeout 时，你需要实例化一个 `RunConfig` 对象并传给 `evaluate` 函数，而不是去改库的源码。

## Prompt设计

Ragas 2.0 的 Evaluator Prompt 设计采用了**结构化、面向对象**的范式，与早期版本（或常见的纯字符串 Prompt）有显著不同。

这种设计的核心目标是**让 LLM 的输出可控且易于解析**，从而保证评估指标（Metrics）计算的稳定性。

以下从 **核心架构**、**Prompt 构成** 和 **实际案例** 三个维度进行深度解析。

### 1. 核心架构：Pydantic 为先

在 Ragas 2.0 中，几乎所有的核心 Prompt 都继承自 `PydanticPrompt`，而不是简单的字符串模板。

- **强类型契约**：Ragas 强制定义 `InputModel` 和 `OutputModel`。这意味着 LLM 不是“生成一段文本”，而是“完成一个函数调用”或“填充一个 JSON 对象”。
    
- **PromptMixin 机制**：所有的评估指标（Metric）都继承自 `PromptMixin`。这使得每个指标都可以通过 `.get_prompts()` 和 `.set_prompts()` 方法来管理、保存或替换自己的 Prompt，实现了 Prompt 与代码逻辑的解耦。
    

### 2. Prompt 的解剖学 (Anatomy of a Prompt)

一个典型的 Ragas Evaluator Prompt 包含以下四个关键部分：

1. **Instruction (指令)**：一段清晰的自然语言，定义任务目标。
    
    - _例如：“给定一个问题和上下文，判断上下文是否包含回答问题所需的信息。”_
        
2. **Input Model (输入模型)**：基于 Pydantic 的类，定义“传给 LLM 的数据结构”。
    
    - 通常包含 `question`, `answer`, `context` 等字段。
        
3. **Output Model (输出模型)**：基于 Pydantic 的类，定义“期望 LLM 返回的数据结构”。
    
    - 这是设计的精髓。它通常包含 `reason` (推理过程) 和 `verdict` (结论，如 0/1 或分数)。
        
4. **Few-shot Examples (少样本示例)**：一组 `(InputModel, OutputModel)` 的配对。
    
    - Ragas 会将这些示例自动序列化为 JSON 格式塞入 Prompt 中，通过 In-context Learning 教会 LLM 如何输出正确的格式。
        

### 3. 实例解析：Faithfulness (忠实度) 的设计

以 `Faithfulness` 指标为例，它的计算过程被拆解为两个步骤，每个步骤都有独立的 Prompt 设计：

#### 第一步：语句拆解 (Statement Generation)

- **目标**：把长回答拆解为一个个独立的原子事实（Claims）。
    
- **Input**：`question`, `answer`
    
- **Output**：`statements` (List[str])
    
- **设计亮点**：如果不做这一步，直接让 LLM 评分，它容易“胡言乱语”。拆解后，评估粒度变得极细。
    

#### 第二步：NLI 判断 (Natural Language Inference)

- **目标**：判断每个原子事实是否能从 Context 中推导出来。
    
- **Input**：`context`, `statements`
    
- **Output**：`verdict` (0 或 1), `reason`
    
- **代码示意 (伪代码)**：

```Python
# 定义输入结构
class NLIStatementInput(BaseModel):
    context: str
    statements: List[str]

# 定义输出结构（关键设计）
class StatementFaithfulnessAnswer(BaseModel):
    statement: str
    verdict: int  # 0 或 1
    reason: str   # 强制 LLM 生成理由，防止瞎猜

class NLIStatementOutput(BaseModel):
    statements: List[StatementFaithfulnessAnswer]

# 组合成 Prompt
class FaithfulnessPrompt(PydanticPrompt):
    instruction = "Your task is to judge the faithfulness..."
    input_model = NLIStatementInput
    output_model = NLIStatementOutput
    examples = [ ... ] # 高质量的少样本示例
```

### 4. 这种设计的好处

1. **消除解析错误**：因为强制要求 JSON 输出并用 Pydantic 校验，几乎消除了“Output Parser Error”。
    
2. **可解释性 (Explainability)**：通过在 `OutputModel` 中强制包含 `reason` 字段，不仅得到了分数，还得到了“扣分理由”。
    
3. **易于适配**：如果你想换成中文评估，只需要调用 `.adapt(language="chinese", llm=...)`，Ragas 会自动翻译指令和示例，但保持 JSON 结构不变。
    

### 总结

Ragas 的 Evaluator Prompt 设计不再是“写小作文”，而是**定义 API 接口**。它把 Prompt Engineering 变成了软件工程的一部分，通过 `Instruction` + `Pydantic Schema` + `Few-shot JSONs` 的组合，确保了在非 OpenAI 模型（如 Llama 3, Mistral）上也能获得相对稳定的评估效果。

### 示例

基于 `ragas/metrics/_faithfulness.py` 的源码，Faithfulness 指标的计算包含**两个独立的 Prompt 交互步骤**。

这是一个完整的 Faithfulness Prompt 示例，包含了**第一步：语句拆解 (Statement Generation)** 和 **第二步：NLI 判决 (NLI Verdict)** 的实际输入输出结构。

### 1. 第一步：语句拆解 (Statement Generation)

这一步的目标是将 LLM 生成的长文本回答拆解为独立的原子事实（Statements）。

**Input Model (输入给 LLM 的 JSON):**

```JSON
{
  "question": "What is the capital of France and what is it famous for?",
  "answer": "The capital of France is Paris. It is famous for the Eiffel Tower and the Louvre Museum."
}
```

**Prompt (Ragas 发送给 LLM 的指令):**

> **Instruction:** Given a question and an answer, analyze the complexity of each sentence in the answer. Break down each sentence into one or more fully understandable statements. Ensure that no pronouns are used in any statement. Format the outputs in J1SON.
> 
> Examples:
> 
> Input: {"question": "Who was Albert Einstein...?", "answer": "He was a German-born..."}
> 
> Output: {"statements": ["Albert Einstein was a German-born theoretical physicist.", ...]}
> 
> Task:
> 
> {"question": "What is the capital of France and what is it famous for?", "answer": "The capital of France is Paris. It is famous for the Eiffel Tower and the Louvre Museum."}

**Output Model (期望 LLM 返回的 JSON):**

```JSON
{
  "statements": [
    "The capital of France is Paris.",
    "Paris is famous for the Eiffel Tower.",
    "Paris is famous for the Louvre Museum."
  ]
}
```

---

### 2. 第二步：NLI 判决 (NLI Verdict)

这一步的目标是根据检索到的上下文（Context），判断上一步生成的每个原子事实是否属实。

**Input Model (输入给 LLM 的 JSON):**

```JSON
{
  "context": "Paris is the capital and most populous city of France. The Eiffel Tower is a wrought-iron lattice tower on the Champ de Mars in Paris.",
  "statements": [
    "The capital of France is Paris.",
    "Paris is famous for the Eiffel Tower.",
    "Paris is famous for the Louvre Museum."
  ]
}
```

**Prompt (Ragas 发送给 LLM 的指令):**

> **Instruction:** Your task is to judge the faithfulness of a series of statements based on a given context. For each statement you must return verdict as 1 if the statement can be directly inferred based on the context or 0 if the statement can not be directly inferred base2d on the context.
> 
> Examples:
> 
> Input: {"context": "John is a student...", "statements": ["John is majoring in Biology.", ...]}
> 
> Output: {"statements": [{"statement": "John is majoring in Biology.", "reason": "...", "verdict": 0}, ...]}
> 
> Task:
> 
> {"context": "Paris is the capital and most populous city of France. The Eiffel Tower is a wrought-iron lattice tower on the Champ de Mars in Paris.", "statements": ["The capital of France is Paris.", "Paris is famous for the Eiffel Tower.", "Paris is famous for the Louvre Museum."]}

**Output Model (期望 LLM 返回的 JSON):**

```JSON


{
  "statements": [
    {
      "statement": "The capital of France is Paris.",
      "reason": "The context explicitly states that Paris is the capital of France.",
      "verdict": 1
    },
    {
      "statement": "Paris is famous for the Eiffel Tower.",
      "reason": "The context mentions the Eiffel Tower is in Paris, implying its association, though 'famous' is subjective, it is inferable from the context description.",
      "verdict": 1
    },
    {
      "statement": "Paris is famous for the Louvre Museum.",
      "reason": "The provided context mentions the Eiffel Tower but does NOT mention the Louvre Museum. Therefore, this statement cannot be inferred from the context.",
      "verdict": 0
    }
  ]
}
```

### 最终分数计算

$$Faithfulness = \frac{\text{Number of Verdict 1}}{\text{Total Statements}} = \frac{2}{3} \approx 0.67$$

### 源码出处

- Prompt 定义见 `ragas/metrics/_faithfulness.py` 中的 `StatementGeneratorPrompt` 和 `NLIStatementPrompt` 类。
    
- 计算逻辑见 `Faithfulness._compute_score` 方法。