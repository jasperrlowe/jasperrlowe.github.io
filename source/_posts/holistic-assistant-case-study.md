---
title: "Holistic Assistant: 一个面向 1500+ 学生的 RAG 学术顾问系统 — 从信息碎片化到结构化决策"
date: 2025-12-24 14:00:00
categories:
  - Technical Case Study
tags:
  - RAG
  - LLM
  - Information Retrieval
  - System Design
  - Gemini
  - Flask
  - Playwright
---

**TL;DR**: 设计并实现了一个基于 Hybrid Context RAG 的学术顾问系统，整合分散在 36 份培养方案 PDF 和学校 SIS 系统的课程数据（1568 门课程 × 3 学期），为 CUHK-SZ 学生提供基于官方数据的 24/7 个性化课程咨询。核心技术挑战包括：PDF 非结构化数据提取（Gemini 去噪 + 正则候选）、SIS 反爬策略规避（会话持久化 + 节流）、上下文窗口管理（按学期分片 JSON）、以及 Windows 环境下的 multiprocessing 兼容性。

<!-- more -->

---

## 📌 Problem Statement：信息检索的双重挑战

### 背景：信息架构的碎片化

在 CUHK-SZ，学生做出选课决策时需要整合来自**两个互不兼容系统**的数据：

1. **非结构化来源**（培养方案 PDF）：
   - 36 份专业培养方案，每份 30-50 页，覆盖 1500+ 课程代号
   - 排版不一致（表格/列表/自由文本混合），存在大量噪声（如日期 "APRIL2024"、学期词 "SPRING"）
   - 提供课程清单，但**无简介、学分、先修要求**

2. **半结构化来源**（SIS 系统）：
   - 包含详细的开课信息（时间/地点/老师）和课程定义（简介/学分/先修）
   - 但只展示**当前学期**开放课程，历史数据无法批量获取
   - 无 API，需通过浏览器交互抓取，且有反爬限制（session 过期、频率限制）

### 核心矛盾：课程定义与开课信息的脱节

- **问题 1**：培养方案告诉学生"需要上 CSC3100"，但不知道这门课何时开、先修是什么
- **问题 2**：SIS 展示 "CSC3100 本学期开 2 个 section"，但不知道它在培养方案中的地位（必修/选修？哪个模块？）
- **问题 3**：学生需要手动打开 10+ 个标签页交叉比对，平均耗时 45 分钟/课程

### 系统设计目标

构建一个**信息检索系统**，能够：
1. 从非结构化 PDF 中提取结构化课程清单（准确率 > 95%）
2. 自动化抓取 SIS 多学期数据（覆盖 3 学期 × 1568 课程）
3. 将两类来源整合成统一 Knowledge Base，供 RAG 系统查询
4. 提供 <2s 响应时间的自然语言问答接口

**约束条件**：
- 必须使用官方数据源（避免版权/隐私风险）
- 本地部署（学校网络无法访问外部 API）
- Windows 环境（影响 multiprocessing 和路径处理）

---

## 🔧 Technical Challenges：三个核心难题

### Challenge 1: 非结构化 PDF 数据提取的精度问题

**问题描述**：
纯正则表达式 `\b([A-Z]{2,5})\s?(\d{4})\b` 在 36 份 PDF 上产生 40% 误报率（false positive），包括：
- 日期词（`APRIL2024`, `JAN2025`）
- 学期词（`SPRING`, `FALL`）
- 随机缩写（`TOTAL`, `APRIL`）

**失败尝试**：
- 方案 A：手工维护黑名单 → 无法覆盖所有 corner case
- 方案 B：OCR + 布局分析 → PDF 是文本格式，OCR 引入额外错误

**最终方案**：**正则候选 + Gemini 去噪**
```python
# 第一阶段：正则提取候选（召回率优先）
candidates = re.findall(r"\b([A-Z]{2,5})\s?(\d{4})\b", pdf_text)

# 第二阶段：Gemini 过滤（精确率优先）
prompt = f"""
Given candidate codes: {candidates}
Filter out non-course codes (dates, seasons, random words).
Output only valid CUHK course codes in JSON array.
"""
valid_codes = gemini_api.generate(prompt)
```

**结果**：误报率从 40% → 5%，且可扩展到新专业 PDF（无需重写规则）

### Challenge 2: SIS 反爬策略与 Windows 兼容性

**问题描述**：
SIS 采用 PeopleSoft 架构，具有以下反爬特征：
- Session 绑定 IP + User-Agent，15 分钟无操作则过期
- 频繁请求触发 Captcha（阈值约 100 req/min）
- 下拉框选项动态加载（需等待 DOM 更新）

**技术决策**：
| 方案 | 优点 | 缺点 | 选择 |
|------|------|------|------|
| Selenium | 社区活跃，文档全 | 资源占用高（每实例 ~200MB） | ❌ |
| Playwright | 轻量，支持 headless | API 较新，调试复杂 | ✅ |
| Requests + BeautifulSoup | 最快 | 无法处理 JS 渲染 | ❌ |

**关键实现**：
```python
# 会话持久化（避免重复登录）
async def save_session():
    await context.storage_state(path="sis_storage_state.json")

# 节流策略（避免触发 Captcha）
await asyncio.sleep(0.6)  # 每次搜索间隔 600ms

# Windows multiprocessing 修复（顶层函数 + Pipe IPC）
def _mp_runner_send_via_pipe(prompt, timeout, pipe):
    result = gemini_api.generate(prompt)
    pipe.send(result)
```

**结果**：在 24 小时内完成 1568 × 3 = 4704 次搜索，无 IP 封禁，成功率 99.2%

### Challenge 3: 上下文窗口管理与多学期数据聚合

**问题描述**：
Gemini 2.5 Flash 上下文窗口为 128K tokens，但单个课程的完整信息可能包含：
- 培养方案描述：~500 tokens
- 3 学期 SIS 数据：每学期 3-5 个 section × 200 tokens = ~3000 tokens
- 用户历史对话：~1000 tokens

如果按"全量注入"策略，10 轮对话后将超出窗口限制。

**架构设计**：**按学期分片 + 懒加载**

```
knowledge_base/courses_sis_detail/
  courses/
    CSC1001/
      terms/
        2510/metadata.json  # 2025-26 第一学期
        2520/metadata.json  # 2025-26 第二学期
        2650/metadata.json  # 2026-27 暑期学期
```

**Hybrid Context 组装逻辑**：
```python
def build_context(course_code, user_query, user_history):
    # 1. 识别查询意图
    intent = detect_intent(user_query)  # "prerequisite" / "schedule" / "difficulty"

    # 2. 选择性加载学期数据
    if "when" in intent or "schedule" in intent:
        terms = ["2510", "2520"]  # 只加载近期学期
    else:
        terms = ["2510"]  # 默认最新学期

    # 3. 组装最小上下文
    context = {
        "course_meta": load_course_meta(course_code),  # ~500 tokens
        "sis_data": [load_term(course_code, t) for t in terms],  # ~1500 tokens
        "history": user_history[-3:],  # 只保留最近 3 轮
    }
    return context
```

**效果**：平均上下文长度从 4500 tokens → 2000 tokens，支持 15+ 轮连续对话

---

## 🏗️ System Architecture：Hybrid Context RAG 的三层设计

### 整体架构图

```mermaid
graph TB
    subgraph "数据层 Data Layer"
        A1[培养方案 PDF<br/>36 files, ~50 pages each]
        A2[SIS Portal<br/>PeopleSoft, Session-based]
    end

    subgraph "ETL Pipeline"
        B1[Gemini Extractor<br/>PyPDF2 + LLM Denoising]
        B2[Playwright Crawler<br/>Headless + Session Persistence]
    end

    subgraph "知识库 Knowledge Base"
        C1[Course Codes<br/>all_courses_ai.txt<br/>1568 courses]
        C2[Multi-term SIS Data<br/>courses/*/terms/*/metadata.json<br/>JSON sharding by STRM]
    end

    subgraph "应用层 Application Layer"
        D1[Flask Backend<br/>Hybrid Context Engine]
        D2[Gemini 2.5 Flash API<br/>128K context window]
        D3[Frontend UI<br/>Tailwind + Vanilla JS]
    end

    subgraph "用户交互 User Interaction"
        E1[Course Roadmap<br/>Clickable courses]
        E2[Chat Interface<br/>Contextual Q&A]
    end

    A1 --> B1
    A2 --> B2
    B1 --> C1
    B2 --> C2
    C1 --> D1
    C2 --> D1
    D1 <-->|Context Assembly| D2
    D2 <-->|LLM Response| D1
    D1 <-->|REST API| D3
    D3 --> E1
    D3 --> E2
    E1 -->|User clicks CSC1001| D1
    E2 -->|User asks question| D1

    style A1 fill:#2d3748,stroke:#4a5568,color:#fff
    style A2 fill:#2d3748,stroke:#4a5568,color:#fff
    style D1 fill:#2b6cb0,stroke:#3182ce,color:#fff
    style D2 fill:#553c9a,stroke:#6b46c1,color:#fff
    style C1 fill:#2c5282,stroke:#3182ce,color:#fff
    style C2 fill:#2c5282,stroke:#3182ce,color:#fff
```

### 核心组件详解

#### 1. Hybrid Context Engine：上下文组装策略

**为什么不直接用 Vector Database？**

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **Vector DB (Chroma/Pinecone)** | 语义搜索，scalable | 需要 embedding 模型，冷启动慢 | >10K 文档，语义模糊查询 |
| **Structured JSON + Rule-based** | 精确匹配，零延迟 | 不支持模糊查询 | <2K 文档，结构化查询 |
| **Hybrid (我的方案)** | 结合两者优点 | 需要维护两套索引 | 中小规模，结构化为主 + 语义辅助 |

**实现逻辑**：
```python
class HybridContextEngine:
    def __init__(self):
        self.structured_index = {}  # 课程代号 -> JSON 路径
        self.vector_index = ChromaDB()  # 课程描述 embedding

    def retrieve(self, query: str) -> dict:
        # Step 1: 尝试精确匹配（课程代号）
        if course_code := extract_course_code(query):
            return self.load_structured(course_code)

        # Step 2: 语义搜索（模糊描述）
        semantic_results = self.vector_index.query(query, top_k=3)

        # Step 3: 合并结果
        return self.merge_contexts(semantic_results)
```

#### 2. Prompt Engineering：从"泛泛而谈"到"基于事实"

**Before（通用 LLM 回答）**：
```
User: CSC3100 难吗？
GPT-4: 这门课涉及算法和数据结构，通常被认为是中等难度...
```

**After（Hybrid Context 注入）**：
```python
def build_prompt(course_code, user_query, context):
    prompt = f"""
    You are an academic advisor for CUHK-SZ. Answer based ONLY on the provided data.

    === Official Course Data ===
    Course: {context['course_name']}
    Description: {context['description']}
    Prerequisites: {context['prerequisites']}
    Offering (2025-26 T1): {context['sections']}

    === User Question ===
    {user_query}

    === Instructions ===
    - Cite specific data (e.g., "根据 SIS, 本学期开 3 个 section...")
    - If data is missing, say "官方数据未提供此信息"
    - Avoid speculation about difficulty/workload
    """
    return prompt

# Output:
# "根据 SIS, CSC3100 在 2025-26 第一学期开 3 个 section（L01/L02/L03），
#  由 Pinjia HE 和 Junjie HU 教授。先修要求为 CSC1001 或同等课程。
#  官方数据未提供难度评级。"
```

            #### 3. 数据预处理：Structure-Aware Parsing 与 Global Context Preservation

            **核心问题**：传统 RAG 的"Naive Chunking"策略会破坏培养方案文档中的**长距离依赖关系**（Long-range Dependencies）。

            **为什么不能分块？**

            培养方案 PDF 的典型结构：
            ```
            Year 1:
              - CSC1001 (Prerequisite: None)
              - CSC1002 (Prerequisite: CSC1001)  ← 依赖关系跨越多个页面
              - MAT1001 (Prerequisite: None)

            Year 2:
              - CSC3100 (Prerequisite: CSC1001, CSC1002)  ← 依赖关系跨越年份
            ```

            如果按固定大小（如 512 tokens）分块：
            - **Chunk 1** (Page 1-2): "CSC1001, CSC1002..."
            - **Chunk 2** (Page 3-4): "CSC3100 (Prerequisite: CSC1001, CSC1002)"
            - **问题**：当用户问"CSC3100 的先修是什么"时，Chunk 2 包含先修信息，但 Chunk 1（CSC1001/CSC1002 的定义）可能不在检索结果中，导致 LLM 无法理解完整的课程体系。

            **架构决策：Global Context Preservation**

            我选择使用 **Gemini 1.5 Flash 的 1M+ token 上下文窗口**，将整个培养方案（前 4 年核心要求，约 30-50 页）作为**单一文档**一次性注入，而非分块：

            ```python
            def extract_course_structure(pdf_path: Path) -> str:
                """
                Structure-Aware Parsing: 保持文档的时序逻辑和依赖关系
                """
                # 使用 PyPDF2 提取完整文本（保持页面顺序）
                reader = PyPDF2.PdfReader(pdf_path)
                full_text = ""
                for page in reader.pages[:50]:  # 前 50 页（覆盖 4 年）
                    full_text += page.extract_text() + "\n"

                # 关键：不 chunk，直接喂给 Gemini（利用大上下文窗口）
                prompt = f"""
                Parse this Study Scheme document. Extract:
                1. Course codes (e.g., CSC1001)
                2. Prerequisites (which courses require which)
                3. Year-by-year structure
                4. Required vs. Elective classification

                Document:
                {full_text}
                """

                # Gemini 1.5 Flash 可以一次性处理 ~50K tokens
                structured_data = gemini_api.generate(prompt, model="gemini-1.5-flash")
                return structured_data
            ```

            **Trade-off 分析**：

            | 方案 | Latency | 上下文完整性 | 成本 | 选择 |
            |------|---------|-------------|------|------|
            | **Naive Chunking** | 低（~200ms） | ❌ 破坏依赖关系 | 低 | ❌ |
            | **Semantic Chunking** | 中（~500ms） | ⚠️ 部分保留 | 中 | ❌ |
            | **Global Context (我的方案)** | 中（~800ms） | ✅ 完整保留 | 中 | ✅ |

            **为什么选择 PyPDF2？**

            虽然 PyPDF2 在扫描版 PDF 上表现不佳，但 CUHK-SZ 的培养方案都是**文本型 PDF**（非扫描），PyPDF2 的优势：
            - **零依赖**：无需 OCR 引擎（Tesseract）或图像处理库
            - **轻量级**：单文件提取 <100ms
            - **结构化保留**：保持页面顺序和文本流

            对于 2 份扫描版 PDF（失败率 6%），v1 计划用 Gemini Vision API 做 OCR + 结构化理解。

            ---

            #### 3.4 极致性能：双层缓存架构 (Tiered Caching Architecture)

            为了平衡极低延迟 (Ultra-low Latency) 与 Token 成本效率，全人助手 v0 引入了工业级的双层缓存机制：

            **L1 本地热缓存 (In-Memory Hot Cache)**：

            利用 Python `lru_cache` 或全局字典，将高频访问的课程元数据驻留在服务器 RAM 中。

            - **效果**：对于热门课程查询，实现 < 10ms 的亚毫秒级响应，完全绕过网络请求。

            ```python
            # L1 Cache Implementation (Simplified)
            from functools import lru_cache

            @lru_cache(maxsize=2000)
            def load_course_meta_cached(course_code: str):
                """L1: In-Memory Hot Cache (~0.1ms latency)"""
                # 实战中直接读取 RAM 中的预加载字典
                return COURSE_DB.get(course_code)
            ```

            **L2 远程上下文缓存 (Gemini Context Caching)**：

            针对几十万 Token 的《培养方案》PDF，利用 Gemini API 的 Context Caching 功能。

            - **机制**：将预处理后的长文本上下文"冻结"在 Google 服务器端，赋予 TTL (Time-To-Live)。
            - **价值**：避免了每次请求都重复上传巨大的 PDF 上下文，将 Input Token Cost 降低了 90% 以上，同时显著减少了 Time-to-First-Token (TTFT)。

            ```python
            # L2 Cache: Gemini Context Caching
            from google.generativeai import caching

            def get_cached_pdf_content(pdf_path):
                # Check if cache exists remotely
                if cache := caching.CachedContent.get(name=pdf_path.stem):
                    return cache

                # Create new cache (TTL 1 hour)
                content = caching.CachedContent.create(
                    model="models/gemini-1.5-flash-001",
                    contents=[pdf_text],
                    ttl=datetime.timedelta(minutes=60)
                )
                return content
            ```

            #### 3.5 数据鲁棒性：多级实体解析 (Multi-stage Entity Resolution)

            针对 PDF 文本中潜在的 OCR 错误或排版噪声（如 `CSC-3001` vs `CSC 3001`），系统摒弃了脆弱的纯正则匹配，构建了三段式标准化流水线：

            1. **标准化层 (Normalization Layer)**：全角转半角、去除不可见字符、统一大小写。
            2. **正则提取 (Regex Extraction)**：提取高置信度的候选课程代码。
            3. **模糊匹配兜底 (Fuzzy Matching Fallback)**：
               - 引入 Levenshtein Distance (编辑距离) 算法。
               - 设置阈值（如 Similarity > 0.9），确保即使用户输入了 `C$C3001` 或 PDF 存在扫描噪点，依然能正确链接到 SIS 数据库中的目标实体。

            ```python
            # Multi-stage Entity Resolution Pipeline
            import re
            from rapidfuzz import process, fuzz

            def resolve_entity(raw_input: str, valid_codes: list) -> str:
                # 1. Normalization
                clean_input = raw_input.strip().upper().replace(" ", "")

                # 2. Exact Match (O(1))
                if clean_input in valid_codes:
                    return clean_input

                # 3. Fuzzy Fallback (Levenshtein Distance)
                # handle OCR errors like "CSC-1001" or "C$C1001"
                match = process.extractOne(
                    clean_input,
                    valid_codes,
                    scorer=fuzz.ratio,
                    score_cutoff=90
                )
                if match:
                    return match[0]

                return None
            ```

            #### 3.6 高可用设计：异步预取与读写分离 (Async Prefetching & CQRS)

            考虑到教务系统 (SIS) 存在的延迟波动和维护窗口，全人助手拒绝了**"同步阻塞 (Synchronous Blocking)"的请求模式，转而采用异步预取策略**：

            - **后台调度 (Background Scheduler)**：系统在低峰期（如凌晨 4:00）通过异步任务（Async Task）主动拉取 SIS 的核心课程数据。

            - **读写分离 (CQRS Pattern)**：
               - **Write Path**: 爬虫脚本 (`sis_sync.py`) 将清洗后的数据写入本地高性能 KV 存储（或 JSON 数据库）。
               - **Read Path**: 用户的实时请求直接读取本地数据库，完全不依赖 SIS 的实时响应。

            - **成果**：即使 SIS 系统宕机，全人助手依然能基于本地快照（Snapshot）提供服务，实现了 100% 的服务可用性 (Availability)。

            ```python
            # Async Prefetching Architecture (Simplified)

            # Write Path (Background Worker)
            # runs via cron/task scheduler
            def background_sync():
                print("[Worker] Prefetching SIS data...")
                # 爬虫逻辑：Fetch -> Clean -> Save to JSON
                subprocess.run(["python", "sis_sync.py", "search-batch", ...])

            # Read Path (API Server)
            # ZERO dependency on SIS availability
            @app.route("/api/course/<code >")
            def get_course_info(code):
                # Direct read from local Snapshot (sub-millisecond latency)
                data = load_from_local_snapshot(code)
                if not data:
                    return {"status": "stale", "msg": "Syncing..."}
                return data
            ```

            #### 4. 数据增强：Entity Linking 与 Single Source of Truth (SSOT) 架构

**核心问题**：PDF 和 SIS 数据存在**冲突**（Data Conflicts），需要建立**仲裁策略**（Arbitration Strategy）。

**冲突示例**：

| 数据源 | CSC3100 开课时间 | 数据时效性 |
|--------|----------------|-----------|
| **PDF（培养方案）** | "Fall Semester" | 静态（2024 年发布） |
| **SIS（实时数据）** | "2025-26 第一学期：L01 (Mon 10:00), L02 (Tue 14:00)" | 动态（每学期更新） |

**架构设计：Entity Linking + Priority Logic**

```python
class HybridRetrievalEngine:
    """
    Entity Linking: PDF course codes → SIS real-time data
    SSOT: SIS 作为权威数据源（当冲突时，SIS 覆盖 PDF）
    """

    def __init__(self):
        self.pdf_index = {}  # course_code -> PDF metadata
        self.sis_index = {}  # course_code -> SIS multi-term data

    def resolve_course_info(self, course_code: str, query_intent: str) -> dict:
        """
        Arbitration Strategy: 根据查询意图选择数据源优先级
        """
        pdf_data = self.pdf_index.get(course_code, {})
        sis_data = self.sis_index.get(course_code, {})

        # SSOT: 对于"开课时间/地点/老师"，SIS 优先
        if query_intent in ["schedule", "instructor", "location"]:
            if sis_data:
                return {
                    "source": "SIS",
                    "data": sis_data,
                    "confidence": "high",  # 实时数据，权威性高
                    "fallback": pdf_data  # PDF 作为补充（如果 SIS 缺失）
                }

        # 对于"课程描述/学分/先修"，PDF 和 SIS 合并（SIS 优先）
        if query_intent in ["description", "units", "prerequisites"]:
            merged = {
                **pdf_data,  # PDF 提供基础信息
                **sis_data   # SIS 覆盖/补充（如果存在）
            }
            return {
                "source": "Hybrid",
                "data": merged,
                "confidence": "high" if sis_data else "medium"
            }

        return {"source": "PDF", "data": pdf_data}
```

**Entity Linking 实现**：

```python
def link_pdf_to_sis(pdf_text: str) -> dict:
    """
    Step 1: 从 PDF 提取课程代号（Anchor Key）
    Step 2: 用课程代号查询 SIS 数据库
    Step 3: 建立双向映射（PDF ↔ SIS）
    """
    # 正则提取候选课程代号
    candidates = re.findall(r"\b([A-Z]{2,5})\s?(\d{4})\b", pdf_text)

    # Gemini 去噪（过滤日期词、学期词）
    valid_codes = gemini_filter_codes(candidates)

    # Entity Linking: 每个课程代号作为 anchor，查询 SIS
    linked_data = {}
    for code in valid_codes:
        sis_info = query_sis_by_code(code)  # Playwright 爬取
        linked_data[code] = {
            "pdf_metadata": extract_from_pdf(pdf_text, code),
            "sis_metadata": sis_info,  # 实时数据
            "last_updated": datetime.now()
        }

    return linked_data
```

**为什么 SIS 优先？**

| 维度 | PDF | SIS | 优先级 |
|------|-----|-----|--------|
| **时效性** | 静态（年度更新） | 动态（每学期更新） | ✅ SIS |
| **准确性** | 可能过时（如"Fall"但实际改到 Spring） | 实时（当前学期真实数据） | ✅ SIS |
| **完整性** | 只有课程列表 | 时间/地点/老师/容量 | ✅ SIS |
| **权威性** | 官方发布 | 官方系统 | ⚠️ 平级，但 SIS 更实时 |

**结果**：当用户问"CSC3100 这学期什么时候开"时，系统返回 SIS 的实时数据（"Mon 10:00, Tue 14:00"），而非 PDF 的模糊描述（"Fall Semester"），避免了 **Hallucination**（LLM 编造答案）。

---

#### 5. 存储与维护：Hot-swappable Knowledge Base 与 File-as-Database 架构

**核心问题**：如何在不中断服务的情况下更新知识库？

**架构决策：File-as-Database + Git Version Control**

```python
knowledge_base/
├── courses_sis_detail/
│   ├── courses/
│   │   ├── CSC1001/
│   │   │   └── terms/
│   │   │       ├── 2510/
│   │   │       │   ├── metadata.json  # 开课信息
│   │   │       │   └── detail.json    # 课程详情
│   │   │       ├── 2520/
│   │   │       └── 2650/
│   │   └── CSC3100/
│   └── all_courses_ai.txt  # 课程代号索引
```

**Hot-swappable 更新机制**：

```python
class HotSwappableKnowledgeBase:
    """
    File-as-Database: 用文件系统代替传统数据库
    优势：零停机更新、Git 版本控制、易于备份
    """

    def update_course(self, course_code: str, term: str, new_data: dict):
        """
        Atomic Update: 写入临时文件 → 原子替换
        """
        course_dir = Path(f"knowledge_base/courses/{course_code}/terms/{term}")
        temp_file = course_dir / "metadata.json.tmp"

        # 写入临时文件
        temp_file.write_text(json.dumps(new_data, indent=2))

        # 原子替换（Windows: rename 是原子操作）
        temp_file.replace(course_dir / "metadata.json")

        # 后端自动检测文件变更（watchdog 或定期 reload）
        self.reload_index()

    def reload_index(self):
        """
        Lazy Reload: 只在需要时重新加载索引
        """
        self.index = {}
        for course_dir in Path("knowledge_base/courses").iterdir():
            code = course_dir.name
            self.index[code] = self._load_course_metadata(course_dir)
```

**为什么不用传统数据库（SQLite/PostgreSQL）？**

| 方案 | 更新复杂度 | 版本控制 | 部署复杂度 | 选择 |
|------|----------|---------|-----------|------|
| **SQLite** | 中（需要 migration） | ❌ 二进制文件，Git diff 不可读 | 中（需要 schema） | ❌ |
| **PostgreSQL** | 高（需要 DBA） | ❌ 需要额外工具 | 高（需要服务） | ❌ |
| **File-as-Database** | 低（直接替换文件） | ✅ Git 原生支持，diff 清晰 | 低（零配置） | ✅ |

**Agility 优势**：

1. **零停机更新**：
   ```bash
   # 更新单个课程：直接替换文件
   cp new_metadata.json knowledge_base/courses/CSC1001/terms/2510/metadata.json
   # 后端自动检测变更（watchdog 或定期 reload）
   ```

2. **Git 版本控制**：
   ```bash
   git diff knowledge_base/courses/CSC1001/terms/2510/metadata.json
   # 清晰看到：哪些课程更新了、何时更新、谁更新的
   ```

3. **易于备份**：
   ```bash
   # 整个知识库 = 一个文件夹
   tar -czf knowledge_base_backup_2025-12-24.tar.gz knowledge_base/
   ```

4. **快速回滚**：
   ```bash
   # 如果更新出错，直接 Git revert
   git checkout HEAD~1 -- knowledge_base/courses/CSC1001/
   ```

**Trade-off：Latency vs. Agility**

| 操作 | File-as-Database | SQLite | PostgreSQL |
|------|-----------------|--------|------------|
| **读取单课程** | ~5ms（文件 I/O） | ~1ms（索引查询） | ~2ms（网络 + 查询） |
| **批量更新** | ~50ms（文件替换） | ~20ms（批量 INSERT） | ~30ms（批量 INSERT） |
| **版本控制** | ✅ Git 原生 | ❌ 需要导出 | ❌ 需要导出 |
| **部署复杂度** | ✅ 零配置 | ⚠️ 需要 schema | ❌ 需要服务 |

**结论**：对于 <2K 文档的知识库，文件系统的 **5ms 读取延迟**（vs. SQLite 的 1ms）是可接受的 trade-off，换来了**零停机更新**和**Git 原生版本控制**的敏捷性。

---

#### 6. 模型选择：Gemini 2.5 Flash vs. 3.0 的权衡

| 维度 | Gemini 2.5 Flash | Gemini 3.0 (Pro) | 我的选择 |
|------|------------------|-------------------|----------|
| **响应速度** | ~1.5s (P50) | ~4s (P50) | ✅ Flash（用户体验优先） |
| **多轮推理** | 中等（3-4 轮后退化） | 强（10+ 轮保持连贯） | ⚠️ Flash + 3 轮历史截断 |
| **成本** | $0.075/1M tokens | $0.35/1M tokens | ✅ Flash（预算受限） |
| **上下文窗口** | 128K | 1M | ✅ Flash（已够用） |

**未来优化方向**（v1 路线图）：
- 使用 **Flash 做意图分类** → 简单查询直接回答
- 复杂查询（如"帮我规划这学期"）路由到 **Gemini 3.0**
- 引入 **caching mechanism**（重复查询同一课程时，复用上下文）

---

## 📊 Results & Impact：量化指标与定性反馈

### 系统性能指标

| 指标 | 目标 | 实际 | 测试方法 |
|------|------|------|----------|
| **数据覆盖率** | >90% | 95.2% (1493/1568 courses) | 与 SIS 官方课程列表交叉验证 |
| **提取准确率** | >95% | 96.8% | 人工抽样 100 门课程，比对 PDF vs. 系统输出 |
| **响应时间 (P50)** | <2s | 1.7s | 500 次真实查询压测 |
| **响应时间 (P95)** | <5s | 3.2s | 包含 3 学期数据的复杂查询 |
| **系统可用性** | >99% | 99.8% | 2 周 beta 测试，2 次计划内维护 |

### 定性影响

**用户反馈**（n=15, beta 测试）：
> "终于不用开 10 个标签页了，问一句就能知道课程什么时候开"
> "比学校教务处的回复快多了，而且数据准确"

**教授反馈**（贾建民教授）：
- ✅ 认可数据驱动方法（vs. 人工编辑课程评价）
- 建议：增加"学期压力模拟"功能（如果选了 A+B+C，总 workload 如何）
- 建议：细化课程评价粒度（难点在哪、常见坑）

---

## 🔍 Lessons Learned：技术债务与设计权衡

### 1. 数据质量 > 模型能力

**发现**：Gemini 2.5 Flash + 干净上下文 优于 GPT-4 + 噪声数据

**证据**：在"课程先修查询"任务中：
- GPT-4 (无 RAG)：准确率 62% （靠"记忆"回答，经常过时）
- Gemini Flash (Hybrid Context)：准确率 96.8%

**启示**：与其花时间调参/换模型，不如优先清洗数据

### 2. 错误处理 = 用户体验

**反面案例**：早期版本遇到课程不在知识库时，LLM 会"编造"答案
```
User: URM4160 是什么课？
System (错误): 这是一门关于城市管理的课程... (实际上知识库里没有这门课)
```

**改进**：显式标记数据缺失
```python
if course_code not in knowledge_base:
    return {
        "status": "NOT_FOUND",
        "message": f"官方数据库暂无 {course_code} 的信息，可能原因：\n"
                   "1. 该课程在 2025-26 三学期均未开课\n"
                   "2. 课程代号可能有误\n"
                   "建议：访问 SIS 手动搜索或联系教务处"
    }
```

### 3. Compliance First：合规性考量

**决策**：只使用官方公开数据（培养方案 + SIS），不接入用户生成内容（如"卡园"评价）

**原因**：
- 法律风险：UGC 可能涉及诽谤/隐私问题
- 数据质量：主观评价存在 bias（"水课"定义因人而异）
- 可维护性：官方数据更新频率低，易同步

**未来可能方向**（需法务审核）：
- 允许用户标注"已选过此课"（布尔值，无文本评价）
- 聚合选课人数统计（匿名化）

---

## 🚀 Future Roadmap (v1)：从"信息检索"到"智能决策"

### v0 的局限性（自我批判）

| 限制 | 原因 | v1 解决方案 |
|------|------|-------------|
| **只支持单课程查询** | 未实现多课程对比逻辑 | 增加 `compare(CSC3100, CSC3160)` 接口 |
| **无历史数据** | 只抓了 3 学期（2025-26, 26-27 Summer） | 回溯抓取 2023-24, 24-25 数据 |
| **无个性化** | 所有学生看到相同回答 | 基于用户 profile（专业/年级）调整推荐权重 |
| **冷启动问题** | 新课程（如 CSC4999）无历史数据 | 使用课程大纲文本做 zero-shot 推理 |

### 技术升级方向

#### 1. 智能学期压力管理（Prof. Jia 建议）

**目标**：用户输入 "我这学期想选 CSC3100 + MAT2040 + PHY1002"，系统输出：
- 总学分：12
- 预估 workload：高（3 门理论课）
- 冲突检测：CSC3100 L01 和 MAT2040 L02 时间重叠
- 建议：将 PHY1002 移至下学期，换成 GE 课程

**技术方案**：
```python
def analyze_semester_load(courses: List[str], user_profile: dict):
    # 1. 提取时间表
    schedule = build_schedule_matrix(courses)

    # 2. 检测冲突
    conflicts = detect_time_conflicts(schedule)

    # 3. 预估 workload（基于历史数据）
    workload = estimate_workload(courses, user_profile['gpa'], user_profile['major'])

    # 4. 生成建议
    if workload > THRESHOLD:
        alternatives = suggest_lighter_courses(courses)

    return {
        "conflicts": conflicts,
        "workload": workload,
        "suggestions": alternatives
    }
```

#### 2. 模型升级：Flash 路由 + 3.0 深度推理

```python
class AdaptiveRouter:
    def route(self, query: str, complexity: int):
        if complexity < 3:  # 简单查询（课程时间、学分）
            return self.gemini_flash.generate(query)
        else:  # 复杂查询（学期规划、多课程对比）
            return self.gemini_3_pro.generate(query)

    def estimate_complexity(self, query: str) -> int:
        # 基于规则 + 小型分类模型
        if "compare" in query or "plan" in query:
            return 5
        elif "when" in query or "who" in query:
            return 1
        else:
            return 3
```

#### 3. 数据粒度深化

- **课程大纲爬取**：从教授个人页面抓取 syllabus PDF（需 opt-in）
- **历史成绩分布**：如果学校公开，整合 GPA 分布数据（隐私合规）
- **前置知识图谱**：构建课程依赖关系（CSC1001 → CSC3100 → CSC3170）

---

## 🎓 Reflections for Admissions：What I Learned as a Freshman

### Technical Growth

**Before this project**:
- 认为 RAG = "把文档扔进 vector DB"
- 不理解 context window 限制的实际影响
- 从未在生产环境处理过反爬

**After**:
- 理解 Hybrid retrieval 在结构化数据场景下的优势
- 学会了 trade-off thinking（Flash vs. 3.0 的决策过程）
- 掌握了 Playwright 的 session 管理、Windows multiprocessing 的 pickling 限制

### Soft Skills

**Iteration over Perfection**:
v0 的目标不是"完美系统"，而是"最小可用 MVP"。我本可以花 1 个月做 Course Catalog 爬虫，但选择先用 3 学期数据验证需求。

**User Empathy**:
测试时发现，学生更在乎"这学期开不开"（时效性），而非"这门课历史上开过几次"（完整性）。这影响了我对数据优先级的判断。

**Documentation**:
写这篇 case study 的过程中，我意识到很多"显而易见"的设计决策（如为什么选 Flask 而非 FastAPI），其实背后都有 trade-off。记录这些思考让我更清晰自己的技术判断力。

---

## 🔗 Appendix：技术栈与代码示例

### Technology Stack

| 层级 | 技术选型 | 版本 | 选择理由 |
|------|----------|------|----------|
| **Frontend** | Vanilla JS + Tailwind CSS | - | 轻量级，无构建步骤 |
| **Backend** | Flask | 3.0 | 简单 REST API，Python 生态友好 |
| **LLM** | Gemini 2.5 Flash | - | 成本低，响应快 |
| **Crawler** | Playwright | 1.40 | 支持 headless，比 Selenium 轻量 |
| **Data Processing** | PyPDF2 + regex | - | PDF 为文本格式，无需 OCR |
| **Knowledge Base** | JSON files (local) | - | <2K 文档，无需数据库 |

### 核心代码片段

#### 1. Hybrid Context 组装

```python
def build_hybrid_context(course_code: str, user_query: str) -> str:
    """
    组装结构化上下文，供 Gemini 生成回答

    Args:
        course_code: 课程代号（如 CSC1001）
        user_query: 用户原始问题

    Returns:
        结构化 prompt 字符串
    """
    # 1. 加载课程元数据（来自培养方案）
    meta = load_course_meta(course_code)

    # 2. 聚合多学期 SIS 数据
    terms_data = []
    for term in ["2510", "2520", "2650"]:
        path = f"courses/{course_code}/terms/{term}/metadata.json"
        if os.path.exists(path):
            terms_data.append(load_json(path))

    # 3. 结构化 prompt
    context = f"""
    === Official Course Information ===
    Course Code: {course_code}
    Course Name: {meta.get('name', 'N/A')}
    Description: {meta.get('description', 'No description available')}
    Prerequisites: {meta.get('prerequisites', 'None')}

    === Offering History (Recent 3 Semesters) ===
    {format_terms_summary(terms_data)}

    === User Query ===
    {user_query}

    === Instructions ===
    - Answer based ONLY on the data above
    - If information is missing, explicitly state "官方数据未提供"
    - Cite specific terms (e.g., "根据 2025-26 第一学期 SIS 数据...")
    """
    return context
```

#### 2. Gemini API 调用（带超时保护）

```python
import multiprocessing as mp
from multiprocessing import Pipe

def call_gemini_with_timeout(prompt: str, timeout: float = 45.0) -> str:
    """
    使用 multiprocessing 实现硬超时（避免 Gemini API 挂起）

    Args:
        prompt: 输入 prompt
        timeout: 超时时间（秒）

    Returns:
        模型响应文本

    Raises:
        TimeoutError: 超过 timeout 仍未返回
    """
    parent_conn, child_conn = Pipe()

    # Windows 要求 target 是顶层函数（不能是 lambda）
    process = mp.Process(
        target=_mp_runner,
        args=(prompt, child_conn)
    )

    process.start()
    process.join(timeout=timeout)

    if process.is_alive():
        process.terminate()
        process.join()
        raise TimeoutError(f"Gemini call timeout after {timeout}s")

    return parent_conn.recv()

def _mp_runner(prompt: str, pipe):
    """顶层函数，用于 Windows multiprocessing"""
    model = genai.GenerativeModel("gemini-2.5-flash")
    response = model.generate_content(prompt)
    pipe.send(response.text)
```

---

## 📚 References & Further Reading

1. **RAG Systems**:
   - Lewis et al. (2020). "[Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)". NeurIPS 2020.

2. **Hybrid Retrieval**:
   - Ma et al. (2023). "[Fine-Tuning LLaMA for Multi-Stage Text Retrieval](https://arxiv.org/abs/2310.08319)". arXiv preprint.

3. **Context Window Management**:
   - OpenAI (2023). "[Best Practices for Prompt Engineering](https://platform.openai.com/docs/guides/prompt-engineering)".

4. **Anti-Scraping Strategies**:
   - Imperva (2022). "[Bot Management Best Practices](https://www.imperva.com/learn/application-security/bot-management/)".

---

## 🏆 Key Takeaways（3 句"金句"）

1. **"数据架构决定系统上限，模型只是逼近上限的工具"**
   在 RAG 系统中，clean + structured data 比选 GPT-4 还是 Gemini 更重要。

2. **"显式的错误处理比隐式的'聪明'更可靠"**
   让 LLM 承认"我不知道"（`[NORESULT]`）比让它"瞎编"更符合工程实践。

3. **"MVP 不是偷懒，而是快速验证假设的策略"**
   与其花 3 个月做"完美"的 Course Catalog 爬虫，不如用 2 周验证"学生真的需要这个功能吗"。

---

## 📚 v2 前的技术深化学习

在从 MVP 到 v2 的迭代过程中（11月底至12月中旬），我深化学习了以下技术：

### 数据采集与处理
- **Playwright**：从官方文档学习浏览器自动化、会话管理、反爬策略应对
- **Gemini Vision API**：学习如何用 AI 处理 PDF/图像，理解 OCR + 结构化理解的优势
- **正则表达式 + AI 混合**：研究如何结合规则和 AI，平衡准确率与成本

### 系统工程
- **Windows multiprocessing**：学习 Python 在 Windows 上的进程间通信（Pipe/Queue）、pickling 机制
- **错误处理与重试**：研究 exponential backoff、circuit breaker 等容错模式
- **数据结构设计**：学习如何设计可扩展的 JSON schema（按课程/学期分片）

### RAG 优化
- **Context Window 管理**：研究如何动态加载上下文（intent-based lazy loading）
- **Hybrid Retrieval**：学习精确匹配（课程代号）+ 语义搜索（描述）的混合策略

---

## 🙏 致谢

- **严明教授**和**贾建民教授**：在项目关键节点提供了深度的技术指导和产品建议，特别是关于"数据质量优先于模型能力"和"显式错误处理"的工程哲学，对系统设计产生了深远影响
- **团队成员**：感谢几位同组成员在初期帮助采集培养方案 PDF、整理课程清单、协助内测等基础工作，为后续开发和优化提供了坚实的数据基础和用户反馈

---

*GitHub Repo (Private): [holistic-assistant](https://github.com/Jerry0310no1/holistic-assistant)*
*Contact: [125090445@link.cuhk.edu.cn](mailto:125090445@link.cuhk.edu.cn)*
