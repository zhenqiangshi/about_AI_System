

![[Pasted image 20260702104840.png]]

整个系统被逻辑切分为三个核心流水线：**离线数据打通**、**在线检索生成**，以及**持续测试与评估**。

### 第一阶段：离线数据流水线 (Data Pipeline)

> **目标：** 将非结构化文档转化为高检索质量的向量与排版特征，存入数据库。

**节点 1：文档智能==解析== (Data Ingestion)**

- **技术要点：**
    
    - **版面分析 (Layout Analysis)：** 识别 `Heading`、`List`、`Table` 和 `Image`，避免将跨页的破碎文本或页眉页脚强行拼接。
        
    - **多模态提取：** 将表格强制转化为 Markdown/HTML；将图片通过 VLM（如 Qwen2.5-VL）生成 Caption。
        
- **主流工具：** [[Marker]]、MinerU、Unstructured、[[OCR]][[Docling & opendataloader-pdf]][[PaddleOCR]] 、[[MarkItDown]]
    

**节点 2：语义感知切分 (Advanced Chunking)**

- **技术要点：**
    
    - **父子块组织 (Parent-Child Chunking)：** 将文档切分为大块（Parent，约 1000 Tokens）和小块（Child，约 200 Tokens）。检索时精确匹配小块，送给 LLM 时提供大块以保留完整上下文。
        
    - **滑窗限制：** 采用带重叠区（Overlap，通常为 $10\% \sim 20\%$）的切分，防止关键实体被切断。
        
- **主流工具：** 自定义 Python 脚本结合 `tiktoken` 精确控制边界，或 LangChain 的结构化切分器。
    

**节点 3：多路索引与向量化 (Embedding & Indexing)**

- **技术要点：**
    
    - **稠密向量 (Dense)：** 提取深层语义（如 BGE-M3, OpenAI `text-embedding-3`）。
        
    - **稀疏向量 (Sparse)：** 提取精确词频特征，弥补模型对专有名词、特定型号不敏感的缺陷（如 SPLADE 模型或 BGE-M3 稀疏模式）。
        
    - **多向量预计算 (Optional)：** 如果使用离线 ColBERT 方案，在此阶段直接生成 Token 级别的矩阵索引。
        
- **主流工具：** Qdrant、Milvus 2.4+（原生支持 Dense + Sparse 混合存储）。[[RAGFlow细节]][[Embeding Model]]
- **KG:** [[RAG+KG]]

| Method  方法/方式            | Strength  力量/实力                                      | Trade-off  权衡/取舍                                    | Best For  最适合……的情况/用途                                     |
| ------------------------ | ---------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------------- |
| **Fixed-Size  固定大小**     | Simple, predictable chunks  <br>简单、易于预测的内容/结构        | Ignores structure, breaks meaning  <br>无视结构，破坏了其意义。 | Raw or unstructured text  原始或非结构化文本                       |
| **Sentence  句子/语句**      | Preserves complete thoughts  <br>能够完整地保留原有的思想/内容     | Inconsistent sizes  尺寸不一致                           | RAG, Q&A systems  RAG、问答系统                                |
| **Paragraph  段落/段落文字**   | Aligns with semantic units  <br>与语义单元保持一致/与语义单位相对应   | Large variance in length  长度差异很大                    | Docs, manuals, instructional content  <br>文档、手册、教学内容      |
| **Sliding Window  滑动窗口** | Maintains full context  保留了所有上下文信息                   | Redundant, compute-heavy  冗余的、计算成本过高的系统/架构          | Reranking, high-recall retrieval  <br>重新排序，高召回率的检索结果      |
| **Recursive  递归的**       | Flexible, handles messy input  <br>具有灵活性，能够处理混乱的输入数据 | Heuristic, sometimes brittle  <br>启发式方法，有时不够稳健/容易出错 | Scraped web content, mixed sources  <br>从不同来源获取的网页内容合并后使用 |
| **Semantic  语义的/与语义相关的** | High-quality, meaning-aware  <br>高质量、具备语义理解能力        | Slower, resource-intensive  <br>速度较慢，对资源要求较高        | Legal, research, critical QA  <br>法律、研究、关键质量把控            |
### 第二阶段：在线检索生成流水线 (Serving Pipeline)

> **目标：** 毫秒级响应用户提问，完成意图理解、精准召回、重排打分及最终生成。

> ==可分层、分角色进行细化主题的内容查询==

**节点 4：用户查询改写 (Query Transformation)**

- **技术要点：**
    
    - **指代消解：** 补全多轮对话中的代词（如把“它多少钱”重写为“华为Mate70多少钱”）。
        
    - **假设性文档生成 (HyDE)：** 针对晦涩问题，让 LLM 先“盲猜”一个答案，用假答案的向量去库中进行高维映射召回。
    - 
        备注：doc2query  关键词延伸 ~
- **主流模型：** 7B 级别的高速小模型（如 Qwen2.5-7B-Instruct），确保延迟在 100ms 内。
    

**节点 5：混合检索与粗召回 (Hybrid Retrieval)**

- **技术要点：**
    
    - **两路并发：** 同时发起 Dense 语义检索和 Sparse 关键词检索，召回 Top-K（如 100）个候选切片。
        
    - **倒数秩融合 (RRF)：** 将两路不同维度的得分通过算法合并排序。
        
    - **元数据硬过滤：** 在检索底层直接植入权限标签或时间戳过滤，防止数据越权越界。
    - [[Embeding Model]]
        

**节点 6：重排 (Reranking)**

- ###### 1. 双塔模型 (Bi-Encoder / Representation-based)

- **工作原理：** Query（问题）和 Document（文档）分别各自通过一个模型，并且各自被“强行压缩”成**唯一的一个**一维向量。最后只计算这两个单一向量的余弦相似度。
    
- **优点：** 速度极快。文档的向量可以离线算好存起来，线上只算 Query 向量。
    
- **缺点：** **信息丢失严重。** 无论文档是一句话还是 500 个字，都被压缩成同一个维度的向量，细粒度的词汇特征（比如特定的数字、专有名词）很容易在压缩过程中被抹除。
    
- **代表模型：** OpenAI `text-embedding-3`、BGE-Large。_(注：严格来说，这是检索/召回层的标配，但也有简易系统拿它直接做微调来当 Rerank)。_
    

- ###### 2. 交叉编码器 (Cross-Encoder) —— 传统的 Rerank 霸主

- **工作原理：** 将 Query 和 Document 直接拼接在一起（例如：`[CLS] 问题 [SEP] 文档 [SEP]`），作为一个整体输入进 Transformer 模型。模型在每一层都会让问题里的每一个字和文档里的每一个字进行“全量注意力交互（Full Self-Attention）”。
    
- **优点：** **准确率极高。** 因为模型从一开始就能同时看到问题和文档，能深度理解它们之间的语义关联。
    
- **缺点：** **计算延迟极高，且无法预计算。** 线上需要实时把 Query 和 100 个长文档拼接跑 100 次大模型推理，并发稍微一高，GPU 就会成为瓶颈。
    
- **代表模型：** BGE-Reranker、Cohere Rerank。
    

- ###### 3. 延迟交互 (Late Interaction)
- **工作原理：** 它是双塔和交叉编码器的**折中方案**。Query 和 Document 依然分开独立编码（像双塔），但是**不进行最终的压缩**，而是保留每一个词（Token）的向量。最后在模型外面，用一个极其简单的矩阵点积（MaxSim算子）让两组词向量矩阵进行交叉匹配。
    
- **优点：** 既保留了词级别的细粒度特征（准确率逼近 Cross-Encoder），又避免了全量拼接推理带来的庞大计算量（速度远快于 Cross-Encoder），而且文档的 Token 矩阵可以离线存入数据库。
    
- **代表模型：** ColBERT、ColPali。
    

- ###### 4. 大模型直接重排 (LLM-based Reranking / Prompt-based)

- **工作原理：** 不依赖专门的文本排序模型，而是直接写一段 Prompt，把问题和几十个文档一股脑塞给类似 GPT-4o 这样的大语言模型，让它作为裁判，输出一个排序列表。
    
- **优点：** 零样本（Zero-shot）推理能力极强，不需要针对特定垂直领域微调就能有很好的效果。
    
- **缺点：** 极度依赖大模型的 Context Window，成本极高（Token 消耗大），而且可能会受到大模型“Lost in the Middle（忽略中间信息）”缺陷的影响。
    
- **代表模型：** RankGPT 方案、各类闭源大模型。

- **技术要点：**
    
    - **核心机制：** 引入 **ColBERT**（纯文本）或 **ColPali**（针对 PDF 视觉排版）进行细粒度重排。不压缩全文语义，而是计算 Query 与 Chunk 之间 Token 级别的 `MaxSim` 得分。
    - **Rerank：模型选择、自定义rerank函数**
        
    - **降噪与提纯：** 从 100 个粗排候选集中，精准抠出最相关的 Top-N（如 5）个 Chunk。
        
- **主流模型：** Jina-ColBERT、ColBERTv2、ColPali。

***必要时增加对业务数据的理解，自定义Rerank逻辑***

**节点 7：上下文编排与生成输出 (Prompting & Generation)**

- **技术要点：**
    
    - **信息熵压缩 (Context Compression)：** 剔除 Chunk 中的无用助词，降低干扰并节省 Token（如 LLMLingua 方案）。
        
    - **防迷失站位：** 将最相关的 Chunk 放在 Prompt 的开头和结尾，相关度较低的放中间。
        
    - **流式输出与护栏：** 引导大模型输出最终答案，并在最后一道关卡部署 Guardrails 过滤敏感信息或拦截越狱攻击 (Prompt Injection)。
        

### 第三阶段：持续测试与监控流水线 (Evaluation & LLMOps)

> **目标：** 量化系统表现，建立基准测试集，告别“凭感觉调参”。[[检索器评估指标]]

**节点 8：自动化测试与评估 (RAG Evaluation)**

- **技术要点：**
    
    - **黄金测试集构建 (Golden Dataset)：** 利用大模型反向阅读私有知识库，自动生成海量的 `(Query, Context, Answer)` 问答对作为考卷，并刻意掺入干扰项（Hard Negatives）。
        
    - **RAG 三要素打分 (LLM-as-a-Judge)：** 引入强力大模型（如 GPT-4o）作为裁判，独立评估三个维度：
        
        1. **上下文相关性：** 检索出来的文档是不是垃圾？（测召回/重排）
            
        2. **事实一致性：** 生成的回答是不是胡编乱造的？（测幻觉率）
            
        3. **回答相关性：** 回答是不是用户想问的？（测 Prompt/改写质量）
            
- **主流工具：** Ragas、TruLens、Arize Phoenix。








引用：

Building a RAG Pipeline for 10M+ Documents With Near-Zero Hallucination:
https://medium.com/gitconnected/building-a-rag-pipeline-for-10m-documents-with-near-zero-hallucination-788e4b5b7f25