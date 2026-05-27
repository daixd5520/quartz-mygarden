# 企业级 RAG 系统设计完全指南

构建企业内部文档 RAG 时，很多团队开始时只想着"向量化+检索"，结果踩坑无数。本文基于实战经验，给出从选型到部署的完整方案。

**相关理论文档**：[RAG 技术演进](../bd/Tech/Tech--RAG.md)

## 0. RAG 的完整流水线（先把骨架搞清楚）

任何企业 RAG 系统的核心流程都遵循这个模式：

```
1. Ingestion (接入)
   ↓ [从 SharePoint/Confluence/网盘/数据库同步]
2. Extraction (抽取)
   ↓ [PDF/DOCX/HTML → 保真的文本 + 结构]
3. Cleaning & Normalization (清洗)
   ↓ [去页眉页脚、编码统一、语言检测]
4. Chunking (切分)
   ↓ [按策略切段，加 overlap，绑定元数据]
5. Enrichment (增强，可选但强烈推荐)
   ↓ [实体、关键词、部门、时间、权限标签]
6. Indexing (索引)
   ├─ 向量索引 (embedding → VectorDB)
   └─ 关键词索引 (BM25/倒排)
7. Query Understanding (查询理解)
   ↓ [改写/扩展/实体抽取/过滤器生成]
8. Retrieval (检索)
   ↓ [向量召回 + 关键词召回 → 混合模型]
9. Rerank & Compression (重排 & 压缩)
   ↓ [交叉编码器、MMR 多样性、上下文压缩]
10. Generation (生成)
    ↓ [答案 + 引用]
11. Eval & Monitoring (评估)
    ↓ [召回率、幻觉率、权限泄露、延迟、成本]
```

**关键说明**：NLTK 和 spaCy 主要出现在第 4、5、7、9 这几步。

## 1. 选型：什么时候用 NLTK，什么时候用 spaCy

这不是"哪个更强"的问题，而是"你需要 NLP 到什么深度"。

### 选 NLTK 的条件

**适合场景**
- 快速搭建能跑的 RAG（不追求极致）
- NLP 只做轻量级处理：句子切分、关键词提取、停用词过滤
- 文档主要是英文
- 目标是**可解释的关键词检索增强**（FAQ、制度条款很适合）

**优势**
- 轻量级，依赖少
- 学习成本低
- 对英文文本处理足够好

**劣势**
- 对中文、日文等非英文支持差
- NER 能力弱
- 不适合复杂的语义任务

### 选 spaCy 的条件

**适合场景**
- 想做**结构化增强**（企业常见需求）
  - NER：系统名、产品名、部门、人名、地点、日期
  - noun chunks（名词短语）
  - 依存句法（可选）
  - 规则抽取
- 需要生产级 pipeline：速度、组件化、可扩展
- 想做**按实体/时间/部门过滤的检索体验**

**优势**
- 完整的 NLP pipeline
- 性能优秀（C 核心）
- 可自定义组件（EntityRuler、Matcher）
- 预训练模型丰富

**劣势**
- 学习曲线陡
- 初始配置复杂

### 推荐决策树

```
需要实体标注？
├─ Yes → 用 spaCy
│   ├─ 中文 → zh_core_web_sm
│   ├─ 英文 → en_core_web_sm
│   └─ 多语言 → 自定义 EntityRuler
└─ No / 时间有限 → 用 NLTK
    └─ 英文为主 → 用 NLTK
```

## 2. Chunk 设计：企业 RAG 成败的关键

不管选择哪个库，最终的 chunk 数据结构都类似：

```python
@dataclass
class Chunk:
    chunk_id: str
    doc_id: str
    text: str
    metadata: {
        "title": str,
        "section_path": str,           # "部门/系统/文档类型"
        "page_range": tuple,
        "url": str,
        "source_system": str,          # SharePoint/Confluence/...
        "created_at": datetime,
        "updated_at": datetime,
        "acl_groups": List[str],       # 访问权限
        "language": str,
        "extracted_entities": Dict,    # 可选：NER 结果
        "keywords": List[str]          # 可选：关键词提取
    }
    embedding_vector: np.ndarray
    bm25_terms: List[str]              # 可选：关键词索引
```

### Chunk 设计的黄金法则

**大小平衡**
- 太大：召回不准、context window 装不下、答案难拼接
- 太小：丧失上下文、答案片段化、需要多次检索

**经验阈值**（以 token 计，非字符）
```
通用文档 (手册、报告):
  chunk_size: 300-800 tokens
  overlap: 50-150 tokens

规章制度 / 合同:
  chunk_size: 600-1000 tokens (倾向更大，避免断章取义)
  overlap: 100-200 tokens

FAQ / 短问答:
  chunk_size: 200-400 tokens (更小，定位更精准)
  overlap: 20-50 tokens
```

**关键细节**
- **用目标 LLM 的 tokenizer 做计数**，不要用字符数或估算
- **必须包含结构元数据**：标题、章节路径、页码、链接
- **必须包含权限信息**：在检索时强过滤，避免权限泄露
- **不要丢弃表格 & 代码块**：这些通常包含最重要的信息

### 常见 Chunk 问题及解决方案

| 问题 | 症状 | 解决方案 |
|------|------|---------|
| Chunk 过大 | 召回准确率低，答案难以定位 | 使用 semantic chunking（按语义边界切） |
| Chunk 过小 | 丧失上下文，答案拼不起来 | 增加 overlap，或在检索后进行**上下文扩展** |
| 表格丢失 | 表格型数据变成纯文本，结构信息丧失 | 保留原始表格标记或转换为 markdown 格式 |
| 权限混乱 | RAG 泄露了用户不该看到的信息 | 在 Chunk 元数据中标注 ACL，检索时强制过滤 |

## 3. 方案 A：NLTK 技术方案（传统 IR 强、轻量可解释）

### 核心处理流程

```python
from nltk.tokenize import sent_tokenize, word_tokenize
from nltk.corpus import stopwords
import re

def preprocess_text(text):
    # 1. 清洗
    text = re.sub(r'\n+', ' ', text)  # 断行合并
    text = re.sub(r'\s+', ' ', text)  # 多空格合并
    
    # 2. 句子切分
    sentences = sent_tokenize(text, language='english')
    
    # 3. Tokenize
    tokens = word_tokenize(text)
    
    # 4. 去停用词 + 小写
    stop_words = set(stopwords.words('english'))
    keywords = [w.lower() for w in tokens if w.isalpha() and w.lower() not in stop_words]
    
    return sentences, keywords
```

### Chunking 实现

```python
def chunk_by_sentences(sentences, chunk_size=500):
    """按句子数量切 chunk"""
    chunks = []
    current_chunk = []
    current_size = 0
    
    for sent in sentences:
        tokens = len(word_tokenize(sent))
        if current_size + tokens > chunk_size and current_chunk:
            chunks.append(' '.join(current_chunk))
            current_chunk = [sent]
            current_size = tokens
        else:
            current_chunk.append(sent)
            current_size += tokens
    
    if current_chunk:
        chunks.append(' '.join(current_chunk))
    
    return chunks
```

### 关键词索引（BM25）

```python
from rank_bm25 import BM25Okapi

def build_bm25_index(documents):
    """构建 BM25 索引"""
    # documents: 每个文档都是 token 列表
    bm25 = BM25Okapi(documents)
    return bm25

def retrieve_by_keywords(query, bm25, documents, top_k=5):
    """关键词检索"""
    query_tokens = word_tokenize(query.lower())
    scores = bm25.get_scores(query_tokens)
    top_indices = np.argsort(scores)[-top_k:][::-1]
    return [documents[i] for i in top_indices]
```

### 何时选 NLTK 方案
- 企业文档以英文为主
- 重点是快速部署而非完美精度
- 关键词检索足以覆盖 80% 的用例

---

## 4. 方案 B：spaCy 技术方案（结构化增强、工程化）

### Pipeline 配置

```python
import spacy
from spacy.lang.en.stop_words import STOP_WORDS

nlp = spacy.load("en_core_web_sm")

# 添加自定义组件
ruler = nlp.add_pipe("entity_ruler", before="ner")
patterns = [
    {"label": "SYSTEM", "pattern": "SAP"},
    {"label": "SYSTEM", "pattern": "Salesforce"},
    {"label": "DEPT", "pattern": [{"LOWER": "hr"}, {"LOWER": "department"}]},
]
ruler.add_patterns(patterns)
```

### 结构化抽取

```python
def extract_entities_and_phrases(text):
    """抽取实体和名词短语"""
    doc = nlp(text)
    
    result = {
        "entities": [(ent.text, ent.label_) for ent in doc.ents],
        "noun_chunks": [chunk.text for chunk in doc.noun_chunks],
        "sentences": [sent.text for sent in doc.sents],
    }
    return result

# 示例
text = "The HR department uses Salesforce for CRM management."
info = extract_entities_and_phrases(text)
# {
#   "entities": [("HR", "DEPT"), ("Salesforce", "SYSTEM")],
#   "noun_chunks": ["The HR department", "Salesforce", "CRM management"],
#   "sentences": ["The HR department uses Salesforce for CRM management."]
# }
```

### 过滤器生成（Query Understanding）

```python
def generate_filters_from_query(query):
    """从 Query 自动生成过滤条件"""
    doc = nlp(query)
    
    filters = {}
    
    # 时间过滤
    for token in doc:
        if token.pos_ == "NOUN" and token.text.lower() in ["2026", "2025"]:
            filters["year"] = token.text
    
    # 部门过滤
    for ent in doc.ents:
        if ent.label_ == "DEPT":
            filters["department"] = ent.text
    
    # 系统过滤
    for ent in doc.ents:
        if ent.label_ == "SYSTEM":
            filters["system"] = ent.text
    
    return filters

# 使用
query = "HR department's Salesforce policy for 2026"
filters = generate_filters_from_query(query)
# {'department': 'HR', 'system': 'Salesforce', 'year': '2026'}
```

## 5. 混合检索：向量 + 关键词

```python
def hybrid_retrieve(query, vector_db, bm25_index, alpha=0.5):
    """向量 + 关键词混合检索"""
    
    # 1. 向量检索（语义相关性）
    vector_scores = vector_db.search(query, top_k=10)
    
    # 2. 关键词检索（精确匹配）
    keyword_scores = bm25_index.search(query, top_k=10)
    
    # 3. 融合（线性组合）
    merged = {}
    for doc_id, score in vector_scores:
        merged[doc_id] = alpha * score
    
    for doc_id, score in keyword_scores:
        if doc_id in merged:
            merged[doc_id] += (1 - alpha) * score
        else:
            merged[doc_id] = (1 - alpha) * score
    
    # 4. 排序 & 返回
    top_docs = sorted(merged.items(), key=lambda x: x[1], reverse=True)[:5]
    return top_docs
```

## 6. 常见坑与解决方案

| 坑 | 原因 | 解决方案 |
|---|-----|--------|
| 权限泄露 | 检索时未过滤 ACL | 在 retrieval 步骤强制 AND 权限过滤 |
| 答案幻觉 | RAG 检索不到相关文档 | 提升 chunk 质量、增加 overlap、用 rerank |
| 延迟过高 | 多次 forward pass | 使用 batch 处理、缓存 embedding、异步检索 |
| 多语言混乱 | 混用不同语言的 embedding | 使用多语言 embedding 模型或按语言分库 |
| Token 开销大 | 上下文塞满了冗余引用 | 使用 extraction 或 compression 算法压缩 |

## 7. 部署 Checklist

- [ ] Chunk 质量验证：随机抽样 100 条，人工检查
- [ ] 权限测试：确保用户 A 看不到用户 B 的文档
- [ ] 延迟测试：P99 < 2s
- [ ] 召回率测试：在已知相关文档上能召回到吗？
- [ ] 幻觉率：生成答案中有"文档不支持"的比例 < 5%
- [ ] 监控面板：设置实时告警（延迟、幻觉、权限泄露）

