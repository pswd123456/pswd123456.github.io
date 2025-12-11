---
date created: 12月11日 , 23:0 , 2025
date modified: 12月12日 , 0:15 , 2025
---

本路线将聚焦于核心类 `TestsetGenerator` 的接口调用流程。

如需查看evaluator的工作内容 -> [[ragas evaluation部分]]

## 核心路线概览

1. **初始化 (Initialization)**: 配置 LLM 和 Embedding 模型，构建生成器实例。
    
2. **数据准备 (Data Preparation)**: 准备您的文档数据（支持 LangChain 或 LlamaIndex 格式）。
    
3. **执行生成 (Execution)**: 调用高层 API 生成测试集，理解关键参数（`testset_size` 等）。
    
4. **结果导出 (Export)**: 处理返回的 `Testset` 对象，转换为可用数据。

---

## 第一步：[[初始化生成器]] (`TestsetGenerator`)

在 `ragas/testset/synthesizers/generate.py` 中，`TestsetGenerator` 是所有操作的入口。作为使用者，您不需要手动实例化复杂的 `KnowledgeGraph`，而是应该使用类方法（Class Methods）来快速初始化。

- **关键代码位置**: `ragas/testset/synthesizers/generate.py` 中的 `from_langchain` 和 `from_llama_index` 方法。
    
- **如何使用**: Ragas 依赖于 LLM 来生成问题，依赖 Embedding Model 来计算相似度构建知识图谱。

    ```Python
    from ragas.testset.synthesizers.generate import TestsetGenerator
    from langchain_openai import ChatOpenAI, OpenAIEmbeddings
    
    # 1. 准备您的模型 (以 LangChain 为例)
    generator_llm = ChatOpenAI(model="gpt-4")
    generator_embeddings = OpenAIEmbeddings()
    
    # 2. 初始化生成器
    # 使用 from_langchain 自动封装模型
    generator = TestsetGenerator.from_langchain(
        llm=generator_llm,
        embedding_model=generator_embeddings
    )
    ```

## 第二步：[[准备文档数据]]

Ragas 需要原始文档作为“灵感来源”来生成问答对。源码显示它主要支持 LangChain 的 `Document` 和 LlamaIndex 的 `Document`。

- **关键代码位置**: `generate_with_langchain_docs` 或 `generate_with_llamaindex_docs` 的 `documents` 参数类型提示。
    
- **如何使用**: 您只需将数据加载为 Document 对象列表。
    

    Python

    ```
    from langchain_core.documents import Document
    
    docs = [
        Document(page_content="Ragas 是一个用于评估 RAG 管道的框架...", metadata={"source": "docs"}),
        # ... 更多文档
    ]
    ```

## 第三步：[[生成测试集]] (核心步骤)

这是最重要的一步。您不需要手动处理 transforms（转换）或 query_distribution（查询分布），因为源码中已经提供了默认值。

- **关键代码位置**: `TestsetGenerator.generate_with_langchain_docs` 方法。
    
- **关键参数解析**:
    
    - `documents`: 上一步准备的文档列表。
        
    - `testset_size`: 您希望生成的测试集样本数量（例如 10 个）。
        
    - `transforms`: (可选) 源码中的 `default_transforms` 会自动处理文档切分、关键词提取和节点构建。作为使用者，通常**不需要**修改此项，除非默认的文档切分效果不佳。
        
    - `query_distribution`: (可选) 控制生成问题的类型（如简单问答、多跳推理等）。默认配置通常已足够覆盖多种场景。
        
    - `raise_exceptions`: 建议设为 `False`，这样某一个样本生成失败不会导致整个程序崩溃。
        
- **代码示例**:

    ```Python
    
    # 生成包含 10 个样本的测试集
    testset = generator.generate_with_langchain_docs(
        documents=docs,
        testset_size=10,
        raise_exceptions=False
    )
    ```

## 第四步：[[导出与使用结果]]

生成完成后，您会得到一个 `Testset` 对象。

- **关键代码位置**: `ragas/testset/synthesizers/testset_schema.py` 定义了 `Testset` 和 `TestsetSample`。
    
- **如何使用**: `Testset` 对象提供了方便的方法将数据导出为标准格式，以便后续用于评估。
    
    - `to_list()`: 将测试集转换为字典列表，方便查看或转换为 DataFrame。
        
    - `to_evaluation_dataset()`: 转换为 Ragas 评估所需的 `EvaluationDataset` 对象。

    ```   Python
    # 导出为 Pandas DataFrame 查看
    import pandas as pd
    
    # 使用 .to_list() 获取字典列表
    df = pd.DataFrame(testset.to_list())
    
    # 打印查看
    print(df.head())
    
    # df 中通常包含: 'user_input' (问题), 'reference' (答案), 'retrieved_contexts' (上下文), 'synthesizer_name' (生成类型)
    ```

## 总结：您的代码库使用路线图

1. **Import**: 引入 `TestsetGenerator`。
    
2. **Instantiate**: 使用 `from_langchain(...)` 传入 LLM 和 Embedding。
    
3. **Generate**: 调用 `generate_with_langchain_docs(docs, testset_size=N)`。
    
4. **Export**: 调用 `testset.to_list()` 转为 DataFrame 保存。
    

这一路线避开了底层的 `KnowledgeGraph` 构建、`Node` 操作和 `Transforms` 细节，直接利用了源码中为您封装好的高层 API，符合您“作为库引入使用”的目标。

## Prompt设计

Ragas 的 `TestsetGenerator` 在生成测试集时，Prompt（提示词）设计是其“智能”的核心。它的设计不是一个简单的单一大 Prompt，而是一个**分层、分阶段、结构化**的 Prompt 系统。

这套系统的核心设计理念是：**将复杂的生成任务拆解为小任务，并利用 Pydantic 强制 LLM 输出结构化数据**。

以下是 `TestsetGenerator` 中 Prompt 设计的详细解析：

### 1. 基础架构：`PydanticPrompt`

所有 Ragas 的 Prompt 都继承自 `PydanticPrompt`。

- **结构化输入/输出**: 每个 Prompt 都严格定义了 `input_model` (输入) 和 `output_model` (输出)。这确保了 LLM 不会只吐出一段文本，而是返回易于代码处理的 JSON 对象。
    
- **指令 (Instruction)**: 每个类都有明确的 `instruction` 字符串，告诉 LLM 它的角色和任务。
    
- **示例 (Examples)**: 几乎所有 Prompt 都内置了 Few-Shot（少样本）示例，以提高生成的稳定性。

---

### 2. 关键阶段的 Prompt 设计

生成流程主要分为三个阶段，每个阶段都有专门的 Prompt：

#### **阶段一：角色与主题匹配 (Persona & Theme Matching)**

在生成具体问题前，Ragas 先要决定“谁”在问，“问什么”。

- **Prompt 类**: `ThemesPersonasMatchingPrompt`
    
- **任务**: 给定一组提取出的“主题”（如 "Docker", "CI/CD"）和一组生成的“角色”（如 "DevOps 工程师"），让 LLM 连线。
    
- **设计亮点**:
    
    - 它不是随机分配，而是要求“based on their role description”（基于角色描述）。
        
    - **例子**: 输入主题 "Remote Work", "Empathy" 和角色 "HR Manager"，LLM 会智能地将它们关联起来。这保证了生成的问题符合角色身份，增加了真实感。
        

#### **阶段二：多跳概念组合 (Concept Combination)

对于复杂的推理题（Multi-hop），需要先找到能够关联的两个概念。

- **Prompt 类**: `ConceptCombinationPrompt`
    
- **任务**: 给定来自不同文档节点的概念列表，找出可以逻辑连接或对比的组合。
    
- **Instruction**: _"Identify concepts that can logically be connected or contrasted."_ (识别逻辑上可连接或对比的概念)。
    
- **设计亮点**: 强制 LLM 思考“跨文档”的关联，而不是仅关注单文档内部。这是生成推理题（如“比较 A 和 B”）的关键一步。
    

#### **阶段三：问答生成 (Query & Answer Generation)**

这是最后生成文本的步骤，分为“单跳”和“多跳”两种 Prompt，但结构相似。

**A. 单跳生成 (`SingleHopQuerySynthesizer`)**

- **Prompt 类**: `QueryAnswerGenerationPrompt`
    
- **输入 (`QueryCondition`)**: 包含 `persona` (角色), `term` (关键词), `query_style` (风格), `length` (长度), `context` (上下文)。
    
- **指令设计**:
    
    1. **Generate a Query**: 明确要求基于 Persona 和 Term 生成问题。
        
    2. **Generate an Answer**: 明确要求 **"Using only the content from the provided context"** (仅使用提供的上下文)。这是为了防止 LLM 产生幻觉（Hallucination）。
        
- **输出**: `GeneratedQueryAnswer` (包含 query 和 answer)。
    

**B. 多跳生成 (`MultiHopQuerySynthesizer`)**

- **Prompt 类**: `QueryAnswerGenerationPrompt`
    
- **输入**: 上下文不再是单一字符串，而是一个列表 `context: t.List[str]`，并且打上了标签 `<1-hop>`, `<2-hop>`。
    
- **指令设计**:
    
    - _"Ensure the query uses information from at least two segments"_ (确保问题用到至少两个片段的信息)。
        
    - 要求问题体现出“主题”（Themes）的相关性。
        
- **设计亮点**: 通过显式的 `<1-hop>` 标签，强制 LLM 在生成答案时必须进行跨段落的信息整合，而不是只看其中一段。

---

### 为什么这样设计？

1. **解耦 (Decoupling)**: 讲“思考逻辑”（如：选什么主题、谁来问）和“文本生成”（如：写具体句子）分开。
    
2. **类型安全 (Type Safety)**: 使用 Pydantic 模型，如果 LLM 返回格式不对，Ragas 会自动重试（`RagasOutputParser` 中的重试逻辑）。
    
3. **防幻觉 (Anti-Hallucination)**: 在生成答案的 Prompt 中反复强调 _"Using only the content from the provided context"_，确保生成的 Reference Answer 是可靠的 Ground Truth。
    

作为使用者，你通常不需要直接修改这些 Prompt，但了解它们的设计有助于你理解为什么生成的测试集会有特定的风格，或者在需要时通过继承这些类来定制自己的 Prompt。

### Prompt例子

为了让您彻底理解 Ragas 的 Prompt 运作机制，我将以一个**“多跳抽象查询（Multi-Hop Abstract Query）”**的生成过程为例。这是 Ragas 中流程最长、涉及 Prompt 最多的场景。

我们将模拟一个具体的例子：您上传了两篇关于“深度学习模型”的文档。

- **文档 A (Node 1)**: “Transformer 架构由 Google 在 2017 年提出，它完全基于注意力机制，抛弃了循环和卷积。”
    
- **文档 B (Node 2)**: “RNN（循环神经网络）常用于处理序列数据，但它存在梯度消失问题，且难以并行计算。”
    

以下是完整的 Prompt 流程、源码定义以及实际的数据流转示例。

---

### 阶段一：准备与匹配 (Preparation & Matching)

在生成具体问题之前，Ragas 需要先“搭桥”，把提取出的**知识点（Themes/Concepts）**和预设的**角色（Personas）**联系起来。

#### 1. 概念组合 (Concept Combination)

- **目的**: 找出不同文档中可以关联起来的概念（例如为了进行对比）。
    
- **源码位置**: `ragas/testset/synthesizers/multi_hop/prompts.py` 中的 `ConceptCombinationPrompt` 类。
    
- **Prompt 指令 (Instruction)**:
    
    > "Form combinations by pairing concepts from at least two different lists. ... Identify concepts that can logically be connected or contrasted."
    > 
    > (将来自至少两个不同列表的概念配对组合... 识别逻辑上可连接或对比的概念。)
    

**【举例演示】**

- **输入 (Input)**:

    ```JSON
    {
      "lists_of_concepts": [
        ["Transformer", "Attention Mechanism"],  // 来自文档 A
        ["RNN", "Gradient Vanishing"]            // 来自文档 B
      ]
    }
    ```

- LLM 思考与输出 (Output):
    

    LLM 发现 "Transformer" 和 "RNN" 是两类模型，适合对比。

    ```    JSON
    {
      "combinations": [
        ["Transformer", "RNN"]
      ]
    }
    ```

#### 2. 角色与主题匹配 (Themes & Personas Matching)

- **目的**: 决定由“谁”来问这个问题。
    
- **源码位置**: `ragas/testset/synthesizers/prompts.py` 中的 `ThemesPersonasMatchingPrompt` 类。
    
- **Prompt 指令 (Instruction)**:
    
    > "Given a list of themes and personas with their roles, associate each persona with relevant themes based on their role description."
    > 
    > (给定主题列表和带有角色描述的角色，根据角色描述将每个角色与相关主题关联起来。)
    

**【举例演示】**

- **输入 (Input)**:

    ```    JSON
    {
      "themes": ["Transformer", "RNN"],
      "personas": [
        {
          "name": "Senior AI Researcher",
          "role_description": "Focuses on model architecture evolution and performance comparison."
        },
        {
          "name": "Junior Python Developer",
          "role_description": "Focuses on basic syntax and library usage."
        }
      ]
    }
    ```

- LLM 思考与输出 (Output):
    

    LLM 判断这俩概念属于架构演进，适合“高级 AI 研究员”，不适合“初级 Python 开发”。

    ```JSON

    {
      "mapping": {
        "Senior AI Researcher": ["Transformer", "RNN"]
      }
    }
    ```

---

### 阶段二：场景构建 (Scenario Construction)

这一步在 Python 代码内部完成，不调用 LLM。Ragas 会将上述 LLM 的输出组装成一个 `MultiHopScenario` 对象：

- **Nodes**: [文档 A, 文档 B]
    
- **Persona**: Senior AI Researcher
    
- **Themes**: ["Transformer", "RNN"]
    
- **Style**: "Formal" (随机或指定)
    
- **Length**: "Medium"

---

### 阶段三：最终生成 (Generation)

这是最后一步，LLM 根据完整的场景描述写出具体的问答对。

#### 3. 问答生成 (Query & Answer Generation)

- **目的**: 生成符合角色口吻的问题，并根据上下文生成严格忠实的答案。
    
- **源码位置**: `ragas/testset/synthesizers/multi_hop/prompts.py` 中的 `QueryAnswerGenerationPrompt` 类。
    
- **Prompt 指令 (核心部分)**:
    
    > "Generate a multi-hop query and answer based on the specified conditions...
    > 
    > 1. **Generate a Multi-Hop Query**: ...Ensure the query uses information from at least two segments and connects them meaningfully.
    >     
    > 2. Generate an Answer: Use only the content from the provided context... Avoid adding information that is not directly present..."
    >     
    >     (基于指定条件生成多跳问答... 1. 生成多跳问题：确保问题利用了至少两个片段的信息并有意义地连接... 2. 生成答案：仅使用提供的上下文内容... 避免添加未出现的信息...)
    >     
    

**【举例演示】**

- **输入 (Input) - 传给 LLM 的 Prompt**:

    ```Plaintext
    Instruction: ... (上面的指令) ...
    
    Context: 
    <1-hop> Transformer 架构由 Google 在 2017 年提出，它完全基于注意力机制，抛弃了循环和卷积。
    <2-hop> RNN（循环神经网络）常用于处理序列数据，但它存在梯度消失问题，且难以并行计算。
    
    Persona: Senior AI Researcher
    Themes: ["Transformer", "RNN"]
    Style: Formal
    Length: Medium
    ```

- LLM 思考与输出 (Output):
    

    LLM 扮演研究员，结合两段话（抛弃循环 vs 循环的缺点）生成对比问题。

    ```  JSON
    {
      "query": "From the perspective of model architecture evolution, how does the Transformer differ from RNNs, particularly regarding their core mechanisms and computational limitations?",
      "answer": "The Transformer architecture differs from RNNs by entirely relying on attention mechanisms and discarding recurrence. While RNNs are used for sequence data, they suffer from gradient vanishing and are difficult to parallelize, whereas Transformers overcome these reliance on recurrence."
    }
    ```

### 总结：Prompt 设计的精髓

1. **分而治之**: 并没有用一个巨型 Prompt 直接生成问题，而是先拆解成“概念组合”和“角色匹配”两个小任务，确保逻辑链路（Logic Chain）是通的。
    
2. **强制上下文标记**: 在最终生成时，使用了 `<1-hop>`, `<2-hop>` 这样的标签，强制 LLM 在生成多跳问题时必须“跨越”这些标签去寻找信息，从而保证了生成的题目确实是“多跳”的，而不是只盯着其中一段。
    
3. **Pydantic 约束**: 整个流程的每一步输入输出都是 JSON，这让 Python 代码可以精确控制数据流，而不是去解析一段混乱的自然语言文本。