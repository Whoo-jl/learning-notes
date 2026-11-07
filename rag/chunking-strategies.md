# RAG 分块策略详解

分块把原始资料组织成可独立检索、可以回到出处核对的单元。它影响三个结果：**证据能否被找到、一个命中包含多少完整信息、生成模型为这些证据要读多少内容**。小块有利于找到具体事实，但容易失去指代和适用条件。大块保留更多语境，但也会把无关内容带入索引和提示词。

“分块策略”在不同资料里可能指四件事。区分它们之后，就能解释为什么一个系统可以同时使用递归分块、父子块和 Late Chunking。

| 要决定什么 | 影响的环节 | 代表做法 |
| --- | --- | --- |
| 在哪里切 | 决定原文如何组成检索单元 | 固定窗口、分隔符、句段、递归、文档结构、版面、语义、LLM |
| 用什么粒度组织 | 决定资料有哪些层级、哪些单元可以搜索 | 整文、业务记录、父子层级、多粒度索引、摘要树 |
| 用什么表示去搜 | 决定索引看到哪些文字、向量或图像 | 原文、标题路径、生成前缀、命题、Late Chunking、页图 |
| 何时切、命中后带回什么 | 决定入库和查询如何分担工作 | 预分块、按需分块、邻句窗口、回父块、回整表、按需扩展 |

例如，解析器先把手册识别成章节、段落和表格，切分器再按结构与长度产出子块。索引时可以给子块补标题，命中后再回到原文父块。解析负责恢复结构，切分负责组织证据，检索负责排序和选择。工程上通常组合这些方法。

## 一、按长度、分隔符和句段切分

### 1. 固定长度与滑动窗口

把字符、词或 token 序列切成长度为 `S` 的窗口。重叠长度为 `O` 时，步长为 `S − O`，并要求 `0 ≤ O < S`。按 400 token 切、相邻块重叠 80 token，步长就是 320 token。对足够长的连续文本，块数约为无重叠时的 1.25 倍。实际索引体积还取决于向量、元数据和文本去重。

例如，某份虚构报销制度写道：“普通报销须在费用发生后 30 天内提交。因公出差可延至 60 天，但须经主管批准。”按 **18 个 Unicode 字符、零重叠**切分，得到以下各块。空格和标点也占长度。

    普通报销须在费用发生后 30 天内提
    交。因公出差可延至 60 天，但须经
    主管批准。

切分把期限和条件拆开。配置重叠有机会保住跨界内容，但如果“出差”和“主管批准”之间隔得太远，仍需关联多个块。固定窗口适合做低成本基线，特别是原始文本缺少可靠结构时。其局限是按长度前进，可能切断数值、条款或表格行。字符数适合快速控制体量，模型硬限制则应按对应 tokenizer 检查，并将标题、前缀和特殊 token 一并计入。 [Azure AI Search 分块说明](https://learn.microsoft.com/en-us/azure/search/vector-search-how-to-chunk-documents) · [Pinecone 工程博客](https://www.pinecone.io/learn/chunking-strategies/)

### 2. 单层分隔符、正则和行分块

按一种明确边界切，例如空行、记录结束标记、日志时间戳或自定义正则。短片段可继续打包，过长片段要指定兜底。相比固定窗口，这类策略依赖**格式约定**，容易理解和复现。对于问答对、日志事件、清单和规则记录，稳定的边界通常比平均字数更有用。正则要验证是否误匹配正文，以及切分后是否保留分隔符中的业务信息，例如条款编号或时间戳。行分块适合日志、列表、代码和行式表格。单行超长时，再按更细规则拆。 [阿里云百炼切片选项](https://help.aliyun.cn/zh/model-studio/rag/chunking) · [Dify 分隔符配置](https://docs.dify.ai/zh/cloud/use-dify/knowledge/create-knowledge/chunking-and-cleaning-text) · [Docling Line-Based Token Chunker](https://docling-project.github.io/docling/concepts/chunking/#line-based-token-chunker)

### 3. 按句、按段和相邻单元打包

先识别句子或自然段，再依次把相邻单元填入一个预算。它有两种常见粒度：一条句子或段落直接成为一块，或者若干完整句子或段落一起成为一块。在报销例子里，完整保留第二句，就能把“因公出差”“60 天”“主管批准”放在一起。这个单位适合手册正文、文章和转写记录。过长单句仍要处理。分句要验证中文标点、缩写、网址、编号、列表和小数，以免在 `v2.1` 或 `0.05` 中间断开。长段落可以在段内再按句打包。 [LlamaIndex SentenceSplitter](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#sentencesplitter) · [Bedrock 默认分块](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)

### 4. 递归分块

设定一组优先级从粗到细的分隔符，先尝试空行，放不下的片段再按换行、句号、空格或更细边界切，最后把相邻片段打包。递归的含义是**只继续处理仍然过大的片段**。常见实现还允许自定义长度函数、正则分隔符和目标重叠。LangChain 的通用默认序列从空行和换行退到空格与字符级，并未显式以中文句号作为一级边界。中文资料应按需要加入 `。！？；`。适合格式混杂的文本，也是验证更复杂方法的实用基线。格式损坏时可能一路退到很细的粒度，重叠也可能因片段对齐达不到设定值。 [LangChain 递归分块](https://docs.langchain.com/oss/python/integrations/splitters/recursive_text_splitter) · [Datawhale 文本分块](https://github.com/datawhalechina/all-in-rag/blob/main/docs/chapter2/05_text_chunking.md) · [Hugging Face Advanced RAG](https://huggingface.co/learn/cookbook/advanced_rag)

## 二、按文档结构、版面和业务对象切分

### 5. 按标题和层级切分

从 Markdown 标题、HTML 标题、Word 大纲或 PDF 解析出的标题中建立章节树。先决定能否跨章节，再在每节内部处理长度。给“报销例外”下的一段附上“财务制度 → 差旅报销 → 报销例外”可以保住归属，也能区分其他制度里同名的“报销例外”。需要处理层级跳级、没有标题的开头、只有标题没有正文、误识别的短标题，以及相同标题下的不同版次。小节过长时再用句段或递归切。小节很短时，是否合并应看它们是否共享语境。 [Datawhale 标题分块示例](https://github.com/datawhalechina/all-in-rag/blob/main/docs/chapter2/05_text_chunking.md) · [阿里云百炼切片标题](https://docs.agent.bailian.aliyun.com/zh/flow-agent/knowledge-base/create-and-use)

### 6. 按解析元素和版面切分

输入是解析后的文档元素流，例如标题、段落、列表、表格、图片、图注、代码块，以及元素的页码和坐标。可以将相邻同类元素打包，对表格、图注等设独立规则。Unstructured 的 `basic` 尽量填满相邻元素并隔离表格。`by_title` 引入章节边界，还能配置是否跨页以及是否合并过短的小节。其 `max_characters` 是硬上限，`new_after_n_chars` 是软目标，`overlap` 默认主要用于被拆开的超长元素，开启 `overlap_all` 才会应用到普通合并块。Docling 的 HybridChunker 先取得结构块，再按 tokenizer 拆过长块，并可合并标题和图注相同的相邻短块。结构化解析能保留来源元素，因此比先压平为纯文本更容易将证据映射回页面。 [Unstructured Chunking](https://docs.unstructured.io/open-source/core-functionality/chunking) · [Docling Chunking](https://docling-project.github.io/docling/concepts/chunking/) · [Hugging Face 结构化入库示例](https://huggingface.co/learn/cookbook/rag_with_unstructured_data)

### 7. 按物理页、区域、幻灯片或媒体时间切分

页码、幻灯片、阅读栏、图表区域、视频镜头、音频时间片和说话轮次，都可成为天然边界。按整页切最容易保留出处，但一页可能有多个主题，章节和表格也可能跨页。可以让页或幻灯片做定位父级，将页内段落、表格、文本框或图像区域单独入索引。音视频可以先转写，再按轮次、语句或时间段切文本。若使用原生多模态模型，输入可直接是按时间或视觉区域切出的媒体单元。ColPali 以整页图像为检索单元，用多向量表示页面内容。页级召回可以补充 OCR，具体答案仍要核对对应区域。某些产品的 `pages` 只是长度受限的文本片段，不能据此推断它按物理 PDF 页切。 [ColPali](https://arxiv.org/abs/2407.01449) · [Bedrock 多模态分块](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html) · [Azure Text Split](https://learn.microsoft.com/en-us/azure/search/cognitive-search-skill-textsplit)

### 8. 按表、行、字段或业务记录切分

一条 FAQ 问答、一项产品规格、一条制度条款、一条工单事件或一个 JSON 业务对象，都可作为独立块。对于价格表，一行“企业版｜120”缺少币种、周期和产品维度。入索引前可展开为“产品：企业版；月费：人民币 120 元”，并保留表名、原始行键、列头、单位与来源。长表可按行组切，重复必要列头，保留跨页续表和脚注。复杂表可同时提供整表和行级入口。跨行汇总、计数或排序往往适合走结构化查询并引用原表，避免只依据召回的几行。对于上游已经切好的独立记录，可整条入库。短且主题单一的文档也可以整文作为一块，前提是输入预算和检索粒度都合适。 [Docling 表头重复](https://docling-project.github.io/docling/concepts/chunking/#table-chunking-with-repeated-headers) · [RAGFlow 文档模板](https://github.com/infiniflow/ragflow/blob/main/docs/guides/dataset/configuration.md) · [Bedrock 自定义导入](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-custom-transformation.html)

### 9. 按代码语法切分

用语言感知的分隔符保留类和函数，或用语法解析器构建抽象语法树（AST），按模块、类、方法等节点组织块。对超长函数，可以在语句块处继续切，附上函数签名、类名、文件路径和行号。代码问答还可能需要调用方、导入或类型定义，可用关系引用补充。语法切分依赖语言支持与有效源文件。语法不完整或混合代码时，可回退到按行、标记或递归分块。 [LlamaIndex CodeSplitter 文档](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#codesplitter) · [LlamaIndex 实现](https://github.com/run-llama/llama_index/blob/main/llama-index-core/llama_index/core/node_parser/text/code.py)

## 三、按主题和语义变化切分

### 10. 基于词汇变化的主题切分

TextTiling 是传统做法。它比较一个候选边界两侧文本窗口的词汇相似度，发现相似度下降较明显的位置，再对齐到段落边界。它使用的是词汇凝聚性，适合连续的说明性文字，计算成本相对低。用词反复但主题已变时可能漏切，同义改写或中英文混用也可能制造虚假变化。窗口大小、分词、停用词和对齐规则会影响结果。 [Hearst 的 TextTiling](https://aclanthology.org/J97-1003/)

### 11. 基于 embedding 的局部语义断点

先分句，为每句或包含前后句的窗口生成向量。计算相邻向量的距离，如 `1 − cosine(v_i, v_{i+1})`。超过绝对阈值，或超过文档内部距离分布的某个百分位，就把该处作为候选切点。通常再合并过短块、拆超长块，并尊重标题或业务边界。

主要参数包括分句方式、邻句缓冲、断点阈值、最小块长和最大块长。提高距离百分位通常会减少切点，但还要看距离分布、阈值相等时的处理和长度兜底。增大邻句窗口可以降低单句波动，也可能把真正的主题变化平滑掉。它适合标题弱、话题转换明显的长文本。词义相近的两条规定可能分别适用不同产品或版本，业务边界仍需单独保留。语义分块要额外计算向量，而且结果依赖所选模型和语言。 [LlamaIndex 语义切分](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#semanticsplitternodeparser) · [Datawhale 断点计算示例](https://github.com/datawhalechina/all-in-rag/blob/main/docs/chapter2/05_text_chunking.md) · [Bedrock 语义参数](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)

### 12. 全局语义优化和非连续聚类

局部断点只比较相邻窗口。全局优化会在整个序列中寻找一组切点，兼顾块内一致性和长度预算。Chroma 的 ClusterSemanticChunker 先切短片段并生成向量，再用动态规划对连续片段分组，优化块内成对相似度。另一条路线直接聚类句子，允许把相距较远的相关句子组织在一起，有的实现会用位置距离抑制跳跃。连续方案保留阅读顺序和原文区间，非连续方案更适合做主题聚合入口，引用时应保存每个原文区间，并核查是否跨了例外、版次或适用范围。两条路线都要考虑向量成本、较大范围重算和增量更新。 [Chroma 全局分块实验](https://www.trychroma.com/research/evaluating-chunking) · [Qu 等人的聚类对照](https://arxiv.org/html/2410.13070v1)


### 13. LLM 选边界、生成规则和按文档路由

LLM 选边界的一种实现是先把文档拆成带 ID 的短段，再让模型返回切点 ID，代码据此切出原文。可验证 ID 是否存在、顺序是否单调、是否遗漏原文、结果是否满足长度上限。Chroma 的技术报告对照了直接让模型重写全文与让模型返回索引两种方式，前者在其实验中更慢，也会改写或遗漏原文。另一种方法让模型为文档生成分隔规则，再由代码执行。这里的可维护性取决于规则是否可解释、可重放，是否会随输入或模型版本变化。

“自适应”也可以在文档入口实现。先识别文档类型或检查标题密度、表格比例、段落长度等特征，再选标题、表格、句段或递归切分，失败后回退。路由既可由规则控制，也可由模型判断。Weaviate 的文章用“Agentic Chunking”描述由 Agent 按结构、密度和内容选择策略或组合，有时还补充元数据。这个名称没有统一的算法契约，读实现时应确认模型是在选切分器、选边界，还是在改写和归并内容。需要保真时，尽量让模型返回可校验的边界或原文引用。 [Chroma LLM 分块实验](https://www.trychroma.com/research/evaluating-chunking) · [Adaptive Chunking](https://arxiv.org/abs/2603.25333) · [Weaviate 工程博客](https://weaviate.io/blog/chunking-strategies-for-rag)

## 四、怎样组织层级和命中后的上下文

### 14. 父子分块和多层回取

用小子块获得较精确的匹配，父块保留更长的原文。查询先命中子块，然后按 `parent_id` 回到段落、小节、整页或短文档。多个子块如果指向同一个父块，应把父正文去重，并保存各子块的出处。父块不一定要建向量索引。有些系统用命中子块的数量或覆盖比例决定是否用父块替换，有些则总是回父块。父块过大时，可只取相关小节，或把父块作为后续回读入口。

例如，搜索“出差报销要几天内交”命中例外条款。回到同一小节，就能同时给出“通常 30 天，因公出差可延至 60 天，需主管批准”。但如果问题只问普通期限，回取整份财务制度可能挤掉更相关的证据。Dify 支持段落父块和全文父块。Bedrock 的分层检索会将命中的子块替换为父块，所以多个子块合并后，返回条数可能少于请求数。WeKnora 的父块主要用于回取上下文，常规文本索引由子块承担。 [Dify 父子分段](https://docs.dify.ai/zh/cloud/use-dify/knowledge/create-knowledge/chunking-and-cleaning-text) · [Bedrock 分层分块](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html) · [WeKnora 索引实现，提交 e3fb233](https://github.com/Tencent/WeKnora/blob/e3fb2335bf3f8cadd32142380c60b2677a27bf99/internal/application/service/knowledge_process.go#L707-L740)

### 15. 句子窗口、相邻块和按需上下文扩展

在索引里保存精细的句子或段落。命中后，用偏移、序号或相邻 ID 取前后内容。句子窗口通常固定取前后若干句，块级窗口取相邻块，按需扩展则可以在遇到定义、例外或指代时继续回读。以“该规定不适用于试用期员工”为例，命中后向前取一句，可知道“该规定”指什么。扩展应受文档、章节、版次和权限约束，多个窗口重叠时按原文区间去重。相比入库时重叠，这里把重复内容主要留到查询阶段处理。 [LlamaIndex SentenceWindowNodeParser](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#sentencewindownodeparser)

### 16. 多粒度索引与摘要树

可同时索引记录、段落、章节和文档摘要，再按查询类型选择粒度，或合并多层结果。查一个字段偏向小块，问“这份手册有哪些升级步骤”可能先查章节。多层都参与检索时，要防止同一段内容反复占据候选名额，必要时对不同层级分别设候选预算，融合后再按原文关系去重。它与“只索引子块、命中后回父块”的区别在于**哪些层级都能被搜索**。

RAPTOR 是一种生成式层级索引。它从原文块出发，递归嵌入、聚类、摘要，产生不同概括层次的节点。摘要节点适合定位主题，原文节点保留细节。因为摘要可能丢条件或概括错，精确事实和引用应回到关联原文。构建、更新和重算成本也高于单层索引。 [LlamaIndex HierarchicalNodeParser](https://developers.llamaindex.ai/python/framework/module_guides/loading/node_parsers/modules/#hierarchicalnodeparser) · [RAPTOR](https://arxiv.org/abs/2401.18059)

## 五、同一个块可以用不同方式建索引

### 17. 标题路径、上下文前缀和派生检索入口

存三类内容会更清楚：**原文正文、供检索的表示、原文出处**。标题路径、产品版本、表名等字段可直接拼进索引文本，也可作为过滤字段。Anthropic 的 Contextual Retrieval 用模型结合全文生成块级短说明，再把说明附到原块用于向量索引和 BM25。生成前缀能补指代，但也可能引入原文没有的事实，应独立保存，答案引用原文。某些系统还会把同义问题、关键词或摘要作为额外的检索入口，再把命中映射回同一原文块。派生入口越多，越要处理重复候选和更新一致性。 [Anthropic Contextual Retrieval](https://www.anthropic.com/research/contextual-retrieval)

### 18. 命题或事实单元

把段落拆解成简洁、自足的事实，再按事实建索引。例如将“这种方式需要主管批准”改写为“因公出差的报销延期需经主管批准”。它和按句切的区别在于可解析指代、补齐对象，甚至把一个句子变成多个事实。适合需要高精度定位的细节问答。重要条件、否定、量纲、版次容易在改写时丢失。工程上应保留命题到一个或多个原文区间的映射，并检查拆开的事实在引用时能否恢复限定关系。Dense X Retrieval 的结果支持在其设置下比较命题、句子和段落粒度。 [Dense X Retrieval](https://arxiv.org/abs/2312.06648)

### 19. Late Chunking

常规做法先切文本，再逐块编码和池化。Late Chunking 先用长上下文 embedding 模型编码一整段长文本，保留逐 token 向量，再按已知块边界分别池化。这样第二块里的“该服务”即使没有明说产品名，其 token 向量也可能带上前文语境。这个机制不会改变原始切分边界，也不会自动把前文添加到生成上下文。需要拿到逐 token 表示，或使用支持该机制的服务。超出长上下文窗口时仍要分段。较大的编码范围也可能让局部文档变更触发更多重算。它与 ColPali 使用的 **late interaction** 不同，后者描述查询与多个向量的匹配方式。 [Late Chunking 原文](https://arxiv.org/html/2409.04701) · [ColPali](https://arxiv.org/abs/2407.01449)

## 六、什么时候切

### 20. 预分块、查询时分块与混合方式

**预分块**在入库时切完、建好索引。查询快，边界和引用可复现，但要提前选择粒度，也要为可能很少被查的资料付出入库成本。**查询时分块（Post-chunking）**先以文档或大段检索，命中后再对少量资料切分和筛选。它允许查询时改变粒度，并可缓存结果。代价是首查延迟和两阶段遗漏风险：如果粗搜没找对文档，后面细切再精确也找不回来。缓存应考虑文档版次、切分策略和权限范围。

**混合做法**可以先为正文准备稳定的小块索引，对复杂问题再回源扩大范围或进行局部细切。它需要明确什么时候触发额外回读、哪些原文范围可以跨越，以及怎样记录切分与引用。Weaviate 的 Elysia 提供了按需分块的工程例子。 [Weaviate 分块博客](https://weaviate.io/blog/chunking-strategies-for-rag) · [Elysia 实现说明](https://weaviate.io/blog/elysia-agentic-rag)

## 七、决定分块质量的工程细节

| 决策 | 如何影响结果 |
| --- | --- |
| 字符、字节、词、token | 它们是不同计量单位。模型预算使用目标 tokenizer。原文定位还要明确是字符偏移、字节偏移、页码还是坐标，避免在中文和特殊符号处错位。 |
| 目标长度、硬上限与尾块 | 目标长度控制常见粒度，硬上限防止索引模型截断。短尾块可和同节块合并，超长行、代码、表格和公式须有兜底。保留结构与遵守硬上限发生冲突时，应记录拆分方式和原文关联。 |
| 重叠与去重 | 重叠可按字符、token、句子或完整元素保留。它适合有跨界依赖的资料，也会增加向量、索引和相似候选。检索或组装时可按原文偏移合并重复片段。重叠没有统一最佳比例。 |
| 保留分隔符和附加上下文 | 时间戳、标题、条款号可能就是证据。规范化文本时应保留它们的含义。标题、表头、前缀需要计入模型预算，并和原文内容区分。 |
| 溯源、版本和权限 | 为每块记录文档 ID、版次、原文位置、父子或相邻关系。父块、摘要、前缀和缓存要继承同一访问范围。更新原文时重建受影响的块、向量和派生入口。 |
| 三个预算 | 分开设置 embedding 的单块预算、精排的候选和单条输入预算、生成模型的总上下文预算。一个阶段能容纳的文本，后续阶段仍可能截断。 |

常见资料可以这样组合。

| 语料 | 切分与索引 | 命中后怎么处理 |
| --- | --- | --- |
| FAQ、产品目录、短工单 | 每条记录一块，实体 ID 和版本单独存 | 直接带原记录，或回读关联条目 |
| 制度、手册、网站帮助 | 按章节和条款切，长节用句段或递归，附标题路径 | 需要例外或定义时补同节窗口或父块 |
| 合同和技术规范 | 按章节与条款编号切，保留定义、交叉引用和适用范围 | 跟踪被引用条款，避免只读命中的一句 |
| 表格、报表、清单 | 按表、行或行组，重复必要表头并保留行键和单位 | 对汇总问题读整表或走结构化计算 |
| 聊天、会议、客服录音 | 按说话轮次、主题或时间段，保留时间戳和角色 | 取同议题前后轮，控制跨会话扩展 |
| 代码库 | 按函数、类、模块和文档切，带符号、路径及行号 | 补调用方、类型定义或关联文档 |
| 版式 PDF、扫描件、图纸 | 按页内元素、区域和业务对象建索引，必要时配页图索引 | 回原页或对应区域核对布局、符号与来源 |

不同产品和教程可以互相校准。Azure Text Split 演示按长度预算切且尽量保持句子。Datawhale 与 Hugging Face 展示了如何把递归切分和标题路径放进实际入库流程。Unstructured 与 Docling 提供结构元素上的规则。阿里云百炼和 Dify 给出了产品层的符号、正则、重叠、父子模式入口。Bedrock 展示固定、分层、语义及不分块的不同契约。Weaviate、Pinecone 和 Zilliz 的工程文章有助于对照成本和失败现象。Chroma 则给出了可复现的评估思路。产品参数用于理解各自实现，博客里的示例尺寸和论文里的实验结果需要保留测试条件。 [Pinecone](https://www.pinecone.io/learn/chunking-strategies/) · [Zilliz 中文工程博客](https://zilliz.com.cn/blog/guide-to-chunking-sreategies-for-rag) · [Chroma 评估](https://www.trychroma.com/research/evaluating-chunking)

## 八、怎么做有效对照

1. **从原文标证据。** 标页码、区间、表格行或代码行，避免只标某套切块的 ID。对“条件、期限、审批人”这类问题分别标必要证据。同时收集难负例，例如相似产品、旧版条款、主规则和例外。
2. **区分解析、切分、索引与检索。** 固定解析结果、embedding、混合检索和精排配置，再改切分。结构方案若必须更换解析器，应另做端到端对比。测试回父块、前缀或 Late Chunking 时尽量固定其他变量。
3. **同时控制候选数和上下文量。** 在相同 K 下比较证据 Recall@K 与多证据全部覆盖率。在相同生成输入 token 预算下比较答案和引用。还可测试相同延迟或成本预算。只看相同 Top K 时，长块和短块让模型读到的内容量不同。
4. **看证据密度和完整性。** Chroma 按原文 token 对齐证据与取回文本。其 IoU 计算会让重复取回的 token 体现为成本。也应查关键例外是否齐全、上下文中重复内容占多少、无答案时是否正确拒答。 [Chroma token 级指标](https://www.trychroma.com/research/evaluating-chunking)
5. **测实际开销。** 记录解析与建索引耗时、模型调用量、块数、索引体积、精排输入量、P95 延迟、生成 token 数和增量重建范围。查询时切分要分别测冷缓存和热缓存。

公开结果可以帮助挑基线。Qu 等人在文档检索、证据检索和问答中比较固定句数、语义断点和聚类，语义方法没有稳定获得足以抵消成本的提升。Bennani 与 Moslonka 在英文维基百科的 Natural Questions 短问答上，用 SPLADE 和 Ministral-8B 对比了 0% 与 20% 重叠，在其设置下未发现可测增益。油气企业文档预印本报告结构分块在其专有语料上的整体检索优势，同时四种文本方法在管道与仪表流程图上的效果均有限。这些结论可作为试验假设，中文、代码、表格和本地数据的结果仍需分别验证。 [语义切分对照](https://arxiv.org/html/2410.13070v1) · [Natural Questions 实验](https://arxiv.org/html/2601.14123v1) · [油气企业文档研究](https://arxiv.org/abs/2603.24556)

<a id="口述版与常见追问"></a>

## 九、回顾与问答

**一分钟怎么介绍分块策略？**

> 分块决定系统能以什么粒度找到证据。可以按长度、分隔符、句段、文档结构、业务对象或语义变化切。选边界时先看证据的最小完整单位，再看模型预算。还要分开设计索引块与回答上下文。可以用小块找细节、用标题路径补检索线索，命中后回父块或邻句。比较策略时，我会固定模型和候选数，使用同一份原文证据，并在相同上下文预算下比较答案、引用和成本。

**固定窗口、句段和递归分块怎么区分？** 固定窗口按长度前进。句段分块先识别完整句子或段落再打包。递归分块为分隔符排优先级，对过长的片段再用更细规则。三种做法都可以限制长度，选择边界的机制不同。

**结构分块和语义分块可以一起用吗？** 可以。例如先保留文档章节、表格和代码边界，再对没有明显段落结构的长正文尝试语义断点。这样能在允许切分的范围内找主题变化，同时保护不该跨越的对象和来源边界。

**父子块、重叠和句子窗口分别解决什么？** 重叠把跨界文字预先复制进索引块。父子块建立小块到大段的关联。句子窗口在命中后补邻句。前者增加入库冗余，后两者主要改变命中后给模型的上下文。

**语义分块、上下文前缀和 Late Chunking 分别改变什么？** 语义分块决定断点。上下文前缀改变块的索引文字。Late Chunking 让逐 token 表示先吸收更长语境，再聚合为小块向量。这三个选择可以组合。

**能把块切得更小再把 Top K 开大吗？** 小块可以提高定位粒度，但一个事实被拆成几个块后，需要这些证据同时被召回并正确合并。开大 K 会增加候选、精排成本和噪声。应先保证关键条件能作为一条证据或可追踪的关联被保留，再用召回和上下文预算决定 K。

**回答出错时怎样区分是分块还是排序？** 先检查解析文本中是否有正确证据，再查索引块是否保留对象和条件。如果证据存在于可检索块，检查它是否进入粗排候选、是否被精排保留、是否完整进入生成上下文。用这条链路定位问题，可以避免用更大的 K 掩盖解析错误。 [RAG 检索中的粗排与精排](retrieval-and-reranking.md)
