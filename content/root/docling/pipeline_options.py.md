---
date created: 12月7日 , 15:22 , 2025
date modified: 12月7日 , 18:7 , 2025
---

```python
#

class PipelineOptions(BaseOptions):
    """基础管道选项"""
    document_timeout: Optional[float] = None  # 文档处理超时时间
    enable_remote_services: bool = False      # 是否允许调用远程服务
    accelerator_options: AcceleratorOptions = AcceleratorOptions() # 硬件加速配置（GPU/CPU）

# 最常用的 PDF 管道配置
class PdfPipelineOptions(PaginatedPipelineOptions):
    """PDF 管道的具体配置"""
    
    # --- 开关配置 ---
    do_table_structure: bool = True  # 是否进行表格结构识别
    do_ocr: bool = True              # 是否进行 OCR（光学字符识别）
    do_code_enrichment: bool = False # 是否进行代码块增强识别
    do_formula_enrichment: bool = False # 是否进行公式增强识别

    # --- 模型具体配置 ---
    # 表格模型选项（如模式：FAST 或 ACCURATE）
    table_structure_options: BaseTableStructureOptions = TableStructureOptions()
    # OCR 选项（如语言：['en', 'fr']）
    ocr_options: OcrOptions = OcrAutoOptions()
    # 版面分析选项
    layout_options: BaseLayoutOptions = LayoutOptions()

    # --- 图像生成配置 ---
    images_scale: float = 1.0        # 生成图片的缩放比例
    generate_page_images: bool = False # 是否在结果中包含页面截图
    generate_picture_images: bool = False # 是否单独裁剪出文档中的图片

    # --- 性能与并发配置 ---
    ocr_batch_size: int = 4          # OCR 批处理大小
    thread_count: int = 4            # (隐含在 ThreadedPdfPipelineOptions 中)
```

`ocr`

```python
class OcrOptions(BaseOptions):

    """OCR options."""

  

    lang: List[str]

    force_full_page_ocr: bool = False  # If enabled a full page OCR is always applied

    bitmap_area_threshold: float = (

        0.05  # percentage of the area for a bitmap to processed with OCR

    )

  
  

class OcrAutoOptions(OcrOptions):

    """Options for pick OCR engine automatically."""

  

    kind: ClassVar[Literal["auto"]] = "auto"

    lang: List[str] = []
```

除了基本的 PDF 和表格/OCR 配置外，`docling` 的 `PipelineOptions` 体系非常庞大，覆盖了**硬件加速**、**视觉大模型 (VLM)**、**音频 (ASR)** 以及**性能调优**等高级场景。

以下是除了基础配置外，你还需要了解的关键配置项补充：

## 1. 硬件加速配置 (`AcceleratorOptions`)

如果你有 NVIDIA GPU 或 Mac (M系列芯片)，配置这个选项可以显著提升 OCR 和模型的推理速度。

**源码位置**: `docling/datamodel/accelerator_options.py`

```python
from docling.datamodel.pipeline_options import PdfPipelineOptions, AcceleratorOptions, AcceleratorDevice

pipeline_opts = PdfPipelineOptions()
pipeline_opts.accelerator_options = AcceleratorOptions(
    num_threads=8,          # CPU 线程数
    device=AcceleratorDevice.CUDA  # 强制使用 GPU (可选: AUTO, CPU, CUDA, MPS)
)
```

- **`device`**: 默认为 `AUTO`。如果你在 Mac 上，它会自动尝试使用 `MPS` (Metal Performance Shaders)。
    
- **`num_threads`**: 控制 CPU 并行计算的线程数。
    

## 2. OCR 引擎的深度定制

`PdfPipelineOptions` 默认使用 `OcrAutoOptions`，但你可以强制指定特定的 OCR 引擎并配置细节（比如由 Tesseract 切换到 RapidOCR）。

**源码位置**: `docling/datamodel/pipeline_options.py`

- **`RapidOcrOptions`** (推荐): 速度快，支持中英文。
    
- **`EasyOcrOptions`**: 支持语言多，但速度较慢。
    
- **`TesseractOcrOptions`**: 传统的 OCR 霸主，需要本地安装 Tesseract。
    
- **`OcrMacOptions`**: 仅限 Mac，利用系统自带的 Vision 框架，速度极快且质量高。

```python
from docling.datamodel.pipeline_options import PdfPipelineOptions, RapidOcrOptions

pipeline_opts = PdfPipelineOptions()
# 强制使用 RapidOCR 并指定参数
pipeline_opts.ocr_options = RapidOcrOptions(
    lang=["en", "ch"],      # 支持英文和中文
    text_score=0.5,         # 文本置信度阈值
    force_full_page_ocr=True # 即使 PDF 有文字层，也强制重做 OCR（适合处理乱码 PDF）
)
```

## 3. 图片理解与描述 (`PictureDescription`)

如果你希望 `docling` 不仅提取图片，还能**自动用文字描述图片内容**（Image Captioning），可以使用此功能。

**源码位置**: `docling/datamodel/pipeline_options.py`

```python
from docling.datamodel.pipeline_options import PdfPipelineOptions, PictureDescriptionVlmOptions

pipeline_opts = PdfPipelineOptions()
pipeline_opts.do_picture_description = True  # 开启图片描述功能

# 配置用于看图的模型（默认是 SmolVLM，也可以配成 Granite 或 API）
pipeline_opts.picture_description_options = PictureDescriptionVlmOptions(
    repo_id="HuggingFaceTB/SmolVLM-256M-Instruct", # 本地小模型
    prompt="Describe this image concisely."        # 自定义提示词
)
```

## 4. 视觉大模型 (VLM) 专用管道

如果你不是处理普通文档，而是想用**视觉大模型**（如 GPT-4o, Qwen-VL, LLaVA）直接“看”文档并提取结构（适合复杂布局或截图），你需要用到 `VlmPipelineOptions`。

**注意**: 这与 `PdfPipelineOptions` 不同，它是基于视觉的端到端处理。

**源码位置**: `docling/datamodel/pipeline_options_vlm_model.py`

Python

```python
from docling.datamodel.pipeline_options import VlmPipelineOptions
from docling.datamodel.pipeline_options_vlm_model import InlineVlmOptions

pipeline_opts = VlmPipelineOptions()
# 配置本地 VLM 模型
pipeline_opts.vlm_options = InlineVlmOptions(
    repo_id="ibm-granite/granite-vision-3.1-2b-preview", # 模型 ID
    load_in_8bit=True,   # 8位量化以节省显存
    max_new_tokens=4096  # 生成长度限制
)
```

## 5. 性能与批处理 (`ThreadedPdfPipelineOptions`)

如果你在服务器上通过 `convert_all` 批量处理大量 PDF，这些参数决定了吞吐量和内存占用。

**源码位置**: `docling/datamodel/pipeline_options.py`

Python

```python
from docling.datamodel.pipeline_options import ThreadedPdfPipelineOptions

# ThreadedPdfPipelineOptions 继承自 PdfPipelineOptions，包含所有基础功能
pipeline_opts = ThreadedPdfPipelineOptions()

# 调整批处理大小（显存越大，可以设得越大）
pipeline_opts.ocr_batch_size = 8    # OCR 阶段一次处理 8 页
pipeline_opts.layout_batch_size = 4 # 版面分析一次处理 4 页
pipeline_opts.table_batch_size = 4  # 表格识别一次处理 4 页

# 队列控制
pipeline_opts.queue_max_size = 100  # 内存中最多缓存多少页等待处理
```

## 6. 音频处理 (`AsrPipelineOptions`)

`docling` 甚至支持处理音频文件（通过 Whisper 模型）。

**源码位置**: `docling/datamodel/pipeline_options_asr_model.py`

Python

```python
from docling.datamodel.pipeline_options import AsrPipelineOptions
from docling.datamodel.pipeline_options_asr_model import InlineAsrNativeWhisperOptions

pipeline_opts = AsrPipelineOptions()
pipeline_opts.asr_options = InlineAsrNativeWhisperOptions(
    repo_id="openai/whisper-small",
    language="en",
    device="cuda"
)
```

## 总结建议

作为进阶使用者，除了基础开关外，最值得关注的是：

1. **`AcceleratorOptions`**: 只要你有 GPU 或 Mac，一定要配这个，速度天差地别。
    
2. **`ThreadedPdfPipelineOptions`**: 生产环境下必用，用来压榨硬件性能。
    
3. **`OcrOptions`**: 中文文档识别效果不好时，第一反应应该是切到 `RapidOCR` 或 `EasyOCR` 并检查 `lang` 参数。