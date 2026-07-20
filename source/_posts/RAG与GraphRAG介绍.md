---
title: RAG与GraphRAG介绍
date: 2026-03-25 23:30:02
categories:
  - 技术
tags:
  - Python
---

## RAG

检索增强生成（Retrieval-Augmented Generation，简称RAG）是当前大模型落地场景中解决幻觉、知识滞后、隐私数据调用三大核心痛点的主流技术方案。它通过外部知识库检索+大模型生成的组合模式，让大模型输出更贴合事实、更贴合私有数据的内容。RAG的工作流程可细分为以下5个步骤：

1.  数据预处理（chunking）
1.  向量嵌入（Embedding）
1.  索引构建与向量入库
1.  检索召回
1.  重排序（Reranking）
1.  上下文拼接与生成

<!---->

1.  ### 数据预处理（chunking）

企业文档通常形式多样，RAG的第一步就是把各种格式的资料整理成统一文本结构。企业文档常见形式包括PDF、图片、PPT、Word、网页等，在面对不同格式的文档时，通常采用不同的办法。例如，对于PDF或Word格式的文档，通常使用OCR（文字识别）转化为**层次结构分明**的markdown文档。如果文档中表格、图片居多，还要选择具有版面识别、图像推理的OCR模型。

得到处理好的文本之后，先剔除文档中的乱码、空白等无效内容；再进行分块（chunking）。主流框架LangChain提供了开箱即用的分块函数，支持**固定长度、语义、结构化（按标题）** 等多种策略，分块大小通常控制在200-1000字符，兼顾检索精度与上下文连贯性。分块时，还需要考虑为每个文本块补充来源、页码、关键词等元数据，便于后续溯源与过滤。

代码示例：

```
markdown_document = "# Foo\n\n    ## Bar\n\nHi this is Jim\n\nHi this is Joe\n\n ### Boo \n\n Hi this is Lance \n\n ## Baz\n\n Hi this is Molly"

headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]

markdown_splitter = MarkdownHeaderTextSplitter(headers_to_split_on)
md_header_splits = markdown_splitter.split_text(markdown_document)
md_header_splits

# 输出内容
[Document(metadata={'Header 1': 'Foo', 'Header 2': 'Bar'}, page_content='Hi this is Jim  \nHi this is Joe'),
 Document(metadata={'Header 1': 'Foo', 'Header 2': 'Bar', 'Header 3': 'Boo'}, page_content='Hi this is Lance'),
 Document(metadata={'Header 1': 'Foo', 'Header 2': 'Baz'}, page_content='Hi this is Molly')]
```

2.  ### 向量嵌入（Embedding）

向量的本质是用数值序列表征文本的语义信息，语义相近的文本，向量空间中的余弦相似度更高。这一步的核心是保证嵌入模型与下游检索、生成任务的适配性，垂直领域（医疗、法律、金融）建议选用领域微调后的嵌入模型，提升语义表征精度。

常见的Embedding模型有：**BGE系列（例如bge-m3）、Qwen系列（例如text-embedding-v1）、OpenAI 系列（例如text-embedding-3-large）** 。

代码示例：

```
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(
    model="text-embedding-3-large",
    # dimensions=1024
)
```

3.  ### 索引构建与向量入库

将生成的向量与对应的原始文本块、元数据一并存入向量数据库（如Milvus、Chroma、Pinecone）。在实际系统中，通常会同时保存两种类型的向量表示：

-   **稠密向量（Dense Vector）：** 稠密向量由Embedding模型生成，例如 OpenAI Embedding 或 BGE。这类向量的维度是固定的，例如 1536 维。模型会根据上下文计算语义表示，因此即使查询语句与文档中的词语不完全一致，也可能被检索出来。
-   **稀疏向量（Sparse Vector）：** 稀疏向量通常基于词频统计方法生成，例如 BM25。向量维度对应词表大小，通常达到数万维，其中大部分位置为 0。它依赖关键词匹配，在检索包含特定术语的文档时效果稳定。

在数据库中，这两类向量可以分别存放在不同字段中，例如 `dense` 与 `sparse`。查询时通过 **混合检索（Hybrid Search）** 同时利用两类信息：**稠密向量用于语义相关性匹配，稀疏向量用于关键词匹配**。这样可以同时兼顾语义召回和关键词精确匹配。

为了提高检索效率，向量数据库还会为向量字段建立索引，例如 **HNSW 或 IVF_FLAT**。这些索引基于 Approximate Nearest Neighbor（ANN）搜索方法，用近似计算代替逐条比对，从而减少检索时需要计算的向量数量。其中HNSW 和 IVF_FLAT 是两种主流的近似最近邻（ANN）索引算法，分别基于图结构和聚类分桶实现高效向量检索。在数据规模达到百万甚至更高时，这种算法仍能在较短时间内返回结果。

代码示例：

```
from langchain_milvus import Milvus, BM25BuiltInFunction
from langchain_openai import OpenAIEmbeddings

vectorstore = Milvus.from_documents(
    documents=docs,
    embedding=OpenAIEmbeddings(),
    builtin_function=BM25BuiltInFunction(),  # 用于生成稀疏向量, 可指定输出字段名 output_field_names="sparse"),
    vector_field=["dense", "sparse"],
    connection_args={
        "uri": URI,     # Milvus 服务器连接配置
    },
    consistency_level="Bounded",  # 数据一致性级别, Supported values are (`"Strong"`, `"Session"`, `"Bounded"`, `"Eventually"`). See https://milvus.io/docs/consistency.md#Consistency-Level for more details.
    drop_old=False,  # 是否删除已存在的同名 Collection
)
```

4.  ### 检索召回

用户发起查询时，被同样的嵌入模型转换为向量，系统在向量空间中计算查询向量与文档向量的相似度（通常使用余弦相似度或点积），返回Top-K个最相似的文本块。部分方案会加入关键词过滤、元数据过滤，进一步缩小召回范围，提升相关性。

代码示例：

```
from langchain_core.runnables import RunnablePassthrough
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model_name="gpt-3.5-turbo", temperature=0)

PROMPT_TEMPLATE = """
Human: You are an AI assistant, and provides answers to questions by using fact based and statistical information when possible.
Use the following pieces of information to provide a concise answer to the question enclosed in <question> tags.
If you don't know the answer, just say that you don't know, don't try to make up an answer.
<context>
{context}
</context>

<question>
{question}
</question>

The response should be specific and use statistics or numbers when possible.

Assistant:"""

prompt = PromptTemplate(
    template=PROMPT_TEMPLATE, input_variables=["context", "question"]
)
retriever = vectorstore.as_retriever()


def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)
```

（更多用法：https://milvus.io/docs/zh/integrate_with_langchain.md）

5.  ### 重排序（Reranking)

初步召回的Top-K结果仅基于向量相似度，可能存在语义相近但相关性不足、或长文本被截断导致关键信息丢失等问题。重排序阶段引入更精细的排序模型，对候选文档进行二次打分，综合考虑查询与文档的深层交互关系，重新排列相关性顺序，将更相关的结果推至前列，再截取Top-N送入上下文拼接阶段。

重排序模型通常采用**交叉编码器（Cross-Encoder）架构**，将查询与每个候选文本块拼接后一同输入模型，逐对计算相关性得分。相较于双塔结构（Bi-Encoder，查询和文档各走各的编码器，分别独立生成向量，最后用余弦相似度或点积来衡量两者的相关性）的嵌入模型，交叉编码器能捕捉查询与文档之间更细粒度的语义交互，但计算成本也更高，因此适合在召回阶段缩量之后使用，而非直接作用于全量文档库。常见的重排序模型包括 Cohere Rerank、BGE-Reranker 等。

代码示例：

```
from langchain.retrievers import ContextualCompressionRetriever
from langchain_community.document_transformers import EmbeddingsRedundantFilter
from langchain_cohere import CohereRerank

# 初始化重排序器（以Cohere为例，也可替换为HuggingFace的Cross-Encoder）
reranker = CohereRerank(model="rerank-multilingual-v2.0", top_n=5)

# 构建带压缩的检索器：先召回20个候选，再重排序取Top-5
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20})
)

# 在RAG链中使用重排序后的检索器
rag_chain = (
    {"context": compression_retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

response = rag_chain.invoke("查询问题")
```

6.  ### 上下文拼接与生成

将召回的文本块按相关性排序、拼接为上下文，与用户查询一同输入大模型；大模型基于检索到的事实信息生成答案，拒绝脱离知识库的臆测，同时保留自然语言的表达流畅度。生成环节可设置温度值（Temperature）为0.1-0.3，降低随机性，强化事实性。（Temperature 是大语言模型的一个核心超参数，用于控制模型生成文本时的随机性和创造性。数值越小，输出更确定；反之，输出更具有随机性。）

代码示例参考上述代码。

## GraphRAG

传统RAG依赖**文本块级别的语义相似度检索**，本质是碎片化的知识匹配。因此，传统RAG无法捕捉实体间的关联关系，且仅聚焦局部文本相似度，无法对知识库做全局归纳、关联分析，难以回答宏观性、探索性问题。

GraphRAG（Graph-based Retrieval-Augmented Generation）由微软研究院于2024年提出，是对传统RAG的架构性升级。它保留了RAG“检索+生成”的核心框架，同时引入**知识图谱**对信息进行结构化建模，将碎片化的文本转化为“节点+关系”的图网络，让检索从“文本匹配”升级为“知识推理”，以解决传统RAG的关联缺失与推理不足的问题。

### GraphRAG的核心定义

GraphRAG的核心是**用图结构表征知识**，即将文本中的人名、地名、机构、概念、产品等具象或抽象事物定义为**节点（Node）** ，将节点之间的关联（如“研发”“隶属”“合作”“因果”）定义为**边（Edge）** ，边可附加属性（如关系时间、权重、可信度）。整个知识库不再是零散的文本块，而是一张相互关联的知识网络，检索环节可沿着关系路径做多层推理，召回完整的知识子图而非孤立片段。

**与传统RAG相比，GraphRAG的架构新增了知识抽取、图谱构建、图检索三大核心模块，向量检索不再是唯一召回方式，而是与图检索形成互补。**

1.  ### 知识抽取

GraphRAG的离线阶段在传统RAG分块、嵌入的基础上，**新增知识结构化与图谱建模**环节，核心步骤如下：

-   **文本分块与预处理**：沿用传统RAG的语义分块逻辑，保证每个文本块包含完整的实体与关系信息，分块粒度更精细（通常100-500字符），便于精准抽取知识。

>     但也有另一种说法是，实体关系抽取依赖上下文，chunk的目的主要是控制 LLM 的 context 大小，而不是为了向量检索，所以 chunk_size 通常设得更大（1200～2400 tokens）。其中，chunk 只是抽取的中间载体，抽完之后真正被存储和检索的是实体节点、关系边和社区摘要，chunk 本身退出了检索链路。既然 chunk 不直接参与检索，就没有必要为了"检索精准"而切得很细，反而应该为了"抽取完整"而设得稍大。
>     所以具体情况具体分析吧。

-   **实体与关系抽取**：通过大模型或命名实体识别（NER）模型、关系抽取模型，从每个文本块中提取标准化的 **（实体，关系，实体）** 三元组。例如从“华为研发出鸿蒙操作系统”中，提取实体“华为”“鸿蒙操作系统”，关系“研发”。这一步需做实体消歧（如区分同名人物、机构）与实体归一化（如统一别称、全称），避免图谱冗余混乱。

  代码示例：

```
from langchain.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
import ast
import os

# 初始化 OpenAI 模型
llm = ChatOpenAI(
    model="gpt-4o",
    temperature=0,
    api_key=os.environ.get("OPENAI_API_KEY")  # 或直接传入字符串
)

# 定义抽取提示词模板
extract_prompt = PromptTemplate(
    input_variables=["text"],
    template="""
    从以下文本中提取(实体,关系,实体)三元组，实体包括人物、机构、产品、概念，关系精准简洁，仅输出三元组，无其他文字：
    文本：{text}
    输出格式：[("实体1","关系","实体2"),("实体1","关系","实体3")]
    """
)

# 构建抽取链，末尾加解析器提取纯文本
from langchain_core.output_parsers import StrOutputParser

extract_chain = extract_prompt | llm | StrOutputParser()

# 执行抽取
test_text = split_docs[0].page_content
triplets_str = extract_chain.invoke({"text": test_text})

# 将字符串解析为 Python 列表
try:
    triplet_list = ast.literal_eval(triplets_str.strip())
    print("抽取的知识三元组：", triplet_list)
except Exception as e:
    print("解析失败，原始输出：", triplets_str)
```

2.  ### 图谱构建

-   **知识图谱** **构建与存储**：将抽取的三元组整合为全局知识图谱，存入图数据库（如Neo4j、NebulaGraph）。同时，为实体、关系、文本块分别生成向量，构建“图结构+向量”的混合索引。

    代码示例：
    ```
        # pip install langchain-openai langchain-community neo4j lancedb graspologic sentence-transformers

        from neo4j import GraphDatabase
        from langchain_openai import ChatOpenAI, OpenAIEmbeddings
        from langchain_core.output_parsers import StrOutputParser
        from langchain.prompts import PromptTemplate
        import ast, os

        # ── 连接 Neo4j ──────────────────────────────────────────────
        NEO4J_URI  = "bolt://localhost:7687"
        NEO4J_USER = "neo4j"
        NEO4J_PASS = "your_password"
        driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASS))

        embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

        # ── 将单条三元组写入 Neo4j，同时存储实体向量 ────────────────
        def store_triplet(tx, head, relation, tail, head_vec, tail_vec):
            tx.run("""
                MERGE (h:Entity {name: $head})
                ON CREATE SET h.embedding = $head_vec
                MERGE (t:Entity {name: $tail})
                ON CREATE SET t.embedding = $tail_vec
                MERGE (h)-[r:RELATION {type: $relation}]->(t)
            """, head=head, relation=relation, tail=tail,
                 head_vec=head_vec, tail_vec=tail_vec)

        def ingest_triplets(triplet_list: list[tuple]):
            with driver.session() as session:
                for head, relation, tail in triplet_list:
                    head_vec = embeddings.embed_query(head)
                    tail_vec = embeddings.embed_query(tail)
                    session.execute_write(store_triplet, head, relation, tail, head_vec, tail_vec)

        def process_documents(split_docs):
            for doc in split_docs:
                raw = extract_chain.invoke({"text": doc.page_content})
                try:
                    triplets = ast.literal_eval(raw.strip())
                    ingest_triplets(triplets)
                    print(f"已存入 {len(triplets)} 条三元组")
                except Exception as e:
                    print(f"解析失败：{e}\n原始输出：{raw}")
        ```
>     进阶方案会通过Leiden社区检测算法，对图谱中的密集关联节点做聚类，形成层次化社区（community），每个社区（community）对应一个核心主题，便于全局检索与归纳。

-   **社区摘要生成**：为每个实体社区生成摘要，概括社区内的核心知识与关联逻辑，将摘要转化为向量存入索引，提升宏观问题的检索效率。

    代码示例：
    ```
        # ── 为每个社区生成摘要并回写 Neo4j ───────────────────────────
        summary_prompt = PromptTemplate(
            input_variables=["entities", "relations"],
            template="""
            以下是一个知识图谱社区中的实体与关系：
            实体：{entities}
            关系：{relations}
            
            请用3~5句话概括该社区的核心主题、关键实体及其关联逻辑，语言简洁客观。
            """
        )
        summary_chain = summary_prompt | llm | StrOutputParser()

        def write_community_summary(tx, cid, summary, embedding, entities):
            # 创建 Community 节点
            tx.run("""
                MERGE (c:Community {id: $cid})
                SET c.summary = $summary, c.embedding = $embedding
            """, cid=cid, summary=summary, embedding=embedding)
            # 将社区内实体与 Community 节点关联
            for entity in entities:
                tx.run("""
                    MATCH (e:Entity {name: $name})
                    MATCH (c:Community {id: $cid})
                    MERGE (e)-[:BELONGS_TO]->(c)
                """, name=entity, cid=cid)

        def build_community_summaries(split_docs):
            G = load_graph_from_neo4j()
            partition = detect_communities(G)
            communities = group_by_community(G, partition)

            for cid, data in communities.items():
                entities_str  = "、".join(data["entities"])
                relations_str = "\n".join(data["relations"]) or "无"

                # 生成摘要
                summary = summary_chain.invoke({
                    "entities": entities_str,
                    "relations": relations_str
                })

                # 摘要向量化
                summary_vec = embeddings.embed_query(summary)

                # 写回 Neo4j
                with driver.session() as session:
                    session.execute_write(
                        write_community_summary,
                        cid, summary, summary_vec, data["entities"]
                    )
                print(f"社区 {cid}（{len(data['entities'])} 个实体）摘要已写入")
        ```

3.  ### 图检索

GraphRAG的检索部分采用**查询解析→实体定位→子图检索→信息融合→生成输出**的推理流程，技术细节如下：

-   **查询意图解析**：对用户查询做实体链接与关系提取，识别查询中的核心实体与目标关系，将自然语言问题转化为图查询指令（如Cypher语句），明确检索的节点与路径范围。
-   **混合检索召回**：先通过向量检索定位相关实体与文本块，再以核心实体为起点，在知识图谱中做图遍历（广度优先、深度优先），沿关系边扩展1-3跳，召回与查询相关的完整知识子图；同时匹配社区摘要，补充全局视角的知识。这一步既保留语义相关性，又覆盖完整的关联逻辑。
-   **子图修剪与排序**：剔除子图中无关的节点与边，过滤低权重、低可信度的关系，按知识与查询的相关性排序，保留核心推理路径，减少大模型的上下文负担。
-   **知识融合与生成**：将修剪后的知识子图、关联文本块、社区摘要拼接为上下文，输入大模型；大模型基于结构化的知识网络生成答案，可清晰呈现推理路径，针对多跳推理、全局分析、关联归纳类问题，输出更完整、更严谨的结果。

例如，在项目[Yuxi-Know](https://github.com/xerrors/Yuxi-Know#)中，针对用户查询进行GraphRAG的过程如下：

-   **查询解析** ：首先对用户输入的关键词进行预处理和分词。在 query_node 方法中，通过**空格分割**将输入字符串分解为多个token，确保系统能够处理复合查询，如"北京 中国"这样的多关键词输入，同时保留了原始关键词作为兜底方案。
-   **实体定位** ：采用混合检索策略精准定位相关实体。一方面，通过 _query_with_vector_sim 方法，将查询文本转换为向量表示，利用Neo4j的向量索引 entityEmbeddings 进行**语义相似度搜索**，返回相似度超过阈值（默认0.9）的实体及其得分；另一方面，通过 _query_with_fuzzy_match 方法执行**不区分大小写的模糊匹配**，为包含关键词的实体赋予较低权重（0.3），确保不遗漏相关实体。系统对每个token的检索结果进行聚合，按得分排序后选取top N个实体作为候选实体集。
-   **子图检索** ：针对定位到的候选实体，系统执行多跳图谱遍历查询。通过复杂的Cypher查询语句，同时查询实体的1跳和2跳邻居节点，包括出边和入边关系。查询结果包含头节点、关系和尾节点的完整信息，支持无向关系的检索。系统为每个候选实体执行独立的子图查询，收集所有相关的节点和边，形成完整的知识子图。
-   **信息融合** ：系统对多个实体的查询结果进行智能融合处理。首先执行节点去重，基于节点ID消除重复节点；然后对边进行去重，通过(source_id, target_id, type)三元组作为唯一标识避免重复边。另外，系统还移除了embedding字段以减少数据传输量，并将节点的properties属性扁平化处理，优化数据结构。融合后的结果保持了图谱的连通性和完整性。
-   **生成输出** ：最终，系统将处理后的数据转换为标准化的图谱格式返回。通过_format_results方法，将原始节点和边数据转换为统一的标准格式，包含id、name、type、properties等核心字段。输出数据结构为{"nodes": [...], "edges": [...]}，可直接用于前端图谱可视化展示或供智能体工具调用。整个过程确保了从用户输入到结构化输出的端到端检索能力。

## 总结

| 技术方案     | 核心特点                | 适用场景                           | 落地成本                         |
| -------- | ------------------- | ------------------------------ | ---------------------------- |
| 传统RAG    | 文本级语义匹配，架构简单，响应速度快  | 单跳事实问答、文档摘要、常规客服、简单信息查询        | 低，无需图数据库与知识抽取模型，快速落地         |
| GraphRAG | 图结构知识推理，关联能力强，可解释性高 | 多跳推理、全局分析、知识关联挖掘、垂直领域专业问答、决策支持 | 中高，需图数据库、知识抽取、社区聚类模块，部署复杂度更高 |

传统RAG解决了大模型“用外部知识”的基础问题，是轻量化检索增强的最优解；GraphRAG则解决了大模型“理解知识关联”的进阶问题，是复杂场景下的升级方向。二者并非替代关系，而是递进互补：简单场景可直接用传统RAG保障效率，复杂关联场景则通过GraphRAG提升精度与推理能力。
