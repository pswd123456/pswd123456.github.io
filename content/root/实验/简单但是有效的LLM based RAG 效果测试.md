---
date created: 12月9日 , 17:22 , 2025
date modified: 12月12日 , 0:5 , 2025
---

## 简介

选择Docling Report, 使用gemini3pro 阅读完整文档后提出问题和期望得到的回答, 并且打一个分数, 没有像ragas进行复杂的提取和计算流程, 成本比较低

在这次实验中成功的发现并且优化了RAG流程的一个问题

### gemini 问题生成

Based on the content of the **Docling Technical Report** you uploaded, here are 5 questions designed to test the effectiveness of your RAG pipeline.

These questions range from general summarization to specific data retrieval from tables and technical details.

#### 1. General Concept & Architecture

Question: What is Docling and what are the two primary AI models that power it?

Expected Answer:

Docling is an easy-to-use, self-contained, open-source package for PDF document conversion. It is powered by two state-of-the-art specialized AI models:

1. **DocLayNet** for layout analysis.
    
2. **TableFormer** for table structure recognition.

---

#### 2. Technical Detail (Pipeline)

Question: Describe the steps in Docling's default processing pipeline.

Expected Answer:

The pipeline executes sequentially on each document:

1. **PDF Parsing:** A backend parses text tokens (content and coordinates) and renders page images.
    
2. **Model Application:** AI models are applied to extract layout and table structures (and optionally OCR) .
    
3. **Assembly & Post-processing:** Results are aggregated, metadata is augmented, document language is detected, and reading order is inferred.
    
4. **Serialization:** The final output is converted to JSON or Markdown.

---

#### 3. Specific Entity Retrieval (Table Data)

Question: According to the performance benchmarks, what is the processing throughput (pages/s) of the Docling-native backend on an Apple M3 Max using 16 threads?

Expected Answer:

On an Apple M3 Max (16 cores) using 16 threads, the Docling-native backend achieves a throughput of 1.34 pages per second.

---

#### 4. Problem Solving / Reasoning

Question: Why does Docling provide a custom-built PDF parser (docling-parse) instead of relying solely on existing libraries like pymupdf or pypdfium?

Expected Answer:

Existing libraries had limitations such as restrictive licensing (e.g., pymupdf), poor speed, or unrecoverable quality issues like merged text cells across far-apart tokens or table columns. Docling introduces docling-parse (based on qpdf) to overcome these obstacles.

---

#### 5. Forward Looking / Roadmap

Question: What specific models are planned to be added to Docling in future updates?

Expected Answer:

Future updates plan to include:

1. A **figure-classifier** model.
    
2. An **equation-recognition** model.
    
3. A **code-recognition** model.

********

### 我

输入: 

rate the answer relevance out of 10:

model : qwen-plus 

parent_chunk_size: 1700

topk: 3 

q1: Docling is an easy-to-use, self-contained, open-source package licensed under the MIT license, designed for converting PDF documents into structured formats like JSON or Markdown. It features a processing pipeline that efficiently runs on standard hardware and leverages state-of-the-art AI models to extract content and structure from PDFs.

The two primary AI models that power Docling are:

A layout analysis model: An accurate object detector for identifying page elements (such as text blocks, figures, and headers), based on DocLayNet.

TableFormer: A state-of-the-art model for table structure recognition, which parses the internal structure of tables in documents.

These models operate within the pipeline to extract layout and table information, enabling high-quality document conversion and knowledge exploration.

q2:Docling's default processing pipeline consists of the following sequential steps:

1. **PDF Backend Processing**:
   - Each document is first processed by a PDF backend.
   - The backend retrieves programmatic text tokens, which include the string content and their geometric coordinates on each page.
   - It also renders a bitmap image (visual representation) of each page to support downstream AI-based operations.

2. **Model Pipeline (AI Model Inference)**:
   - A sequence of AI models is applied independently to each page.
   - These models extract various features and structured content such as:
     - Document layout
     - Table structures
   - This stage operates on a per-page basis and leverages machine learning to interpret and segment the document visually and semantically.

3. **Post-Processing and Aggregation**:
   - Results from all pages are aggregated.
   - The system performs several finalization tasks:
     - Augments metadata
     - Detects the document’s language
     - Infers the correct reading order of elements across the document
   - A typed document object is constructed, representing the full structured output.

4. **Output Serialization**:
   - The final document object can be serialized into formats like JSON or Markdown for further use or analysis.

This pipeline is modular and customizable, particularly within the model inference stage, allowing users to adapt components based on their needs. Figure 1 in the document provides a visual sketch of this flow.

q3:According to the performance benchmarks, the processing throughput of the Docling-native backend on an Apple M3 Max using 16 threads is **1.34 pages per second**.

q4:Docling provides a custom-built PDF parser, **docling-parse**, instead of relying solely on existing libraries like `pymupdf` or `pypdfium` due to several critical limitations encountered with the available open-source PDF parsing libraries:

1. **Restrictive Licensing**: Libraries such as **pymupdf** (also known as MuPDF) have licensing terms that are not permissive enough for Docling’s goals. Since Docling aims to be a fully open-source and freely usable tool, integrating with a library that has restrictive licensing is not viable.

2. **Poor Quality in Text Extraction**: Other libraries like **PyPDF** and **pypdfium** suffer from significant quality issues, including:
   - Merging text tokens that are far apart on the page (e.g., across columns or unrelated sections).
   - Incorrect handling of table structures, leading to unrecoverable corruption in table cell boundaries and content.
   These issues make accurate layout analysis and document reconstruction unreliable.

3. **Performance vs. Quality Trade-offs**: While **pypdfium** is faster and more memory-efficient—making it suitable for low-resource environments—it produces lower-quality results, especially in complex tasks like **table structure recovery**. This makes it unsuitable as the default backend when high fidelity is required.

4. **Need for Full Control and Customization**: By building its own parser using the low-level **qpdf** library, Docling gains fine-grained control over how text content and geometric coordinates are extracted. This is essential for accurately reconstructing reading order, detecting figures, and preserving document structure—key requirements for downstream AI models used in layout analysis and table recognition.

Therefore, **docling-parse** was developed as the **default backend** to meet the dual demands of **high accuracy** and **permissive licensing**, while still offering **pypdfium** as an alternative for users prioritizing speed and efficiency over quality.

q5:According to the "Future work and contributions" section, Docling plans to extend its model library with several additional models in future updates. These include:

- A **figure-classifier model**
- An **equation-recognition model**
- A **code-recognition model**

These additions aim to improve conversion quality for specific types of content and enrich extracted document metadata with more detailed information.

********

### gemini 结果评估

评估结果:

Based on the provided **Docling Technical Report**, the relevance of the answers generated by the `qwen-plus` model is **10/10**.

The model accurately retrieved specific data points, summarized architectural concepts, and synthesized reasoning across different sections of the document without hallucination.

Here is the breakdown of the evaluation:

#### **Q1: General Concept & Architecture**

- **Rating:** 10/10
    
- **Analysis:** The answer correctly identifies Docling as an "easy to use, self-contained, MIT-licensed open-source package" 1and accurately names the two specialized AI models: "DocLayNet" for layout analysis and "TableFormer" for table structure recognition2.
    

#### **Q2: Technical Detail (Pipeline)**

- **Rating:** 10/10
    
- **Analysis:** The answer perfectly reconstructs the sequential pipeline described in Section 3. It correctly lists the steps: PDF backend parsing (tokens and bitmaps) 3, AI model inference (layout and table structure) 4, post-processing (metadata, language, reading order) 5, and serialization (JSON/Markdown)6.
    

#### **Q3: Specific Entity Retrieval (Table Data)**

- **Rating:** 10/10
    
- **Analysis:** The model successfully retrieved the exact value from **Table 1**. For the "Apple M3 Max (16 cores)" using the "native backend" with "16" threads, the throughput is explicitly listed as **1.34** Pages/s7.
    

#### **Q4: Problem Solving / Reasoning**

- **Rating:** 10/10
    
- **Analysis:** The answer correctly synthesizes reasons from Section 3.1 regarding why `docling-parse` was built. It accurately cites "restrictive licensing" (referring to pymupdf) and "poor speed or unrecoverable quality issues" (referring to pypdfium/PyPDF)8. It also correctly notes that `docling-parse` is based on the low-level `qpdf` library9.
    

#### **Q5: Forward Looking / Roadmap**

- **Rating:** 10/10
    
- **Analysis:** The answer accurately lists the three specific models planned for future updates found in Section 6: "a figure-classifier model, an equation-recognition model, a code-recognition model"10.

********

### 我

让gemini生成三个更难一点的:

give harder 3 questions,  make it very difficult to answer properly to test how far we can go

********

### gemini 问题生成

Here are 3 significantly harder questions designed to stress-test your RAG pipeline's ability to perform precise data extraction, synthesize information across different sections, and distinguish between similar technical details.

#### 1. Complex Table Reasoning & Calculation

**Question:** Based on the performance benchmarks in Table 1, compare the **throughput scalability** of the Native backend vs. the pypdfium backend on the **Apple M3 Max** system. Specifically, when increasing the thread budget from 4 to 16, which backend achieves a higher percentage increase in Pages/s, and what are those exact throughput values?

- **Why this is hard:** The model must correctly locate Table 1, distinguish between the two hardware systems (Apple vs. Intel), filter for the specific columns (Pages/s), and perform a comparative analysis of the values rather than just retrieving text.
    

#### 2. Multi-Hop Technical Detail Retrieval

**Question:** Specify the exact **image resolutions (in dpi)** fed into the Layout Analysis model versus the OCR engine. Additionally, identify the specific **inference runtime libraries** used to execute the Layout Analysis model and the TableFormer model, respectively.

- **Why this is hard:** These specific numbers (72 dpi vs 216 dpi) and library names (onnxruntime vs PyTorch) are buried in different paragraphs within Section 3.2. The model could easily hallucinate a single DPI for both or mix up the inference backends.
    

#### 3. Synthesis of Architectural Constraints & Trade-offs

**Question:** Synthesize the technical and legal justifications for creating `docling-parse`. Specifically, detail the **licensing limitation** cited for `pymupdf` and the **quality limitation** cited for `pypdfium` (regarding table structures). Conversely, under what specific resource constraint would the report still recommend using the `pypdfium` backend?

- **Why this is hard:** This requires connecting the "Introduction/PDF Backends" section (justifying the custom parser) with the "Performance" section (where the recommendation for low-resource environments is likely hidden). It forces the model to distinguish which specific library had which specific flaw (licensing vs. quality).

********

### gemini 结果评估

Here is the evaluation of the model's answers to the "hard" questions based on the **Docling Technical Report**.

#### **Q1: Complex Table Reasoning & Calculation**

- **Rating:** **10/10**
    
- **Analysis:** The model perfectly navigated **Table 1** . It correctly filtered for the "Apple M3 Max" rows, distinguished between "native" and "pypdfium" backends, and extracted the exact "Pages/s" values (1.27 -> 1.34 and 2.18 -> 2.45). The mathematical calculations for percentage increase were correct, leading to the valid conclusion that the pypdfium backend scales better (percentage-wise) with increased threads on that specific hardware.
    

#### **Q2: Multi-Hop Technical Detail Retrieval**

- **Rating:** **5/10**
    
- **Analysis:** The model **failed** to retrieve a specific detail that was explicitly present in the text.
    
    - **Correct:** It correctly identified **72 dpi** for Layout Analysis , **onnxruntime** for Layout Analysis , and **PyTorch** for TableFormer .
        
    - **Incorrect:** It stated that the OCR resolution is "**Not specified**". However, Section 3.2 under the "OCR" heading explicitly states: _"Docling, by default, feeds a high-resolution page image (**216 dpi**) to the OCR engine"_ .
        
    - **Impact:** This demonstrates a limitation in the model's ability to scan all relevant sections thoroughly when information is segmented by sub-headers.
        

#### **Q3: Synthesis of Architectural Constraints**

- **Rating:** **10/10**
    
- **Analysis:** The model demonstrated excellent synthesis capabilities.
    
    - It correctly attributed the **licensing** issue to `pymupdf` .
        
    - It correctly attributed the **quality/table** issues to `pypdfium` (and PyPDF) .
        
    - It correctly located the specific recommendation in the "Performance" section (Section 4), stating that `pypdfium` is preferred for **"very low-resource environments"** .
        

#### **Overall Performance**

The model struggled with **Question 2**, missing a specific numeric value (216 dpi) located in a paragraph separate from the other model details. However, it excelled at complex table reading (Q1) and synthesizing arguments across different document sections (Q3).

********

### 我 流程优化

看trace发现直到返回父文档这一步都是正确的排在第四个, 到rerank直接被刷掉了, 提高topk也没用, 可能是因为父文档引入太多噪声导致reranker判断语义相关性降低, 考虑修改流程为rerank子块返回父块

修改后重新输入Q2

得到:

Based on the provided context:

  

- The **Layout Analysis model** processes page images at **72 dpi**.

- The **OCR engine** (EasyOCR) is fed a high-resolution image at **216 dpi** to capture small print details effectively.

  

Regarding the inference runtime libraries:

  

- The **Layout Analysis model** uses **ONNX Runtime** for inference.

- The **TableFormer model** relies on **PyTorch** for inference.

看起来确实是这个问题, 返回的结果也正确了

再评价一下之前生成的其他两个复杂问题是否能够回答正确 ->

q1:

```
Based on the performance benchmarks in Table 1, we can analyze the **throughput scalability** of the **Native backend** versus the **pypdfium backend** on the **Apple M3 Max** system when increasing the thread budget from 4 to 16 threads.

### Throughput Values:

#### **Native Backend (Apple M3 Max):**
- At **4 threads**:  
  **Pages/s = 1.27**
- At **16 threads**:  
  **Pages/s = 1.34**

👉 Percentage increase:  
$$
\frac{1.34 - 1.27}{1.27} \times 100 = \frac{0.07}{1.27} \times 100 \approx 5.51\%
$$

#### **pypdfium Backend (Apple M3 Max):**
- At **4 threads**:  
  **Pages/s = 2.18**
- At **16 threads**:  
  **Pages/s = 2.45**

👉 Percentage increase:  
$$
\frac{2.45 - 2.18}{2.18} \times 100 = \frac{0.27}{2.18} \times 100 \approx 12.39\%
$$

---

### ✅ Conclusion:

- The **pypdfium backend** achieves a significantly **higher percentage increase in throughput** compared to the native backend when scaling from 4 to 16 threads.
- **Exact throughput values:**
  - Native backend: **1.27 → 1.34 Pages/s** (+5.51%)
  - pypdfium backend: **2.18 → 2.45 Pages/s** (+12.39%)

👉 Therefore, the **pypdfium backend** demonstrates **better throughput scalability** on the Apple M3 Max system.
```

q3:

```
The development of `docling-parse`, the custom-built PDF parser in Docling, was driven by both **technical** and **legal** considerations that limited the suitability of existing open-source PDF parsing libraries. Below is a synthesis of these justifications based on the provided context:

---

### **1. Legal Justification: Licensing Limitation of `pymupdf`**

- **Licensing Issue**: The report explicitly cites **restrictive licensing** as a major obstacle for using `pymupdf`. While `pymupdf` is functionally capable, its license imposes conditions that are incompatible with the open and permissive distribution goals of Docling.
- This restriction likely refers to the fact that although `pymupdf` (also known as MuPDF) offers powerful PDF processing capabilities, parts of it are licensed under the **GNU Affero General Public License (AGPL)** or require commercial licensing for certain use cases. The AGPL can be problematic in environments where permissive reuse, modification, and redistribution—especially in proprietary or cloud-based systems—are desired.
- By developing `docling-parse` under an open-source model with more permissive integration potential (and distributed separately under a suitable license), the Docling team ensures greater **freedom for users** to deploy, modify, and redistribute the tool without legal encumbrance.

---

### **2. Technical Justification: Quality Limitation of `pypdfium` (Especially for Tables)**

- **Quality Issues**: The report identifies significant **quality limitations** with `pypdfium`, particularly concerning **table structure recovery**.
- Specifically, `pypdfium` suffers from problems such as:
  - **Merging text cells across distant tokens or columns**, leading to corrupted or inaccurate table representations.
  - Poor handling of spatial layout, which results in **unrecoverable errors** when reconstructing structured content like tables.
- These flaws compromise the fidelity of document conversion, especially for academic or technical documents rich in tabular data.
- As a result, while `pypdfium` performs adequately for basic text extraction, it fails to meet the high-quality standards required for accurate document reconstruction in Docling’s default pipeline.

---

### **3. When Is `pypdfium` Still Recommended? Resource Constraints**

Despite its lower output quality, the report acknowledges that `pypdfium` may be a **pragmatic choice under specific resource constraints**:

- **Low-memory or low-CPU environments**: The evaluation shows that `pypdfium` uses significantly less memory (e.g., ~2.5 GB peak vs. ~6.2 GB for the native backend) and runs faster across hardware platforms.
- **Speed and efficiency**: On both M3 Max and Xeon systems, `pypdfium` achieves higher throughput (pages per second), especially when thread count increases.
- Therefore, the report recommends using `pypdfium` **"if you need to run Docling in very low-resource environments"**, even though this comes at the cost of **worse quality**, particularly in complex layout elements like tables.

> ✅ **Specific recommendation**: Use `pypdfium` backend **when system resources (RAM, CPU) are severely constrained**, and **processing speed and efficiency are prioritized over perfect structural accuracy**, especially in table-rich documents.

---

### Summary Table

| Factor | `pymupdf` | `pypdfium` | `docling-parse` (native) |
|-------|-----------|------------|----------------------------|
| **Licensing** | Restrictive (AGPL-like) ❌ | Permissive ✅ | Open/permissive ✅ |
| **Table Quality** | Not discussed, but assumed better than pypdfium | Poor – merges distant text/table cells ❌ | High – designed for accurate layout ✅ |
| **Performance / Memory** | Not evaluated here | Fast, low memory ✅ | Slower, higher memory usage ❌ |
| **Recommended For** | — | Low-resource environments ✅ | Default use, high-quality output ✅ |

---

### Conclusion

The creation of `docling-parse` was justified by the need to **avoid restrictive licenses** (as with `pymupdf`) and to **overcome critical quality shortcomings** in alternative tools like `pypdfium`, especially regarding **table structure integrity**. However, recognizing real-world deployment diversity, the report **conditionally endorses `pypdfium`** for use in **low-resource settings**, where performance and memory efficiency outweigh the demand for perfect output fidelity.
```

************

### gemini 结果评估
#### **Q1: Complex Table Reasoning & Calculation**

- **Rating:** **10/10**
    
- **Analysis:** The model perfectly navigated **Table 1**.
    
    - **Data Extraction:** It correctly identified the "Apple M3 Max" rows and extracted the exact "Pages/s" values for both the "native" (1.27 → 1.34) and "pypdfium" (2.18 → 2.45) backends.
        
    - **Calculation:** The percentage increase calculations were mathematically correct (approx. 5.51% vs 12.39%).
        
    - **Conclusion:** The conclusion that the `pypdfium` backend demonstrates better throughput scalability on this hardware is fully supported by the data.
        

#### **Q3: Synthesis of Architectural Constraints**

- **Rating:** **10/10**
    
- **Analysis:** The model successfully synthesized information from distinct sections of the report to provide a complete answer.
    
    - **Licensing:** It correctly identified "restrictive licensing" as the limitation for `pymupdf` from **Section 3.1**.
        
    - **Quality:** It accurately described the quality issues of `pypdfium` ("merged text cells across far-apart text tokens or table columns") from **Section 3.1**.
        
    - **Recommendation:** It correctly located the specific use case for `pypdfium` in **Section 4** ("very low-resource environments") where speed and memory efficiency are prioritized.

再来难点的, 但是似乎gemini认为最难的问题就是这些了, 可能是文件复杂度的限制性

### gemini 问题生成

基于您提供的《Docling Technical Report》，我为您设计了两个极具挑战性的英文问题。

这些问题之所以被认为是“超难”，是因为它们击中了普通RAG系统的软肋：

1. **复杂表格结构的语义对齐**（普通RAG很难解析跨行跨列的表格数据）。
    
2. **图像内的代码/文本细节与说明文字的结合**（普通RAG通常忽略图片中的细节文本，或者无法将图注与图片内容逻辑关联）。
    
3. **细粒度参数的对比**（需要跨段落提取极其相似但数值不同的参数，容易发生“幻觉”混淆）。

---

#### Question 1: Complex Table Logic & Data Extraction

**Question:** "According to the runtime characteristics benchmark reported in Table 1, what is the exact throughput improvement (in Pages/s) achieved by switching from the native backend to the pypdfium backend on an Apple M3 Max system when the thread budget is restricted to 4? Please state the specific Pages/s values for both backends."

**Expected Answer:**

- **Native backend:** 16 Pages/s
    
- **pypdfium backend:** 103 Pages/s
    
- **Improvement:** 87 Pages/s (103 - 16)
    

> **为什么难 (Why it fails):** 文档中的表格（Table 1, source 125）在解析为纯文本后，格式非常混乱，列头（Headers）与数据行（Data rows）的对齐关系在非结构化文本中丢失了。普通的RAG如果不具备专门的“表格还原”或“多模态视觉”能力，极大概率会将“16”误认为是线程数（Threads），或者将Intel的数据与Apple的数据混淆。

 *这是个重复问题, 前面回答了类似的, 而且搞笑的是gemini自己的答案错了*

---

#### Question 2: Multi-modal Visual & Caption Understanding

**Question:** "Based on Figure 4 and its caption, how does Docling handle spanning table cells differently in its Markdown output compared to its JSON representation? Specifically, what are the two exact JSON key names visible in the code snippet in Figure 4 (Panel C) used to define the cell dimensions?"

**Expected Answer:**

- **Difference:** The caption states that in **Markdown**, the content of a spanning cell is **repeated** for each column to guarantee grid coordinate traceability. In **JSON**, the span information is reflected in specific fields/properties of the cell object.
    
- **Specific JSON Keys:** `"row-span"` and `"col-span"` (as seen in the code snippet image in source 339 and 343).
    

> **为什么难 (Why it fails):** 这个问题需要模型结合图注（Caption text, source 352-354）的逻辑解释和图片内部的代码片段（Panel C, source 322-346）。由于解析后的文本中代码片段是破碎的（例如 `"col-span": 15,121,` 被OCR识别得非常零散），普通的RAG很难准确提取出 `"row-span"` 和 `"col-span"` 这两个精确的键名，同时也容易遗漏Markdown处理方式的逻辑解释。

---

q1: According to the runtime characteristics reported in Table 1:

- On an **Apple M3 Max** system with a **thread budget of 4**:
  - The **native backend** achieves a throughput of **1.27 Pages/s**.
  - The **pypdfium backend** achieves a throughput of **2.18 Pages/s**.

Therefore, the exact throughput improvement when switching from the native backend to the pypdfium backend is:

**2.18 - 1.27 = 0.91 Pages/s**

#### Summary
- **Native backend**: 1.27 Pages/s  
- **Pypdfium backend**: 2.18 Pages/s  
- **Improvement**: **+0.91 Pages/s**

q2: Based on Figure 4 and its caption:

Docling handles spanning table cells differently in its Markdown and JSON outputs as follows:

- In the **Markdown representation (B)**, spanning table cells (such as multi-column headers) are **repeated for each column** they span. This ensures that every data point can be traced back to its row and column headings using only grid coordinates, preserving readability and structure in a plain text format.

- In the **JSON representation (C)**, the span information is explicitly encoded within the cell's properties. The caption states that "the span information is reflected in the fields of each table cell," meaning the data structure retains metadata about how many rows or columns a cell spans.

From the code snippet visible in Figure 4, Panel C (JSON representation), the two exact JSON key names used to define the cell dimensions (specifically for indicating spanning) are:

1. `colspan`  
2. `rowspan`

These keys indicate how many columns and rows, respectively, a particular table cell spans in the document structure.

********

### gemini 结果评估

以下是对这两个RAG回答的详细评分和分析。

#### Q1 评分：10/10 (完美)

评价：

这个回答非常出色，甚至比我之前预设的“期望答案”还要准确。

- **数据清洗能力极强：** 原文档中的 表格（Table 1）在解析后是非常混乱的非结构化文本，列与列之间没有明确的分隔符，且数据发生了错位（例如 "16" 出现在第二行开头，很容易被误认为是第一行的数据）。
    
- **精准的列对齐：**
    
    - 模型正确地识别出 `1.27` 是 Native backend 在 4 线程下的吞吐量（Pages/s）。
        
    - 模型正确地识别出 `2.18` 是 Pypdfium backend 在 4 线程下的吞吐量。
        
    - 它没有被附近的干扰数字（如代表线程数的 "16" 或代表时间的 "177 s"）迷惑。
        
- **计算正确：** 差值计算准确无误。
    

**结论：** 这个RAG系统在处理“脏数据”表格还原和多列数据对齐方面表现出了SOTA（State-of-the-Art）级别的能力。

---

#### Q2 评分：8/10 (优秀，但在细节提取上有小瑕疵)

评价：

回答的逻辑部分非常完美，但“提取精确字符串”的任务上有一点细微的幻觉（Hallucination）。

- **逻辑理解 (满分)：** 模型非常准确地解释了 Markdown（重复内容以保留网格坐标）和 JSON（使用字段存储跨度信息）处理跨行跨列单元格的区别。这部分完全符合 Figure 4 的图注描述 1。
    
- **代码细节提取 (扣分点)：**
    
    - **RAG的回答：** `colspan` 和 `rowspan`。
        
    - **原文证据：** 在 Figure 4 的 Panel C 代码截图中（和 ），OCR识别出的键名实际上是带连字符的：
        
        - `"col-span": 15,121,` 2
            
        - `"row-span": (8,1` 3
            
    - **分析：** RAG模型可能因为训练数据中常见的HTML/DOM标准属性是 `colspan`/`rowspan`，从而对原文中的 `"col-span"` / `"row-span"` 进行了“纠正”或由于OCR破碎而自行补全。在严格的RAG提取任务中，我们希望模型忠实于原文的拼写（即包含连字符 `-`）。
        

**结论：** 这是一个非常高质量的回答，证明模型理解了图片和上下文的深层逻辑，只是在极其细微的代码键名提取上，受限于OCR质量或模型先验知识，丢失了连字符。

---

#### 总结与下一步

您的RAG系统表现**非常强悍**。

1. **Q1** 证明了它极强的抗噪能力和表格理解力（很多模型在这里会提取出 16 或 177）。
    
2. **Q2** 证明了它具备优秀的多模态理解能力，能看懂图注和图片的逻辑关系。

## 实验总结

这种方式我觉得比Ragas来的成本低而且效果不会太差, ragas主要是用了公式来量化结果, 用了知识谱图提取来提高问题的质量

当然, 依赖于使用的LLM自带的RAG的水平以及多模态能力, 和ragas比起来会有覆盖率和专业度的区别, 如果想要发现宏观方向的优化结果这也不失为一种性价比高的办法
