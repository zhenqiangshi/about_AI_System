

基于ragflow-0.27 以上

```text
Question
  │
  ▼
Planner Agent              (Phase 1 — 扇出分解)
  │ query fanouts
  ▼
RAG Agent / Search Fanout  (Phase 2 — 并行检索所有扇出)
  │ 检索结果 + 草稿
  ▼
Sufficient Context Agent   (Phase 3 — 审查原问题 + 片段 + 中间草稿)
  │ suff / insuff
  ▼
Synthesis + Gap analysis   (Phase 4 — 查询重写，针对缺口再检索)
  │
  ▼
Synthesis Agent            (Phase 5 — 最终答案合成)
```



基于ragflow agentic rag 的代码解读（GPT），基于LangGraph

让我查看这个目录下的文件结构：现在让我获取一些关键文件的摘要：现在我有足够的信息来生成一份完整的解读。让我创建一份详细的技术总结文档。

## 📘 **RAGFlow Advanced RAG 架构完整解读**

---

## 目录结构与模块分解

```
rag/advanced_rag/
├── __init__.py                     # 模式配置导出（低级）
├── agentic_rag.py                  # RAG工具类（能力层）
├── agentic_rag_graph.py            # 代理搜索图（LangGraph编排）
├── think_log.py                    # 推理日志处理
└── harness/                        # 执行引擎核心
    ├── config.py                   # 模式规范（low/medium/high/ultra）
    ├── action_session.py           # 槽表驱动的搜索会话（106K行）
    ├── stats.py                    # LLM计数和使用统计
    ├── keywords.py                 # 关键词提取与加权
    ├── memory.py                   # 停词表和缓存管理
    ├── grep_sed_narrow.py          # 关键词窄化（grep+sed风格）
    ├── arithmetic.py               # 数值计算验证
    ├── chunk_utils.py              # 文本块工具
    ├── orchestrator/               # 搜索执行器（direct/sufficient_context/query_rewriter）
    ├── prompts/                    # 阶段性提示模板
    ├── tools/                      # 检索工具集（search/navigation/explore）
    └── knowlege_compile/           # 知识编译（证据行生成）
```

---

## 🎯 **核心技术要点**

### **1. 分层架构设计**

| 层级       | 模块                     | 职责                                                   |
| -------- | ---------------------- | ---------------------------------------------------- |
| **API层** | `agentic_rag.py`       | 工具类 RAGTools，绑定两个外部工具：`rag()`、`summarize_document()` |
| **编排层**  | `agentic_rag_graph.py` | LangGraph 5阶段流程编排                                    |
| **执行层**  | `harness/*`            | 检索、评审、重写等原子操作                                        |
| **推理层**  | `action_session.py`    | 模型驱动的槽表研究引擎                                          |

**关键特性**：
- ✅ **不暴露内部工具**：图的步骤（检索、评审等）直接调用，不通过模型工具调用
- ✅ **两轨制**：快速路径（low）vs 完整代理路径（medium/high/ultra）
- ✅ **分离存储和评审**：池最多 60 个块，但 SCA 只看排序后的 60 个视图

---

### **2. 思维模式（Thinking Mode）**

```python
# config.py 中的模式规范
THINKING_MODES = {
    "low": ModeSpec(
        agentic=False,              # 无代理图
        enable_sca=False,           # 无充分性检查
        use_fanout=False,           # 无分解规划
        sca_max_rounds=0,
        label="low"
    ),
    "medium": ModeSpec(
        agentic=True,
        enable_sca=False,           # SCA 仅用于信息
        use_fanout=False,           # 直接用原始问题
        sca_max_rounds=1,
    ),
    "high": ModeSpec(
        agentic=True,
        enable_sca=True,            # ✨ 激活迭代循环
        use_fanout=True,            # ✨ 规划 + 预取
        sca_max_rounds=3,           # 最多 3 轮研究
    ),
    "ultra": ModeSpec(
        agentic=True,
        enable_sca=True,
        use_fanout=True,
        sca_max_rounds=5,           # ✨ 超深迭代（成本更高）
    ),
}
```

**流程差异**：
```
low:     formalize → direct_search → answer            (无思考)
         
medium:  formalize → rag_agent → draft → sca → answer  (单轮，SCA 信息用途)

high:    formalize → planner → prefetch → 
         rag_agent → draft → sca ↔ query_rewrite      (迭代循环，3轮上限)
         ↓ 充分时 ↓
         answer

ultra:   高模式的基础上，sca_max_rounds=5              (深度探索，5轮)
```

---

### **3. 槽表（Slot Table）驱动的研究**

#### **3.1 核心数据结构**

```python
class Variable:
    """一个待解决的未知数"""
    id: int                         # 不可变标识
    type: str                       # "entity" / "aspect" / "answer"
    question_clues: list[str]      # 该槽的搜索线索（来自分解）
    discovered_clues: list[str]    # 研究中发现的线索（累积）
    candidate: str | None          # 最终候选答案
    candidate_strength: float | None # 置信度 (0-1)
    
    def filled(self) -> bool:
        return self.candidate is not None and self.candidate != ""

class State:
    """槽表树节点"""
    state: list[Variable]          # 当前槽列表
    depth: int                     # 树深（防止无限分解）
    id: str                        # 唯一标识（时间戳 + 随机字节）
    retrieved_evidence_ids: list   # 该树获得的证据ID
```

#### **3.2 槽表生命周期**

```
PHASE 1: 规划器分解
─────────────────────
问题 "Paris population in 2019 and GDP?"
   ↓ (LLM 分解)
fan-outs: ["Paris population 2019", "Paris GDP 2019"]
   ↓
_build_slot_table()
   ↓
槽表: [
    Variable(id=0, type="entity", question_clues=["Paris population 2019"]),
    Variable(id=1, type="entity", question_clues=["Paris GDP 2019"]),
]
depth=0


PHASE 2: 并行搜索
──────────────────
for slot in unresolved():
    session = action_session(direction=slot.question_clues[0])
    → 该 session 运行槽特定的搜索和推理
    → 可能产生 new_states（分支）或 found_answer（终止）
    
PHASE 4: 合并结果
──────────────────
for session_result in results:
    if session_result.new_states:
        slot_table = _merge_slot_patch(slot_table, new_states)
    if session_result.found_answer:
        collected_answer = session_result.found_answer
```

#### **3.3 强度选择合并**

```python
def _merge_slot_patch(base, branch):
    """并发的 session 结果合并到共享表中"""
    for v_base in base.state:
        v_branch = branch_by_id.get(v_base.id)
        if v_branch is None:
            继续用 v_base
        else:
            # ✨ 关键：只有当分支强度更高时才替换
            if (v_branch.candidate_strength or 0) > (v_base.candidate_strength or 0):
                采用 v_branch.candidate
            else:
                保持 v_base.candidate
```

**为什么**：session 并发运行，完成顺序不确定。无条件的"分支赢"会导致结果不确定（同一输入不同顺序得不同答案）。强度最高原则确保**确定性**且**最优**。

---

### **4. 两通道检索架构**

#### **Channel A：精确匹配（BM25 + 窄化）**

```python
# 行 637-690: _fanout_search
terms = _query_to_terms("Paris population 2019")
    # → ["paris", "population", "year"]
    
keyed = [t for t in terms if len(t) >= 3 and t not in STOPWORDS]
    # → ["paris", "population"]

bm25_candidates = await bm25_search(
    question="Paris population 2019",
    keywords="paris population",  # ← entity 权重x3
    top_n=60
)

narrowed = narrow_by_terms(
    candidates=bm25_candidates,
    terms=keyed,
    max_out_chars_per_chunk=1200,
)
kept_a = narrowed.get("kept", [])[:top_n]
```

**目标**：捕获包含"Paris"、"population"等实体名的精确文本。

#### **Channel B：语义相关（向量混合，绕过窄化）**

```python
# 窄化会过滤掉：
#   - 语义相关但不含关键词的块
#   - 例如："The French capital's demographics in 2019 were..."
#     （含"demographics"而非"population"）

kept_b = []
seen_ids_a = {_chunk_id(c) for c in kept_a}  # 已取出的 ID

hybrid_results = await hybrid_search(
    question="Paris population 2019",
    top_n=30
)

for c in hybrid_results:
    if _chunk_id(c) not in seen_ids_a:        # 去重：避免重复
        kept_b.append(c)
        if len(kept_b) >= 4:
            break
```

**目标**：补充 Channel A 漏掉的**语义相关**但**表面无交集**的内容。

#### **证据行快速通道（Channel 0）**

```python
# _collect_evidence() - 行 692-733
# 对于编译的数据集（已生成事实+引用）

evidence_chunks = await recall_dataset_claims(
    question="Paris population 2019",
    top_n=max(2, top_n)
)
# 返回：[
#   {
#     "name": "Paris 2019 Census",
#     "quote": "Paris had 2.161 million inhabitants...",
#     "chunk_ids": [chunk_id_1, chunk_id_2],  # 引用的原始块
#   },
#   ...
# ]

# 在池中合成为规范的块
for ev_row in evidence_chunks:
    chunk = {
        "chunk_id": f"claim_{hash(doc_id:name)}",
        "content_with_weight": f"[evidence] {name} — {desc}\nEvidence (verbatim): \"{quote}\"",
        "source_chunk_ids": chunk_ids,  # ← 记录来源
    }
```

**优化**：证据行是**原子命题**（事实+引用），密度高（≈1.2K 字符 = 300 tokens）。一个证据行顶多个普通块。优先摄入，保留给 raw 通道的窄配额。

---

### **5. 充分性代理（SCA）工作流**

#### **5.1 证据视图排序与选择**

```python
def _select_sca_view(chunks: list, focus_terms: list[str], cap: int = 60) -> tuple[list, str]:
    """从总池中排序出 SCA 评审视图"""
    
    # 评分：相似度 × 0.45 + 关键词覆盖 × 0.45 + 新鲜度 × 0.1
    def _score(chunk):
        rel = chunk.get("similarity", 0.0)      # 向量相似度
        cov = sum(1 for t in focus_terms if t in chunk["content"])
        cov_ratio = cov / len(focus_terms)      # 术语覆盖比
        fresh = min(i / 20.0, 0.2)              # 新一轮的块得分高
        
        return rel * 0.45 + cov_ratio * 0.45 + fresh * 0.1
    
    ranked = sorted(enumerate(chunks), key=_score, reverse=True)
    
    # ✨ 证据行优先（答案材料）
    evidence = [c for _, c in ranked if is_evidence(c)]
    rest = [c for _, c in ranked if not is_evidence(c)]
    
    view = (evidence + rest)[:cap]
    
    # 生成稳定标识（用于检测重复评审）
    ident = "|".join(sorted(chunk_id for c in view))
    return view, hash(ident)
```

#### **5.2 不生产轮检测**

```python
# 行 1185-1189
prev_id = state.get("sca_view_id")
view_id = _select_sca_view(chunks, ...)[1]

if prev_id and view_id == prev_id:
    # 📌 同一视图评审了两次
    # → SCA 会给出相同的不足反馈
    # → 后续搜索也取不到新证据（池满了或已穷尽）
    # → 再搜也是徒劳，停止迭代
    
    _LOG.info("[SCA] view UNCHANGED; closing out (loop futility)")
    return {"verdict": {"status": "INSUFFICIENT"}, "no_progress": True}
```

---

### **6. 查询重写与自适应规划**

#### **6.1 信息增强重写**

```python
# 行 1281-1304
research_context = """
Previously searched queries and their outcomes:
- "Paris population 2019" (round 0: 8 new passages)
- "GDP France 2019" (round 1: 0 new passages)

Evidence currently at hand (top stored snippets):
- [1] France's GDP reached €2.9 trillion in 2019...
- [2] Paris had 2.161 million residents (census)...
"""

rewritten = await rewrite_gap_to_query(
    question="Paris population in 2019 and GDP?",
    gaps=[
        ("GDP of Paris specifically", "not national GDP"),
        ("Per capita income", "wealth measure"),
    ],
    research_context=research_context,
)
# → [
#   {"query": "Paris metropolitan area GDP 2019"},
#   {"query": "Île-de-France regional economic data"},
# ]
```

**与传统去重对比**：
- ❌ 规则去重：依赖硬编码规则，易漏、易误
- ✅ 信息去重：重写器看到完整历史和当前证据，**自动寻找未覆盖的角度**

#### **6.2 自适应规划（Decompose）**

```python
# 行 1344-1376: 将不足的缺口升级为新槽

if gaps and slot_table.depth < _MAX_SLOT_DEPTH:
    # 例如 SCA 反馈："missing Per capita income"
    
    for what, hint in gaps:
        if len(slots) >= _MAX_SLOTS_TOTAL:
            break
        if already_known(what):
            continue
        
        # 创建新槽：gap 变成一个独立的研究目标
        new_slot = Variable(
            id=next_id,
            type="derived_entity",
            question_clues=[what, hint],
        )
        slot_table.state.append(new_slot)

# 下一轮 rag_agent 会为这个新槽启动 action_session
```

**对标**：APT-RAG 的"自适应规划"，但我们：
- ✅ 限制深度 `<= 3` 和总数 `<= 8`（防止爆炸）
- ✅ 用 SCA 的**已有反馈**（无额外 LLM 调用）

---

### **7. 预算与超时控制**

#### **7.1 多层级预算**

```python
# 行 69-81
_TOTAL_BUDGET_S = 180.0             # 整个问题的全局上限
_MIN_ROUND_HEADROOM_S = 50.0        # 启动新轮需要的最小余地
_PASS_TIMEOUT_S = 120.0             # 单次搜索轮的超时
_PREFETCH_TIMEOUT_S = 90.0          # 预取的超时
_DRAFT_TIMEOUT_S = 60.0             # 初稿合成的超时
_SCA_TIMEOUT_S = 60.0               # SCA 评审的超时
_REWRITE_TIMEOUT_S = 45.0           # 查询重写的超时
```

#### **7.2 路由守卫**

```python
def _route_sca(state: AgenticState) -> str:
    """在 SCA 后做出路由决策"""
    
    if state.get("no_progress"):
        return "formalize_answer"
    
    # 池满了 → SCA 看不到新块 → 无法改变判决
    if len(tools.kbinfos["chunks"]) >= _SCA_VIEW_CAP and state.get("search_rounds") >= 1:
        return "formalize_answer"
    
    if not enable_sca:
        return "formalize_answer"
    
    # 充分性判决 + 预算充裕 + 轮次未满 → 继续研究
    if (
        state.get("verdict", {}).get("status") == "INSUFFICIENT" and
        state.get("search_rounds", 0) < sca_max_rounds and
        _remaining_s(state) > _MIN_ROUND_HEADROOM_S
    ):
        return "query_rewrite"
    
    return "formalize_answer"
```

---

### **8. 缓存与去重机制**

#### **8.1 跨 RAG 调用缓存**

```python
# agentic_rag.py 行 837-858
# 当外部 LLM 重复提问时（稍作改述）

qk = _question_keywords("Paris population in 2019")
    # → (sig_words={"paris", "population", "2019"}, numbers={"2019"})

for cached_q, (cached_answer, cached_gram) in self._rag_cache.items():
    if _cache_similar(qk, cached_gram):
        # 关键词 overlap >= 0.6 AND >= 2 个共同词
        # AND 数字集相同（防止"2019"vs"2015"混淆）
        
        _LOG.info("[Cache] Reused prior answer for similar question")
        return cached_answer  # ← 跳过整个图
```

**何时跳过缓存**：
```python
# 仅当上一轮不足反馈明确要求新搜索时
if self._rag_verdict.get("status") in {"INSUFFICIENT", "UNANSWERABLE"}:
    skip_cache = True  # 强制重新执行
```

#### **8.2 每请求检索缓存**

```python
# agentic_rag.py 行 330
self.search_cache: dict = {}  # key: (question + scope)

# 在同一轮内，同一查询不重复检索
```

---

### **9. 原始问题恢复机制**

```python
# agentic_rag.py 行 155-175
def _resolve_effective_question(question: str, original_user_question: str) -> str:
    """使用用户的原始问题而非模型重写版本（关键！）"""
    
    # 场景：
    # 用户："How many days after X's death did Y die?"  (多跳，需2个实体)
    # 外部LLM重写："When did Y die?"                   (删了X，只留第一跳)
    # 
    # 内图应该用原始问题，否则回答变成："Y died in 1965"
    # 而非："23天后"（多跳结果）
    
    if not original_user_question:
        return question
    
    qk = _question_keywords(question)
    ok = _question_keywords(original_user_question)
    
    # 仅当两者关键词 overlap >= 2 个时才替换（防止混淆不同问题）
    if len(qk & ok) >= 2:
        return original_user_question
    
    return question
```

**为什么重要**：
- ❌ 问题："谁购买了 X？他死后多少天 Y 死了？"
- ❌ 模型压缩："Y 死了吗？"（删了第二部分）
- ✅ 恢复原问题 → 槽表包含两个实体 → 能回答"23天"

---

## 📊 **完整流程图（以 high 模式为例）**

```
                              START
                                ↓
                    ┌───────────────────────┐
                    │ formalize_question    │
                    │  → 问题 + 关键词      │
                    └───────────┬───────────┘
                                ↓
                    ┌───────────────────────┐
                    │ planner (Phase 1)     │
                    │  → 2-5 个子问题(fanout)│
                    │  → 构建槽表           │
                    └───────────┬───────────┘
                                ↓
                    ┌───────────────────────┐
                    │ prefetch (可选)       │
                    │  → 并行搜索所有fanout │
                    │  → 预填 kbinfos       │
                    └───────────┬───────────┘
                                ↓
              ┌─────────────────────────────────────┐
              │   rag_agent (Phase 2, 循环)        │
              │  ┌──────────────────────────────┐  │
              │  │ FOR 每个未解槽:              │  │
              │  │  - 预填证据行（跳session）  │  │
              │  │  - 启动 action_session      │  │
              │  │  - 合并结果 (_merge_patch)  │  │
              │  └──────────────────────────────┘  │
              │  → 输出：draft + unresolved_slots  │
              └─────────────┬──────────────────────┘
                            ↓
                    ┌───────────────────────┐
                    │ draft (Phase 3)       │
                    │  → 是否生成初稿       │
                    │  → 若无则合成         │
                    └───────────┬───────────┘
                                ↓
                    ┌───────────────────────┐
                    │ sca (Phase 3)         │
                    │  → 排序视图           │
                    │  → SCA 评审           │
                    │  → 充分 / 不充分      │
                    └───────────┬───────────┘
                                ↓
                           ┌────────────┐
                           │   路由     │
                           └──┬──────┬──┘
                  充分 / 预算尽│       │不充分 & 轮次<3
                              │       │
                              ↓       ↓
                        formalize   query_rewrite
                        _answer     (Phase 4)
                           ↑         ↓
                           │    (重新搜索)
                           │         │
                           └─────←───┘
                                ↓
                    ┌───────────────────────┐
                    │ formalize_answer      │
                    │ (Phase 5)             │
                    │  → 最终答案合成       │
                    │  → 流式输出令牌       │
                    └───────────┬───────────┘
                                ↓
                              END
```

---

## 🔍 **关键算法细节表**

| 算法 | 文件 | 行数 | 核心逻辑 |
|------|------|------|---------|
| **两通道搜索** | agentic_rag_graph.py | 637-690 | BM25+窄化 vs 向量+绕过 |
| **槽表合并** | agentic_rag_graph.py | 1928-1982 | 强度最高原则 |
| **证据视图排序** | agentic_rag_graph.py | 121-158 | 相似度+覆盖+新鲜度 |
| **不生产轮检测** | agentic_rag_graph.py | 1185-1189 | 视图 ID 哈希对比 |
| **信息增强重写** | agentic_rag_graph.py | 1281-1304 | 完整历史+当前证据 |
| **自适应规划** | agentic_rag_graph.py | 1344-1376 | 缝隙升级为槽（深度限制） |
| **证据行预填** | agentic_rag_graph.py | 1703-1749 | 术语覆盖 >= 60% 跳 session |
| **跨调用缓存** | agentic_rag.py | 837-858 | 关键词重叠 + 数字匹配 |
| **原始问题恢复** | agentic_rag.py | 155-175 | 多跳保护（>= 2 共词） |

---

## 💡 **设计哲学总结**

| 原则 | 体现 |
|------|------|
| **分离存储和评审** | 池 60 个块，SCA 看 60 个视图（去重、排序、优先化） |
| **确定性并发** | 槽表合并用强度最高，非顺序依赖 |
| **代理胜于规则** | 查询重写看完整历史，自动寻找缝隙，而非硬规则 |
| **预算为王** | 多层超时 + 路由守卫，确保全流程 180 秒内完成 |
| **速度优先** | 证据行快速通道、session 预填、跨调用缓存 |
| **多跳保护** | 原始问题恢复、槽表类型化、强度选择 |
| **渐进深化** | low < medium < high < ultra，成本递增 |

---

## 🚀 **性能优化总结**

1. **证据行快速通道**：跳过 session（最大成本），直接匹配
2. **批证据处理**：共享证据的槽在一个 LLM 调用中回答（APT-RAG）
3. **缓存三层**：
   - 跨调用（关键词相似）
   - 每请求（检索去重）
   - 每环节（内存）
4. **两通道搜索**：Channel B 绕过窄化，捕获语义相关但表面无交集的块
5. **不生产轮检测**：池满或穷尽时早停
6. **预算守卫**：路由前检查时间，避免无意义的轮

---

## 📚 **代码示例：完整 RAG 调用**

```python
# 使用者代码
from rag.advanced_rag.agentic_rag import RAGTools

tools = RAGTools(
    tenant_ids=["tenant_123"],
    chat_mdl=llm_bundle,
    kb_ids=["kb_1", "kb_2"],
    thinking_mode="high",           # ← 启用完整代理
    original_user_question="Paris population 2019 and GDP?",
)

# 外部 LLM 调用工具
answer = await tools.rag(question="Paris: 2019 statistics?")

# tools 内部执行流程（隐藏）：
# 1. RAGTools.rag() 调用
# 2. 检查缓存（关键词相似）
# 3. 调用 run_agentic_rag(tools, messages)
#    ├─ 图编排开始（build_agentic_graph）
#    ├─ formalize_question
#    ├─ planner → fanouts
#    ├─ prefetch → 预初始化 kbinfos
#    ├─ rag_agent 循环（最多 3 轮）
#    │  ├─ action_session × N（并发）
#    │  ├─ 槽表合并
#    │  └─ 生成初稿
#    ├─ SCA 评审 → 充分 / 不充分
#    ├─ query_rewrite（若不充分）
#    └─ formalize_answer → 流式答案
# 4. 缓存答案
# 5. 返回并追加充分性反馈
```

---

**这是一个工业级的多代理 RAG 系统，融合了检索优化、推理编排、预算管理和缓存策略，在保持成本控制的同时最大化答案质量。**

---

### 名词解释

我来查看代码文件，获取这些概念的精确定义。现在让我用代码中的具体定义来为你详细解释这些核心概念。

---

# 📖 **RAGFlow 核心名词详解**

## 1️⃣ **SCA = Sufficient Context Agent（充分性代理）**

### 定义
从 `agentic_rag_graph.py` 的第 121-158 行代码看，**SCA 是一个评审代理，在每一轮检索后评估已有的证据是否足以回答问题**。

### 工作流程

```
第 N 轮搜索完成
    ↓
[SCA 步骤]
    ├─ 收集所有已获得的证据块（chunks）
    ├─ 用搜索项 + 相似度 + 新鲜度对块排序
    ├─ 选出排序后的前 60 个块作为"视图"
    ├─ 询问 LLM："这 60 个块能回答用户问题吗？"
    │
    └─ LLM 判决：
       ├─ SUFFICIENT（充分）→ 进入答案合成，停止迭代
       ├─ INSUFFICIENT（不充分）→ 输出缺漏的信息
       └─ UNANSWERABLE（无法回答）→ 返回部分答案
          
缺漏反馈（如果不充分）
    ↓
[Query Rewriter]
    └─ 基于缺漏，自动生成新搜索查询
       （例如：缺少"GDP"→ 生成"Paris GDP"查询）
    
新一轮搜索...
```

### 代码实例

```python
# 行 121-158: _select_sca_view
def _select_sca_view(chunks: list, focus_terms: list[str], cap: int = 60) -> tuple[list, str]:
    """从总池中排序出 SCA 评审视图"""
    
    # ✨ 评分公式：相似度 × 0.45 + 关键词覆盖 × 0.45 + 新鲜度 × 0.1
    def _score(chunk):
        rel = chunk.get("similarity", 0.0)      # 向量相似度 (0-1)
        cov = sum(1 for t in focus_terms if t in chunk["content"])
        cov_ratio = cov / len(focus_terms)      # 术语覆盖比
        fresh = min(i / 20.0, 0.2)              # 新一轮的块得分高
        
        return rel * 0.45 + cov_ratio * 0.45 + fresh * 0.1
    
    ranked = sorted(enumerate(chunks), key=_score, reverse=True)
    
    # ✨ 证据行优先（答案材料）
    evidence = [c for _, c in ranked if is_evidence(c)]  # claim 开头的块
    rest = [c for _, c in ranked if not is_evidence(c)]
    
    view = (evidence + rest)[:cap]  # ← 前 60 个
    
    # 生成稳定标识（用于检测重复评审）
    ident = "|".join(sorted(chunk_id for c in view))
    return view, hash(ident)
```

### 配置参数（从 config.py）

| 模式 | 使用 SCA | 最大轮数 |
|------|---------|---------|
| **low** | ❌ 否 | 0 |
| **medium** | ⚠️ 信息用途 | 3 |
| **high** | ✅ 是（决策用）| 3 |
| **ultra** | ✅ 是（决策用）| 5 |

---

## 2️⃣ **槽位 / 槽表 = Variable / State（变量与状态树）**

### 定义
从 `action_session.py` 的第 89-134 行，**槽位是待解决的"未知数"**，**槽表是这些变量组成的树状结构**。

### 核心类定义

```python
@dataclass
class Variable:
    """一个待解决的未知数"""
    
    id: int                         # 不可变唯一标识（跨树补丁）
    type: str                       # "entity" / "aspect" / "answer"
    question_clues: list            # 该槽的搜索线索（来自规划器分解）
    discovered_clues: list          # 研究中发现的线索（累积）
    candidate: str | None           # 最终候选答案
    candidate_strength: float | None # 置信度 (0-1，由模型给出)
    
    def filled(self) -> bool:
        """判断该槽是否已被解决"""
        return bool(self.candidate)


@dataclass
class State:
    """槽表树节点"""
    
    state: list                     # 该树的所有变量（槽）
    depth: int = 0                  # 树深（防止无限分解）
    id: str = ""                    # 唯一标识（时间戳+随机字节）
    retrieved_evidence_ids: list    # 该树获得的证据 ID
    
    def unresolved(self) -> list:
        """返回未填充的槽"""
        return [v for v in self.state if not v.filled()]
    
    def brief(self) -> str:
        """返回简洁表示：d1(+..+.) 表示深度1，其中+表示已填，.表示未填"""
        marks = ["+" if v.filled() else "." for v in self.state]
        return f"d{self.depth}({''.join(marks)})"
```

### 生命周期示例

#### **例题：Paris 2019 人口和 GDP**

```
PHASE 1: 规划器分解
─────────────────────
问题: "Paris population in 2019 and GDP?"

↓ [LLM 分解成多个独立槽]

State(depth=0) = [
    Variable(id=0, type="entity", question_clues=["Paris population 2019"]),
    Variable(id=1, type="entity", question_clues=["Paris GDP 2019"]),
]
brief: "d0(..)"  ← 两个槽，都未填


PHASE 2: 第一轮搜索（并发）
──────────────────────
for slot in state.unresolved():
    session = action_session(direction=slot.question_clues[0])
    
槽 0 搜索：→ 发现"Paris population 2.161M"
    候选 = "2.161 million"
    强度 = 0.95  ← 模型置信度

槽 1 搜索：→ 发现"Paris GDP €2.9T"
    候选 = "€2.9 trillion"
    强度 = 0.88


PHASE 3: 合并结果
──────────────────
State(depth=1) = [
    Variable(id=0, ..., candidate="2.161 million", candidate_strength=0.95),
    Variable(id=1, ..., candidate="€2.9 trillion", candidate_strength=0.88),
]
brief: "d1(++)"  ← 都已填


PHASE 4: SCA 评审
──────────────────
[收集所有 60 个块] → LLM 评审
  ├─ "Paris 2019 population was 2.161 million"  ✓
  ├─ "GDP reached €2.9 trillion in 2019"       ✓
  └─ SCA 判决：SUFFICIENT
  

PHASE 5: 合成答案
──────────────────
输出："Paris had 2.161 million inhabitants and GDP of €2.9 trillion in 2019."
```

### 不生产轮检测（行 1185-1189）

```python
# 为什么需要这个？
# session 1 完成，得到 view_A
# session 2 完成，得到 view_B
# 如果 view_A == view_B（相同证据），SCA 的判决会完全相同
# → 再搜一轮也是徒劳

prev_id = state.get("sca_view_id")
view_id = _select_sca_view(...)[1]

if prev_id and view_id == prev_id:
    _LOG.info("[SCA] view UNCHANGED; closing out (loop futility)")
    return {"verdict": {"status": "INSUFFICIENT"}, "no_progress": True}
```

---

## 3️⃣ **fanout = 分解 / 扇出（规划阶段）**

### 定义
**规划器将一个复杂问题分解为多个独立的子问题（槽位），然后并发搜索**。

### 代码位置
- `agentic_rag_graph.py` 行 376-420（planner 节点）

### 工作流程

```
问题："Paris and Tokyo: 2019 population?"
                    ↓
         [Planner Agent]
                    ↓
         (fanout 分解)
                    ↓
        ┌─────────────────────────┐
        │ Sub-queries (fanouts)   │
        ├─────────────────────────┤
        │ 1. "Paris population 2019"  │
        │ 2. "Tokyo population 2019"  │
        └─────────────────────────┘
                    ↓
         [Prefetch（可选）]
      并行搜索这两个查询，
      预填充共享证据池
                    ↓
         [RAG Agent ×2（并发）]
      各槽独立研究
                    ↓
    槽 0: "2.161M"  + 槽 1: "37.4M"
                    ↓
         [SCA 评审]
      "Paris 2.161M, Tokyo 37.4M" ✓
                    ↓
           [Answer]
```

### 配置控制

```python
# config.py 行 71-107

"low":      use_fanout=False  # ❌ 直接搜原问题，无分解
"medium":   use_fanout=False  # ❌ 无分解，但有工具循环
"high":     use_fanout=True   # ✅ 有规划 + 并发搜索
"ultra":    use_fanout=True   # ✅ 规划 + 更深工具循环
```

---

## 4️⃣ **其他关键概念速查表**

| 术语 | 英文 | 定义 | 代码位置 |
|------|------|------|---------|
| **Action Session** | - | 单向研究（追求一个槽位）的小 LLM 循环 | action_session.py#L2087 |
| **ToolOutcome** | - | 一次工具调用的结果封装（状态+payload） | action_session.py#L175 |
| **状态补丁** | State Patch | LLM 输出的槽位更新（候选值+强度） | action_session.py#L555 |
| **证据池** | Evidence Pool | 共享的块缓冲（最多 120 个） | action_session.py#L62 |
| **SCA 视图** | SCA View | 从池中排序出的前 60 个块 | agentic_rag_graph.py#L121 |
| **导航梯** | Navigation Ladder | 代码驱动的规则链（navigate_tree → drill → global） | action_session.py#L1861 |
| **原始问题恢复** | - | 保留用户原问题避免多跳丢失 | agentic_rag.py#L155 |
| **跨调用缓存** | - | 同一问题的多次调用复用答案 | agentic_rag.py#L837 |
| **近重复检测** | Near-dup Detection | 跳过改述但意图相同的查询 | action_session.py#L73 |

---

## 5️⃣ **工作流快速图解**

```
用户问题
   │
   ├─ [low 模式]
   │  └─ 直接搜索 → 答案 (1 轮)
   │
   ├─ [medium 模式]
   │  └─ 工具循环 → SCA (信息用) → 答案
   │
   ├─ [high 模式]
   │  ├─ 规划 (fanout 分解)
   │  ├─ 预取 (并行搜索)
   │  ├─ 工具循环 (per-slot)
   │  ├─ SCA 决策 → 充分 / 不充分
   │  │       └─ 不充分 → query_rewrite → 新搜 (≤3 轮)
   │  └─ 答案
   │
   └─ [ultra 模式]
      └─ 同 high，但 action_max_turns=6, sca_max_rounds=5
```

---

## 💡 **理解的关键要点**

1. **槽位不是"prompt template"**：它是真实的数据结构，持有候选值、强度、发现的线索。
2. **SCA 不一定停止搜索**：在 medium 模式下，SCA 的结果仅用于日志；只有在 high/ultra 时才驱动重写循环。
3. **fanout 是自动的**：规划器由 LLM 决定分解的数量和角度，不是硬编码的。
4. **强度最高原则**：并发的 session 结果合并时，谁的置信度高就用谁的，确保确定性。
5. **证据池独立于 SCA 视图**：池最多 120 个块，SCA 看前 60 个排序后的块。解耦让存储和评审各司其职。

---

