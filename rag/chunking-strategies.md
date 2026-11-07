# RAG 分块策略与选型

检索增强生成（RAG）里的 **chunk**，是从资料中整理出来、可以被系统单独找到的一段内容。分块的目标是让系统能用较小的代价找到**完整、可回溯的证据**。块太大，一条具体要求容易被其他主题稀释。块太小，系统可能找到一个数字，却看不出这个数字说的是哪个对象、什么属性、什么条件。

理解分块可以沿着一条问题往下走：**原文怎样切？切完拿什么去搜？命中后给模型看什么？** 这三个决定可以独立调整。下面用同一份虚构图纸说明，最后有一段可以直接口述的答案和自测题。

## 从一条图纸要求开始

假设我们有一张虚构图纸，图号 **D-017**、版次 **C**。第 2 页“孔加工”小节的解析文本如下。

> 底座的盲孔 B 为 M8，螺纹有效深度为 12 mm，底孔钻深为 16 mm。螺纹有效深度自端面 S 起算。此要求仅适用于热处理后的零件。

如果问“盲孔 B 的螺纹有效深度是多少”，第一句就含有直接证据。如果问“从哪里起算，什么条件下适用”，就要把三句话关联起来。`16 mm` 也在原文里，但它是**底孔钻深**。把两个数值分清，比找到“有深度、有数字”的一段话更重要。

这条问答经过三个不同的环节。

| 环节 | 在这个例子里做什么 |
| --- | --- |
| 解析 | 从图纸提取图号、版次、阅读顺序和文字。能定位时，保留页码和标注区域。 |
| 分块与建索引 | 决定以句子、小节、表格行还是图纸对象作为候选。可以给索引文本附上文档标题和小节路径。 |
| 检索与组装上下文 | 找到含有 `12 mm` 的块，再根据问题补端面和热处理条件，把可核对的原文和出处交给模型。 |

若解析结果把 `12 mm` 读成 `72 mm`，或者丢了标注指向哪个孔的信息，调整块长、Top K 或精排都无法可靠恢复原始证据。此时要回到 PDF 页图、区域检测或 OCR 去排查。

## 分块边界怎样影响证据

### 固定窗口与重叠

固定窗口按字符或 token 数前进。用这段正文做一个故意很小的例子，按 **22 个 Unicode 字符、无重叠**切分，空格和标点也计入长度。每行是一个块。

    底座的盲孔 B 为 M8，螺纹有效深度为 1
    2 mm，底孔钻深为 16 mm。螺纹有效深
    度自端面 S 起算。此要求仅适用于热处理后的
    零件。

在这个设置下，`12 mm` 被拆成了 `1` 和 `2 mm`。召回两块也只是重新看到了两段片段，生成端还要知道它们原本连续。把重叠设成 8 字符后，步长变为 14。第二块变成“纹有效深度为 12 mm，底孔钻深为 16 ”，保住了数值和单位，但已经看不到“盲孔 B”。这说明重叠可以缓解边界切断，却不能保证一个事实的所有成分都落在同一块。

固定窗口适合做快速且可复现的基线。块长为 `S`、重叠为 `O`，步长就是 `S − O`。例如 400 token 配 80 token 重叠，步长 320 token。对足够长的连续文本，块数约为无重叠时的 1.25 倍。实际索引和推理开销还要看文本、索引与模型。如果目标是遵守 embedding 输入上限，用该模型的 tokenizer 计数，并给标题等附加内容留空间。中文的字符数和 token 数不能直接互换。 [Azure AI Search 分块说明](https://learn.microsoft.com/en-us/azure/search/vector-search-how-to-chunk-documents) · [Pinecone 工程博客](https://www.pinecone.io/learn/chunking-strategies/)

### 按句段、递归分隔符与文档结构切分

**按句或段落打包**先识别句子或段落，再把相邻单元放入预算。例子里的第一句长 39 个字符，预算容得下时，孔位、两个深度和单位能完整留下。第二、第三句也可能被打包到一起。若预算连第一句都放不下，系统仍要在保留长句和继续切分之间做取舍。小数、缩写、料号和中英文标点也会影响分句效果。 [LlamaIndex SentenceSplitter](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#sentencesplitter)

**递归分隔符**处理的是不同粒度的边界。先尝试空行，超长时再尝试换行、句号或其他分隔符，最后用更细的规则兜底。比如文档有多段文字，只有“孔加工”这一段太长，就只细切这一段，其余短段保持原状。适合纯文本和结构不统一的 Markdown。中文标点需要按语料配置。 [LangChain 递归分块](https://docs.langchain.com/oss/python/integrations/splitters/recursive_text_splitter) · [Datawhale 中文教程](https://github.com/datawhalechina/all-in-rag/blob/main/docs/chapter2/05_text_chunking.md)

**结构与版面分块**先知道“这是标题、正文、表格还是图片”，再决定怎么切。假如“孔加工”和“表面处理”是两个小节，可以先按标题分开，再在各节内部限制长度，让子块继承所属标题。如果数据是一张表，优先保留表名、列头、行键、单位和有约束作用的脚注。一条 BOM 记录可以单独建块，跨行汇总应回读原表。对于图纸，可以按视图、标题栏、标注或技术要求建块，附上页码和区域。**结构必须先由上游识别**，切块器才能利用它。 [Docling](https://docling-project.github.io/docling/concepts/chunking/) · [Unstructured](https://docs.unstructured.io/open-source/core-functionality/chunking) · [Hugging Face 结构化入库教程](https://huggingface.co/learn/cookbook/rag_with_unstructured_data)

**怎么讲清差异。** 按句段打包把“句子”当基本单位。递归分块把“分隔符优先级”当规则。结构分块把“文档元素及其关系”当约束。它们可以组合。先按标题确定章节范围，再用递归分隔符控制章节内的块长，就是一种常见组合。文档短且已经是一条独立记录时，也可以直接整条建索引，比如一对 FAQ 或一条料号记录。 [RAGFlow 文档模板](https://github.com/infiniflow/ragflow/blob/main/docs/guides/dataset/configuration.md) · [Bedrock 分块与不分块选项](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)

### 语义断点与模型辅助

**Embedding 语义分块**先分句，再把每句或带邻句的小窗口编码成向量，比较相邻窗口的距离。距离高，表示话题可能发生变化，可作为切点。若“孔加工”后面紧接一段运输包装说明，语义差异可能帮助找到边界。反过来，“螺纹有效深度”和“底孔钻深”在语义上很接近，却是业务上必须区分的字段。语义断点帮助识别主题变化，图号、孔位、尺寸类型和适用条件仍需要原文结构或字段来承载。常见参数有邻句窗口、断点阈值和块长上限。按距离百分位切分时，提高阈值通常会减少切点。 [LlamaIndex 语义切分](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#semanticsplitternodeparser) · [Bedrock 语义分块](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)

**模型辅助切分**可以让 LLM 提议从哪些段落断开，由代码按原文位置切出块。**命题索引**则进一步把原文整理成能独立理解的事实，例如生成“D-017 C 版盲孔 B 的螺纹有效深度自端面 S 起算”。前者选择边界，后者改变了被索引的文本。要保留命题到原文的对应关系，尤其要核查模型是否漏掉否定、例外和适用条件。复杂方法增加入库开销，应和简单分块用同一数据做对照。 [Adaptive Chunking](https://arxiv.org/abs/2603.25333) · [Dense X Retrieval](https://arxiv.org/abs/2312.06648)

## 检索块和回答上下文怎样配合

即使按句子切得很好，用第二句做检索时，向量只看到了“螺纹有效深度自端面 S 起算”，没有盲孔 B。这里有两个可分别处理的问题。一是让索引知道这句话属于哪份文档、哪个对象。二是找到它以后，再取多少原文来支撑答案。

| 方法 | 如何应用到这条问题 | 作用发生在哪一步 |
| --- | --- | --- |
| 标题路径或上下文前缀 | 给第二句的索引文本加上“D-017 C 版 → 孔加工 → 盲孔 B”。可用可靠的结构字段生成前缀，也可用模型生成后核对。原文与前缀分开保存。 | **索引前**补充检索线索。 [Anthropic Contextual Retrieval](https://www.anthropic.com/research/contextual-retrieval) |
| 句子窗口 | 第二句命中后，取同节的前一句和后一句，恢复孔位、数值和限制条件。窗口不要随意跨小节或图纸版本。 | **命中后**扩展邻近原文。 [LlamaIndex 句子窗口](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#sentencewindownodeparser) |
| 父子分块 | 用小子块找到有效深度，用父 ID 回到“孔加工”小节。多个子块命中同一父块时可只放入一次，并保留每条证据的位置。父块过大时，可只取与命中相关的段落。 | **入库时**建立层级，**命中后**取父块。 [Dify 父子分段操作](https://docs.dify.ai/zh/cloud/use-dify/knowledge/create-knowledge/chunking-and-cleaning-text) · [Bedrock 分层分块](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html) |
| Late Chunking | 先用长上下文 embedding 模型编码整个小节或更长文本，再按已定边界分别聚合每块的 token 表示。第二句的向量因此有机会携带前文语境，检索返回的原文仍可保持小块。 | **向量生成时**引入长文本语境，需要相应的模型或服务支持。 [Late Chunking 论文](https://arxiv.org/abs/2409.04701) |

还有两条容易混淆的路线。**查询时分块（Post-chunking）**先粗搜文档，命中后才对该文档细切。它减少未被访问文档的前置切分开销，但首次查询多了切分成本，而且第一轮必须找对文档。**摘要树**如 RAPTOR，把小块逐层聚类和摘要，再索引不同层级，用来发现跨章节主题。它的父节点可以是生成的摘要，父子分块中的父块通常是较长的原文。工程图、扫描件还可以走页图或区域图像检索，例如 ColPali 的研究以页图为检索对象。图像命中之后，仍要定位具体视图或标注。 [Weaviate 工程博客](https://weaviate.io/blog/chunking-strategies-for-rag) · [RAPTOR](https://arxiv.org/abs/2401.18059) · [ColPali](https://arxiv.org/abs/2407.01449)

**对照 WeKnora。** 在已核对的提交 `e3fb233` 中，`auto` 会根据标题、编号等规则选择 Go 切分器。启用父子模式后，子块进入常规文本索引，父块用于回取上下文。标题路径可随子块入索引。在图纸例子里，想改变被搜到的片段有多大，先看 `child_chunk_size`。想改变回取的上下文有多大，再看父块和后续合并。Top K 控制候选数量。 [WeKnora 策略分派](https://github.com/Tencent/WeKnora/blob/e3fb2335bf3f8cadd32142380c60b2677a27bf99/internal/infrastructure/chunker/strategy.go#L283-L302) · [父子配置](https://github.com/Tencent/WeKnora/blob/e3fb2335bf3f8cadd32142380c60b2677a27bf99/internal/infrastructure/chunker/strategy.go#L225-L252) · [索引实现](https://github.com/Tencent/WeKnora/blob/e3fb2335bf3f8cadd32142380c60b2677a27bf99/internal/application/service/knowledge_process.go#L707-L740)

## 怎样选方案和证明有效

给不同文档选与证据形态相符的起点。纯文本可从按句段打包或递归开始。手册与制度可按章节切，再保留条款号和例外。表格按记录或行组切，携带列头和单位。工程图先查图号、版次和标注归属，再决定按对象建索引，还是加页图或区域检索。精确字段适合先过滤，例如限定图号 D-017 和 C 版，再在范围内找孔 B。

评估时先在**同一份原文**标注证据。对上面的多条件问题，分别标注深度、起算面、热处理条件。不同方案会产生不同 chunk ID，比较时用原文区间、表格行或图纸区域作为共同基准。可以让实验回答三个问题。

1. **候选里有没有证据？** 固定解析产物、检索模型和候选 K，比较固定窗口、递归和结构分块。看证据 Recall@K，多条件题再看必需证据是否齐全。
2. **带给模型的证据够不够？** 固定切块，对照原块、句子窗口、父块等上下文策略。在相同最终 token 预算下比较答案和引用。只固定 Top K 会让长块方案实际送入更多文本。
3. **收益值不值额外开销？** 记录索引体积、入库时间、重复片段、重排候选数、P95 延迟和实际上下文 token 数。Chroma 的 token 级 IoU 提供一个思路，把命中的原文证据与取回内容对照，并让重复取回的 token 体现为额外开销。 [Chroma 实测与方法](https://www.trychroma.com/research/evaluating-chunking)

公开实验可以帮助选对照组。Qu 等人的实验没有发现语义分块相对固定分块的收益能稳定抵消额外计算。Bennani 与 Moslonka 用英文维基百科短问答、SPLADE 和 Ministral-8B 对比 0% 与 20% 重叠，在这组设置下未发现可测增益。油气企业文档研究报告，结构分块在其语料上整体检索更好，但比较的四种文本分块在管道与仪表流程图上的效果都有限。它们适合用来提出待验证的问题，图纸和中文资料的结果仍需要自己的数据。 [语义分块成本实验](https://arxiv.org/abs/2410.13070) · [Natural Questions 实验](https://arxiv.org/html/2601.14123v1) · [油气企业文档实验](https://arxiv.org/abs/2603.24556)

## 口述版与常见追问

**如果只用一分钟讲 RAG 分块**

> RAG 分块是把资料组织成可以单独检索的证据单元。先看原文是什么形态，再选边界。纯文本可以按句段或递归分隔符切，手册适合按标题和条款切，表格和图纸要保留表头、对象、单位和来源位置。块小有利于找到细节，但容易丢上下文，可以给索引块加标题或对象信息，命中后补父块或邻句。块长决定一条候选有多大，Top K 决定有多少条候选。选方案时，我会在相同候选数和上下文预算下比较完整证据覆盖、引用正确率和成本。

**为什么不能把 chunk 越切越小？** 小块更聚焦，但可能只剩“12 mm”，缺了孔位、深度类型和限定条件。还可能让同一问题需要召回多个块，增加索引条目和候选数。切到什么粒度，要看最小的完整证据是什么。

**重叠和父子分块都能补上下文，有什么差别？** 重叠在入库时复制相邻文字，影响索引内容和块数。父子分块建索引时用小块，命中后按父 ID 取较长原文。前者通常缓解边界问题，后者可以按需回取较远的前提或例外。

**语义分块、上下文前缀和 Late Chunking 的区别是什么？** 语义分块用相邻句的向量距离**决定哪里切**。上下文前缀给已经切好的块**加索引文字**。Late Chunking 先编码长文本，再按既定边界聚合，改变的是**块向量怎么生成**。

**候选已经找到 `12 mm`，回答为什么还会错？** 先核对命中的位置能否证明“盲孔 B 的螺纹有效深度”，再检查前提和起算面有没有一起进入最终上下文。如果原始证据齐全，继续看精排、上下文截断和生成。若只搜到了值或召回了底孔钻深，先检查分块和索引表示。 [粗排与精排笔记](retrieval-and-reranking.md)

**怎样判断结构分块在我的图纸库里值得做？** 用同一批图纸标注范围、尺寸和适用条件。先和递归分块在相同候选 K、相同上下文预算下做对比，再比较区域引用正确率和入库成本。若为结构方案换了更好的解析器，把这组结果单列为端到端对比。

**块长和 Top K 应该怎么一起调？** 先确定证据的最小完整单位和 embedding 输入预算，比较几档块长。再看正确证据在候选列表里的位置，决定要给粗排和精排多少候选。块长翻倍时，同一个 Top K 可能带来更多上下文 token 和重复信息。应该同时看证据覆盖和实际输入长度。

### 两道自测题

1. 在示例里，假设检索只命中“螺纹有效深度自端面 S 起算”，你能回答什么？为了回答完整问题，还要找哪些证据？
2. A 方案检索回 10 个短块，B 方案检索回 10 个父块，B 的答案更完整。你还要做什么对照，才能判断是切分边界更好，还是模型仅仅看到了更多原文？

<details>
<summary>展开参考回答</summary>

1. 命中句给出起算面 S。还要用来源确认它属于 D-017 C 版的盲孔 B，并补回 12 mm 和热处理后的适用条件。只拿命中句时，不应把未找到的部分说成已证实。
2. 在同一份原文证据上比较两套边界，固定模型、检索方式、候选 K 和上下文 token 预算。再固定小块索引，单独比较返回原块与回父块。第一组检查边界对召回的影响，第二组检查回取上下文的影响。

</details>

## 按学习任务找资料

- **动手看块长和边界。** 用 [Datawhale 中文教程](https://github.com/datawhalechina/all-in-rag/blob/main/docs/chapter2/05_text_chunking.md) 或 [Hugging Face Advanced RAG](https://huggingface.co/learn/cookbook/advanced_rag)，把同一段中文文字分别按固定窗口和递归规则切开，检查哪一块能独立证明答案。
- **看产品里怎样配置。** 用 [Dify 父子分段指南](https://docs.dify.ai/zh/cloud/use-dify/knowledge/create-knowledge/chunking-and-cleaning-text) 对照段落父块与全文父块。用 [阿里云百炼切片说明](https://help.aliyun.cn/zh/model-studio/rag/chunking) 对照符号、正则和切片预览。
- **看不同方案的取舍。** [Zilliz 中文工程博客](https://zilliz.com.cn/blog/guide-to-chunking-sreategies-for-rag) 与 [Weaviate 英文工程博客](https://weaviate.io/blog/chunking-strategies-for-rag) 适合并读。记下每种方法解决的是边界、索引表示还是切分时机，然后用 [Chroma 的评估方法](https://www.trychroma.com/research/evaluating-chunking) 设计自己的证据标注。
