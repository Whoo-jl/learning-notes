# RAG 分块策略与选型

分块决定 RAG 能按什么粒度找到证据。块太大，一条具体规定可能淹没在无关内容里。块太小，往往会丢掉对象、单位或适用条件。对工程图纸来说，单独检索到“12 mm”没有多少用处。只有把它和孔位、深度类型、图号及版次对应起来，才能判断它能否支撑答案。

工程上有三个可以分别设计的环节。**切分边界**决定哪些内容留在同一块，**索引表示**决定用什么文本、向量或图像去找，**上下文扩展**决定命中后把多少原文交给模型。例如，先按章节和表格切，再给子块加标题路径，最后在命中时补父块。这三种做法可以叠加。以下整理的是能从官方实现和论文核对的代表性做法。

## 按长度和文本边界切分

**固定长度与滑动窗口。** 用字符、词或 tokenizer 计算长度。块长为 `S`、重叠为 `O` 时，滑窗步长是 `S − O`。例如 400 token、重叠 80 token，步长为 320 token。对足够长的连续文本，窗口数约为无重叠时的 1.25 倍。实际存储和推理成本还取决于索引和模型。它适合做快速、可复现的基线，缺点是可能从句中、表格中或条件中间断开。需要控制模型输入时，应按目标模型的 tokenizer 计数，并把标题等附加文本计入预算。 [Azure AI Search 分块说明](https://learn.microsoft.com/en-us/azure/search/vector-search-how-to-chunk-documents) · [Amazon Bedrock 分块说明](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)

**按句子或段落打包。** 先识别句子或段落，再把相邻单元打包到长度预算内。它适合手册正文、文章和会议转写。段落打包通常保留更多局部语境，按句打包更容易控制长度。工程上要定义超长句的兜底规则，并用中英文标点、型号、小数和编号列表检验分句器。例如 `0.05 mm` 中间的小数点不应该触发断句。 [LlamaIndex SentenceSplitter](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#sentencesplitter)

**递归分隔符。** 优先用空行等较粗边界，块仍超限时再尝试换行、标点、空格或字符级兜底。适合已转成纯文本、结构质量参差的资料。中文语料应显式考虑中文标点。列表缩进、表格和代码仍需要保护规则。配置的重叠通常是目标，实际重叠会受可用边界和打包方式影响。 [LangChain RecursiveCharacterTextSplitter](https://docs.langchain.com/oss/python/integrations/splitters/recursive_text_splitter)

## 按文档结构和业务对象切分

**按标题或文档元素切分。** 解析出章节、正文、列表、表格和图片后，先尊重这些结构边界，再按 token 预算拆长块或合并短块。块可以携带文档名、标题路径、页码与元素位置。例如“每 6 个月检查一次”保留所属标题后，才能定位到对应设备。Docling 的 HybridChunker 在文档层级切分上做 token 预算调整。Unstructured 提供按标题组织元素的策略。RAGFlow 则提供 General、Manual、Table、Paper 等模板，按文档类型选择解析与切分流程。结构分块的效果取决于解析结果能否保留正确的阅读顺序、标题层级和表格。 [Docling Chunking](https://docling-project.github.io/docling/concepts/chunking/) · [Unstructured Chunking](https://docs.unstructured.io/open-source/core-functionality/chunking) · [RAGFlow 数据集配置](https://github.com/infiniflow/ragflow/blob/main/docs/guides/dataset/configuration.md)

**按页、区域或时间切分。** 当页码、图面区域或时间戳本身是证据时，可按页、视图区域、说话轮次或时间段组织检索单元。页级单元容易回溯，但一页可能包含多个主题，跨页表格也需要额外处理。可以用页做来源边界，再索引页内的文本元素和图像区域。ColPali 是直接索引页图的多向量检索研究，验证的任务是页级检索。页图命中之后，图纸问答仍需要定位具体标注或视图。 [ColPali 论文](https://arxiv.org/abs/2407.01449)

**按业务对象或记录切分。** FAQ 可以一对问答一块，制度可以按带编号的条款切，BOM 可以按行或同一组件的行组切。表格行应携带表名、完整列头、行键、单位和有约束作用的脚注。例如 `B｜12` 应能还原成“孔位 B，螺纹有效深度 12 mm”，并关联图号及版次。跨行比较、汇总或计算可回读原表或走结构化查询。上游已划分好独立记录时，可以整条导入。Bedrock 的 no chunking 可用于上游已拆分的文件，也提供自定义导入转换。 [Amazon Bedrock 分块说明](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html) · [自定义导入转换](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-custom-transformation.html)

## 按语义变化或模型判断切分

**Embedding 语义断点。** 先分句，再对句子或包含邻句的小窗口生成向量，计算相邻窗口之间的距离。距离越过阈值时切开，并按需要限制块长。常见参数有句子窗口大小、距离阈值或阈值百分位、最大 token 数。以距离百分位为阈值时，提高阈值通常会减少切点。它适合标题少、话题转换明显的长叙述。额外的向量计算会增加入库成本。分句错误、符号密集或同一主题下的条款切换也可能造成误判。 [LlamaIndex SemanticSplitterNodeParser](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#semanticsplitternodeparser) · [Bedrock 语义分块](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)

**LLM 辅助切分与命题索引。** 模型可以辅助识别边界，例如为文档生成适用的分隔规则，再由代码按原文切分。另一条路线把段落拆成可独立理解的事实，并为每条命题建立索引。后一种做法还会改变索引内容，例如把代词换成明确对象。两条路线都增加模型调用和增量更新成本。对数值、否定、例外和版本，应该保留可回溯的原文区间，避免改写内容成为唯一证据。Dense X Retrieval 比较了命题、句子和段落等不同粒度的检索效果。 [Adaptive Chunking](https://arxiv.org/abs/2603.25333) · [Dense X Retrieval](https://arxiv.org/abs/2312.06648)

## 索引表示和命中后的上下文

| 做法 | 实现 | 适用场景与代价 |
| --- | --- | --- |
| 父子分块 | 用小子块检索，命中后通过父 ID 回到较大的原文段。多个子块命中同一父块时，可按父 ID 去重，同时保留各子块的证据位置。 | 适合细节藏在长章节里的资料。父块过大或命中集中在同一父块时，容易耗尽上下文预算。 [Bedrock 分层分块](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html) |
| 句子窗口 | 以单句作为检索单元，命中后补同段或同节内的前后句。 | 可补指代、前提与例外。窗口需要处理章节和表格边界，多个窗口重叠时要去重。 [LlamaIndex SentenceWindowNodeParser](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#sentencewindownodeparser) |
| 上下文前缀 | 建索引前为每块加文档标题、章节路径，或用模型生成描述该块在全文中位置与含义的短前缀。前缀和原文分别保存。 | 适合小块脱离章节后指代不明的情况。规则生成的标题路径成本较低，模型生成的前缀有额外成本，也可能引入原文未表达的说法。 [Anthropic Contextual Retrieval](https://www.anthropic.com/research/contextual-retrieval) |
| Late Chunking | 用长上下文 embedding 模型先编码长文本，得到逐 token 的表示，再按预定块边界分别池化成块向量。 | 小块向量可携带周围语境。需要能访问逐 token 表示或使用提供该能力的服务，并评估长输入和局部更新的重算成本。 [Late Chunking 论文](https://arxiv.org/abs/2409.04701) |
| 摘要树 | 将文本块递归聚类并摘要，建立多层检索节点。可先通过摘要定位主题，再查原文证据。 | 适合跨章节概览。摘要生成与更新有成本，精确尺寸、例外和受控要求应回到原文。 [RAPTOR 论文](https://arxiv.org/abs/2401.18059) |

**查询时分块（Post-chunking）。** Weaviate 的工程文章还介绍了一条不同的路径。先给文档建立较粗的检索入口，查询命中文档后再切小块，必要时缓存切分结果。它把部分入库开销推迟到查询时，但首个请求会多出切分和块级筛选的延迟。如果第一轮漏掉了文档，后面的细切也无法补回。测试时应分别记录冷缓存和热缓存延迟，并按文档版次管理缓存。这个方法改变的是切分时机。前面所述 Late Chunking 改变的是生成块向量的方式。 [Weaviate 工程博客](https://weaviate.io/blog/chunking-strategies-for-rag)

WeKnora 可以作为对照。提交 `e3fb233` 中的 `auto` 根据标题、章节等结构特征选择切分方式。启用父子模式时，子块进入常规文本索引，父块在命中后用于扩展。标题路径也可以随子块进入索引。这套实现分别解决了边界选择、索引表示和上下文补全。 [WeKnora 策略分派](https://github.com/Tencent/WeKnora/blob/e3fb2335bf3f8cadd32142380c60b2677a27bf99/internal/infrastructure/chunker/strategy.go#L283-L302) · [WeKnora 入库索引](https://github.com/Tencent/WeKnora/blob/e3fb2335bf3f8cadd32142380c60b2677a27bf99/internal/application/service/knowledge_process.go#L707-L740)

## 企业文档和图纸的选型起点

| 资料 | 可以先测试的方案 | 核对的证据 |
| --- | --- | --- |
| 制度、规程、产品手册 | 按标题切，再按句或递归分隔符限制长度。跨条款的问题可补父块。 | 条款号、适用范围、前提和例外 |
| FAQ、料号和规格条目 | 一条记录一块。精确编号和版次另设可过滤字段。 | 同义问法、近似型号、版本干扰 |
| 检验表、BOM、规格表 | 按表或行组切，附完整列头、单位与行键。跨行计算回到原表。 | 多级表头、合并单元格、跨页续表、脚注 |
| 工程图、扫描件 | 按视图、标题栏、标注、BOM 或技术要求等对象建块，保留页码和区域。空间关系题可试图像检索。 | OCR 数字和符号、引线和标注归属、版次、区域引用 |

以“图号 D-017 的 C 版中，盲孔 B 的螺纹有效深度是多少”为例，可把图号、版次和孔位存为独立字段，把“盲孔 B，螺纹有效深度 12 mm”作为检索文本，再保存原图页码、区域和对应标注作为来源。精确定位时按图号和版次过滤，再在符合条件的候选里做关键词或向量检索。这里的图号和数值是虚构示例。如果 OCR 或解析阶段已经把引线、孔位与尺寸拆散，增大重叠只能多保留邻近文字。恢复标注归属需要上游布局识别或图面关系抽取。

## 怎样比较分块方案

1. **用原文证据做标签。** 标注页码、区域、字符区间或表格行。不同方案切出的块 ID 不同，应该基于同一份原文证据比较。把问题分成单点事实、跨段条件、表格、图纸和无答案等类型。多证据问题逐项标出必需证据。
2. **单独控制每个改动。** 固定解析产物、embedding 模型、检索方式和精排配置，比较固定窗口、句段或递归、结构感知等策略。再固定切块，单独比较父块、句子窗口或上下文前缀。若结构方案必须更换解析器，单列为端到端对比。
3. **同时看候选 K 和最终 token 预算。** 在相同候选 K 下比较证据召回，在相同最终 token 预算下比较答案和引用。前 10 个短块与前 10 个长块给模型看的内容量不同。还要记录精排实际处理的候选数。
4. **同时记录质量和成本。** 看证据 Recall@K、全部必需证据覆盖率、答案与引用正确率，以及无答案时是否正确拒答。成本至少记录入库耗时、索引体积、重复片段、P95 延迟、生成输入 token 数和增量重建范围。

论文的结果可用来选对照组。Qu 等人在文档检索、证据检索和问答三个任务上比较语义切分与固定长度，收益未能稳定抵消额外计算。Bennani 与 Moslonka 的预印本用英文维基百科上的 Natural Questions、SPLADE 和 Ministral-8B 比较 0% 与 20% 重叠，在该设置下未发现可测增益。油气企业文档的一项预印本报告，结构感知分块在其专有语料上总体检索表现更好，但测试的四种纯文本方法在管道与仪表流程图上的效果都有限。企业落地应以自身资料和问题集复验。 [语义切分成本研究](https://arxiv.org/abs/2410.13070) · [Natural Questions 实验](https://arxiv.org/html/2601.14123v1) · [油气企业文档实验](https://arxiv.org/abs/2603.24556)

## 国内外工程博客与学习资料

国内资料可先读中文教程，再看实际产品如何暴露配置。

| 资料 | 类型 | 适合用来做什么 |
| --- | --- | --- |
| [Datawhale《RAG 技术全栈指南》文本分块](https://github.com/datawhalechina/all-in-rag/blob/main/docs/chapter2/05_text_chunking.md) | 中文开源教程 | 跟着样例对照字符、递归、语义和 Markdown 标题切分。可以用同一篇中文手册复现“先按标题、再递归限制长度”，观察子块是否继承标题元数据。 |
| [Zilliz 中文博客《检索增强生成的分块策略指南》](https://zilliz.com.cn/blog/guide-to-chunking-sreategies-for-rag) | 工程博客 | 快速了解固定、语义和混合分块各自在做什么，并结合自己的语料列出候选方案。博客中的尺寸建议适合作为试验起点。 |
| [Dify 中文文档《指定分段设置》](https://docs.dify.ai/zh/cloud/use-dify/knowledge/create-knowledge/chunking-and-cleaning-text) | 产品操作指南 | 在界面中对比通用模式和父子模式。着重看父段采用段落还是全文，以及子段命中后返回的内容。 |
| [阿里云百炼《切片与向量化》](https://help.aliyun.cn/zh/model-studio/rag/chunking) | 产品操作指南 | 核对按符号、正则等自定义切片入口，在切片详情中检查型号、表头和单位有没有被切散。 |

海外资料适合按“理解机制、跑通实例、设计评估”的顺序阅读。

| 资料 | 类型 | 适合用来做什么 |
| --- | --- | --- |
| [Pinecone《Chunking Strategies for LLM Applications》](https://www.pinecone.io/learn/chunking-strategies/) | 工程博客 | 从文档类型、常见查询和 embedding 输入限制出发，理解块长选择的取舍。 |
| [Weaviate《Chunking Strategies to Improve LLM RAG Pipeline Performance》](https://weaviate.io/blog/chunking-strategies-for-rag) | 工程博客与代码示例 | 对照固定、递归、结构和模型辅助方法，并了解预先切分与查询时切分的区别。 |
| [Hugging Face Cookbook《Advanced RAG》](https://huggingface.co/learn/cookbook/advanced_rag) · [《Building RAG with Custom Unstructured Data》](https://huggingface.co/learn/cookbook/rag_with_unstructured_data) | 可运行教程 | 前一篇把递归切分放在完整 RAG 流程里。后一篇从解析出的标题、列表、表格等元素出发，用按标题分块演示结构化入库。 |
| [Chroma《Evaluating Chunking Strategies for Retrieval》](https://www.trychroma.com/research/evaluating-chunking) | 团队工程实测 | 用原文标注区间评价不同切法，借鉴 token 级召回、精度和 IoU。其 IoU 在命中证据里对重复位置只算一次，分母保留重复检索 token 的成本，适合检查重叠块是否挤占预算。 |

可以先用 Datawhale 或 Hugging Face 的样例，把**同一份文档**分别按固定窗口、递归分隔符和标题切分，再按 Chroma 的思路标注原文证据。产品文档用于核对平台的实际参数，工程博客和示例用于构造可复现的对照。效果排序仍以本地语料的测量结果为准。

如果已经能在切出的块中找到完整证据，可以继续对照 [RAG 检索中的粗排与精排](retrieval-and-reranking.md)，检查它在召回、重排和上下文组装中是否被保留。
