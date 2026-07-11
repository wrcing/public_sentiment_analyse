# 舆情分析平台详细设计说明书 — 分析引擎层

---

## 1. 引言

### 1.1 设计范围

本文档是分析引擎层的详细设计说明书，依据 HLD 第 3.2.4 节（LangGraph 分析引擎）和 SRS 第 3.3-3.5 节（主题检索、报告生成、图谱分析）编写。覆盖：

- 三个 LangGraph 工作流（主题检索/报告生成/图谱构建）的图定义与节点实现
- 线程池调度器（任务提交、状态跟踪、回调通知）
- LLM 调用封装与 LangFuse 集成
- Prompt 模板设计
- 分析结果、报告、图谱数据的存储

### 1.2 与 HLD 的对应关系

| HLD 章节 | 本 LLD 覆盖 |
|----------|------------|
| 3.2.4 LangGraph 分析引擎 | 三个 Graph 定义与节点实现 |
| 4.2.1 PostgreSQL — 分析结果/报告/图谱 | 五个表的 DDL 与 ORM |
| 5.3 API 后端 ↔ LLM | LLM 调用封装 |
| 5.4 API 后端 ↔ LangFuse | LangFuse callback 集成 |

---

## 2. 数据库表设计

本模块独占以下五张表。

### 2.1 分析结果表

```sql
CREATE TABLE analysis_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL,                             -- 对应检索任务 ID（业务 ID，非 FK）
    topic           VARCHAR(500) NOT NULL,                     -- 用户输入的主题词
    source_filters  JSONB NOT NULL DEFAULT '{}',              -- 来源过滤条件
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',   -- pending | running | completed | failed
    summary         TEXT,                                      -- 来源摘要（LLM 生成）
    viewpoints      JSONB,                                     -- 观点提炼 [{"viewpoint":"...","count":N,"sources":[...]}]
    sentiment       JSONB,                                     -- 情绪判断 {"positive":0.3,"neutral":0.5,"negative":0.2}
    raw_output      JSONB,                                     -- LLM 原始输出
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_analysis_results_task_id ON analysis_results(task_id);
CREATE INDEX idx_analysis_results_topic ON analysis_results(topic);
CREATE INDEX idx_analysis_results_created_at ON analysis_results(created_at DESC);
```

### 2.2 报告表

```sql
CREATE TABLE reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       VARCHAR(50) NOT NULL UNIQUE,              -- 对外报告编号，格式: RPT-YYYYMMDD-XXXXXX
    title           VARCHAR(500) NOT NULL,
    analysis_result_id UUID NOT NULL REFERENCES analysis_results(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',   -- pending | running | completed | failed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_reports_report_id ON reports(report_id);
CREATE INDEX idx_reports_created_at ON reports(created_at DESC);
```

### 2.3 报告章节表

```sql
CREATE TABLE report_chapters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES reports(id) ON DELETE CASCADE,
    chapter_order   INTEGER NOT NULL,                         -- 1-8
    chapter_title   VARCHAR(100) NOT NULL,                    -- 章节标题
    content         TEXT NOT NULL,                            -- 章节正文（Markdown）
    evidence_refs   JSONB,                                    -- 证据引用 [{"source_id":"...","quote":"..."}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_report_chapters_report_id ON report_chapters(report_id);
```

**8 个固定章节：**

| 序号 | 章节标题 | 说明 |
|------|---------|------|
| 1 | 事件概述 | 舆情事件的概括性描述 |
| 2 | 时间线 | 关键事件节点的时间顺序梳理 |
| 3 | 传播渠道 | 各平台传播情况分析 |
| 4 | 主要观点 | 各方核心观点汇总 |
| 5 | 情绪倾向 | 公众情绪分布与变化 |
| 6 | 风险判断 | 潜在风险评估 |
| 7 | 重点证据 | 关键来源内容引用 |
| 8 | 结论摘要 | 整体结论与建议 |

### 2.4 图谱节点表

```sql
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES reports(id) ON DELETE CASCADE,
    node_label      VARCHAR(200) NOT NULL,                    -- 节点显示名称
    node_type       VARCHAR(50) NOT NULL,                     -- 实体类型（首版 7 类）
    properties      JSONB DEFAULT '{}',                       -- 附加属性
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_report_id ON graph_nodes(report_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
```

**首版 P1 实体类型（7 类）：** `person`（人物）、`organization`（组织机构）、`event`（事件）、`topic`（话题）、`source`（信息来源）、`platform`（平台）、`stance`（观点立场）

### 2.5 图谱边表

```sql
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id       UUID NOT NULL REFERENCES reports(id) ON DELETE CASCADE,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    relation_type   VARCHAR(50) NOT NULL,                     -- 关系类型（首版 7 种）
    properties      JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_report_id ON graph_edges(report_id);
CREATE INDEX idx_graph_edges_source_node ON graph_edges(source_node_id);
CREATE INDEX idx_graph_edges_target_node ON graph_edges(target_node_id);
```

**首版 P1 关系类型（7 种）：** `publish`（发布）、`source_from`（来源平台）、`involve_event`（涉及事件）、`belong_to_topic`（属于话题）、`express_viewpoint`（表达观点）、`hold_stance`（持有立场）、`relate_to`（关联）

---

## 3. 线程池调度器

### 3.1 调度器设计

```python
# backend/app/engine/scheduler.py

import asyncio
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass, field
from typing import Callable
import logging
import uuid

logger = logging.getLogger("engine")

@dataclass
class EngineTask:
    """分析任务描述。"""
    task_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    task_type: str = ""           # "search" | "report" | "graph"
    params: dict = field(default_factory=dict)
    status: str = "pending"       # pending | running | completed | failed
    error_message: str | None = None
    on_complete: Callable | None = None   # 完成回调（用于 SSE 推送）

class EngineScheduler:
    """
    分析引擎调度器。

    - 管理一个 ThreadPoolExecutor
    - 接收分析任务 → 提交到线程池 → 通过回调通知状态
    - 每个任务在独立线程中运行 LangGraph 工作流
    """

    def __init__(self, pool_size: int = 4):
        self._executor = ThreadPoolExecutor(max_workers=pool_size)
        self._tasks: dict[str, EngineTask] = {}

    async def submit(self, task_type: str, params: dict, on_complete: Callable | None = None) -> EngineTask:
        """
        提交分析任务。

        参数:
            task_type:   "search" | "report" | "graph"
            params:      工作流参数（主题词、报告编号等）
            on_complete: 完成时回调 (task) -> None

        返回:
            EngineTask，包含 task_id 供后续查询状态
        """
        task = EngineTask(task_type=task_type, params=params, on_complete=on_complete)
        self._tasks[task.task_id] = task

        # 在线程池中异步执行
        loop = asyncio.get_event_loop()
        loop.run_in_executor(self._executor, self._run_task, task)

        return task

    def _run_task(self, task: EngineTask):
        """在线程中执行工作流。"""
        task.status = "running"
        try:
            if task.task_type == "search":
                result = run_search_workflow(task.params)
            elif task.task_type == "report":
                result = run_report_workflow(task.params)
            elif task.task_type == "graph":
                result = run_graph_workflow(task.params)
            else:
                raise ValueError(f"未知任务类型: {task.task_type}")

            task.status = "completed"
            task.params["_result"] = result

        except Exception as e:
            logger.error("分析任务 %s 失败: %s", task.task_id, e, exc_info=True)
            task.status = "failed"
            task.error_message = str(e)

        # 回调通知（如 SSE 推送）
        if task.on_complete:
            try:
                task.on_complete(task)
            except Exception as cb_err:
                logger.error("任务完成回调异常: %s", cb_err)

    def get_task(self, task_id: str) -> EngineTask | None:
        return self._tasks.get(task_id)

    def shutdown(self):
        self._executor.shutdown(wait=True, cancel_futures=False)


# 全局单例
scheduler = EngineScheduler(pool_size=4)
```

---

## 4. LangGraph 工作流 — 主题检索

### 4.1 图结构

```
┌──────────┐
│  START   │
└────┬─────┘
     ▼
┌──────────────┐
│ load_data     │  从 collection_data 表按主题词过滤加载数据
└────┬─────────┘
     ▼
┌──────────────┐
│ filter_by_    │  按来源平台和内容类型进一步过滤
│ source        │
└────┬─────────┘
     ▼
┌──────────────┐
│ generate_     │  LLM: 生成来源摘要（每条数据做摘要提炼）
│ summary       │
└────┬─────────┘
     ▼
┌──────────────┐
│ extract_      │  LLM: 提炼核心观点及对应来源引用
│ viewpoints    │
└────┬─────────┘
     ▼
┌──────────────┐
│ analyze_      │  LLM: 判断情绪倾向
│ sentiment     │
└────┬─────────┘
     ▼
┌──────────────┐
│ save_result   │  写入 analysis_results 表
└────┬─────────┘
     ▼
┌──────────┐
│   END    │
└──────────┘
```

### 4.2 节点实现

```python
# backend/app/engine/search_graph.py

from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
from operator import add
from app.engine.llm import get_llm
from app.engine.prompts import (
    SUMMARY_PROMPT,
    VIEWPOINT_PROMPT,
    SENTIMENT_PROMPT,
)

class SearchState(TypedDict):
    topic: str
    source_platforms: list[str]
    collection_data: list[dict]     # 从 DB 加载的采集数据
    filtered_data: list[dict]       # 过滤后的数据
    summaries: Annotated[list, add]
    viewpoints: list[dict]
    sentiment: dict
    status: str
    error: str

# ─── 节点函数 ───

def load_data_node(state: SearchState) -> SearchState:
    """
    从 PostgreSQL 加载采集数据。

    查询条件:
    - collection_data.content ILIKE '%{topic}%'  (或使用 pg_trgm)
    - 如果 source_platforms 非空，加 platform IN (...) 过滤
    - 按 fetched_at DESC 取最近 500 条
    """
    from app.core.database import async_session
    from sqlalchemy import text

    # 在线程中运行 asyncio
    import asyncio
    async def _load():
        async with async_session() as session:
            query = """
                SELECT id, task_id, platform, source_url, title, content,
                       author_name, publish_time, raw_json, fetched_at
                FROM collection_data
                WHERE content ILIKE :topic_pattern
            """
            params = {"topic_pattern": f"%{state['topic']}%"}

            if state.get("source_platforms"):
                platforms = state["source_platforms"]
                placeholders = ", ".join(f":p{i}" for i in range(len(platforms)))
                query += f" AND platform IN ({placeholders})"
                for i, p in enumerate(platforms):
                    params[f"p{i}"] = p

            query += " ORDER BY fetched_at DESC LIMIT 500"

            result = await session.execute(text(query), params)
            rows = result.fetchall()
            return [dict(row._mapping) for row in rows]

    loop = asyncio.new_event_loop()
    try:
        data = loop.run_until_complete(_load())
    finally:
        loop.close()

    state["collection_data"] = data
    return state


def filter_node(state: SearchState) -> SearchState:
    """按来源平台过滤（在 SQL 已做，此处为兜底过滤和去重）。"""
    seen = set()
    filtered = []
    for item in state["collection_data"]:
        content_key = (item.get("title") or "") + (item.get("content") or "")[:100]
        if content_key not in seen:
            seen.add(content_key)
            filtered.append(item)
    state["filtered_data"] = filtered
    return state


def generate_summary_node(state: SearchState) -> SearchState:
    """LLM 生成来源摘要——分批处理，每批 20 条。"""
    llm = get_llm()
    batch_size = 20
    items = state["filtered_data"]
    summaries = []

    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        texts = "\n\n---\n\n".join(
            f"[{j}] 平台:{item.get('platform')} 标题:{item.get('title')}\n内容:{item.get('content','')[:500]}"
            for j, item in enumerate(batch)
        )
        prompt = SUMMARY_PROMPT.format(topic=state["topic"], texts=texts)
        resp = llm.invoke(prompt)
        summaries.append({"batch_start": i, "batch_end": i + len(batch), "summary": resp.content})

    state["summaries"] = summaries
    return state


def extract_viewpoints_node(state: SearchState) -> SearchState:
    """LLM 提炼观点。"""
    llm = get_llm()
    all_summaries = "\n".join(s["summary"] for s in state["summaries"])
    prompt = VIEWPOINT_PROMPT.format(topic=state["topic"], summaries=all_summaries)
    resp = llm.invoke(prompt)

    import json
    try:
        viewpoints = json.loads(resp.content)
    except json.JSONDecodeError:
        viewpoints = [{"viewpoint": resp.content, "count": 1, "sources": []}]

    state["viewpoints"] = viewpoints
    return state


def analyze_sentiment_node(state: SearchState) -> SearchState:
    """LLM 情绪分析。"""
    llm = get_llm()
    all_summaries = "\n".join(s["summary"] for s in state["summaries"])
    prompt = SENTIMENT_PROMPT.format(topic=state["topic"], summaries=all_summaries)
    resp = llm.invoke(prompt)

    import json
    try:
        sentiment = json.loads(resp.content)
    except json.JSONDecodeError:
        sentiment = {"positive": 0.33, "neutral": 0.34, "negative": 0.33}

    state["sentiment"] = sentiment
    return state


def save_result_node(state: SearchState) -> SearchState:
    """将分析结果写入 analysis_results 表。"""
    import asyncio
    from app.core.database import async_session
    from sqlalchemy import text
    import uuid
    import json

    async def _save():
        async with async_session() as session:
            task_id = str(uuid.uuid4())
            await session.execute(
                text("""
                    INSERT INTO analysis_results
                        (id, task_id, topic, source_filters, status, summary, viewpoints, sentiment, raw_output)
                    VALUES
                        (:id, :task_id, :topic, :filters, 'completed', :summary, :viewpoints, :sentiment, :raw)
                """),
                {
                    "id": str(uuid.uuid4()),
                    "task_id": task_id,
                    "topic": state["topic"],
                    "filters": json.dumps(state.get("source_platforms", [])),
                    "summary": state["summaries"][-1]["summary"] if state["summaries"] else "",
                    "viewpoints": json.dumps(state["viewpoints"], ensure_ascii=False),
                    "sentiment": json.dumps(state["sentiment"]),
                    "raw": json.dumps(state["summaries"], ensure_ascii=False),
                },
            )
            await session.commit()
            return task_id

    loop = asyncio.new_event_loop()
    try:
        task_id = loop.run_until_complete(_save())
    finally:
        loop.close()

    state["status"] = "completed"
    return state


# ─── 构建图 ───

def build_search_graph() -> StateGraph:
    graph = StateGraph(SearchState)

    graph.add_node("load_data", load_data_node)
    graph.add_node("filter", filter_node)
    graph.add_node("generate_summary", generate_summary_node)
    graph.add_node("extract_viewpoints", extract_viewpoints_node)
    graph.add_node("analyze_sentiment", analyze_sentiment_node)
    graph.add_node("save_result", save_result_node)

    graph.set_entry_point("load_data")
    graph.add_edge("load_data", "filter")
    graph.add_edge("filter", "generate_summary")
    graph.add_edge("generate_summary", "extract_viewpoints")
    graph.add_edge("extract_viewpoints", "analyze_sentiment")
    graph.add_edge("analyze_sentiment", "save_result")
    graph.add_edge("save_result", END)

    return graph.compile()


def run_search_workflow(params: dict) -> dict:
    """入口函数——被调度器在线程池中调用。"""
    graph = build_search_graph()
    initial_state = SearchState(
        topic=params["topic"],
        source_platforms=params.get("source_platforms", []),
        collection_data=[],
        filtered_data=[],
        summaries=[],
        viewpoints=[],
        sentiment={},
        status="pending",
        error="",
    )
    result = graph.invoke(initial_state)
    return result
```

---

## 5. LangGraph 工作流 — 报告生成

### 5.1 图结构

```
START → load_analysis → generate_chapter_1..8 (并行) → assemble → save_report → END
```

8 个章节节点并行执行（通过 LangGraph 的 `Send` 机制或顺序执行——首版采用顺序执行，简单可控）。

### 5.2 节点实现（关键代码）

```python
# backend/app/engine/report_graph.py

from langgraph.graph import StateGraph, END
from typing import TypedDict

class ReportState(TypedDict):
    analysis_result_id: str
    topic: str
    analysis_summary: str
    analysis_viewpoints: list
    analysis_sentiment: dict
    chapters: list[dict]       # [{"order":1,"title":"事件概述","content":"..."}, ...]
    report_id_str: str         # RPT-YYYYMMDD-XXXXXX
    status: str
    error: str

CHAPTERS = [
    (1, "事件概述"),
    (2, "时间线"),
    (3, "传播渠道"),
    (4, "主要观点"),
    (5, "情绪倾向"),
    (6, "风险判断"),
    (7, "重点证据"),
    (8, "结论摘要"),
]

def load_analysis_node(state: ReportState) -> ReportState:
    """加载关联的 analysis_result。"""
    import asyncio
    from app.core.database import async_session
    from sqlalchemy import text

    async def _load():
        async with async_session() as session:
            result = await session.execute(
                text("SELECT * FROM analysis_results WHERE id = :id"),
                {"id": state["analysis_result_id"]},
            )
            row = result.fetchone()
            return dict(row._mapping) if row else {}

    loop = asyncio.new_event_loop()
    try:
        data = loop.run_until_complete(_load())
    finally:
        loop.close()

    state["analysis_summary"] = data.get("summary", "")
    state["analysis_viewpoints"] = data.get("viewpoints", [])
    state["analysis_sentiment"] = data.get("sentiment", {})
    return state


def generate_chapter_node_factory(chapter_order: int, chapter_title: str):
    """工厂函数——为每个章节创建生成节点。"""
    from app.engine.llm import get_llm
    from app.engine.prompts import CHAPTER_PROMPTS  # 各章节的 prompt 模板

    def node(state: ReportState) -> ReportState:
        llm = get_llm()
        prompt_template = CHAPTER_PROMPTS.get(chapter_order, "请根据以下信息生成'{title}'章节内容。")

        prompt = prompt_template.format(
            topic=state.get("topic", ""),
            summary=state.get("analysis_summary", ""),
            viewpoints=state.get("analysis_viewpoints", ""),
            sentiment=state.get("analysis_sentiment", ""),
            title=chapter_title,
        )
        resp = llm.invoke(prompt)

        state["chapters"].append({
            "order": chapter_order,
            "title": chapter_title,
            "content": resp.content,
        })
        return state

    return node


def assemble_node(state: ReportState) -> ReportState:
    """汇总 8 章节为 Markdown 报告。"""
    state["chapters"].sort(key=lambda c: c["order"])
    return state


def save_report_node(state: ReportState) -> ReportState:
    """写入 reports + report_chapters 表。"""
    import asyncio
    from app.core.database import async_session
    from sqlalchemy import text
    import uuid
    from datetime import datetime

    report_id = f"RPT-{datetime.now().strftime('%Y%m%d')}-{uuid.uuid4().hex[:6].upper()}"

    async def _save():
        async with async_session() as session:
            report_uuid = str(uuid.uuid4())
            await session.execute(
                text("""
                    INSERT INTO reports (id, report_id, title, analysis_result_id, status)
                    VALUES (:id, :report_id, :title, :arid, 'completed')
                """),
                {
                    "id": report_uuid,
                    "report_id": report_id,
                    "title": f"舆情分析报告 - {state['topic']}",
                    "arid": state["analysis_result_id"],
                },
            )
            for ch in state["chapters"]:
                await session.execute(
                    text("""
                        INSERT INTO report_chapters (id, report_id, chapter_order, chapter_title, content)
                        VALUES (:id, :report_id, :order, :title, :content)
                    """),
                    {
                        "id": str(uuid.uuid4()),
                        "report_id": report_uuid,
                        "order": ch["order"],
                        "title": ch["title"],
                        "content": ch["content"],
                    },
                )
            await session.commit()

    loop = asyncio.new_event_loop()
    try:
        loop.run_until_complete(_save())
    finally:
        loop.close()

    state["report_id_str"] = report_id
    state["status"] = "completed"
    return state


def build_report_graph() -> StateGraph:
    graph = StateGraph(ReportState)

    graph.add_node("load_analysis", load_analysis_node)

    for order, title in CHAPTERS:
        node_name = f"chapter_{order}"
        graph.add_node(node_name, generate_chapter_node_factory(order, title))

    graph.add_node("assemble", assemble_node)
    graph.add_node("save_report", save_report_node)

    graph.set_entry_point("load_analysis")

    # 顺序执行 8 个章节
    prev = "load_analysis"
    for order, _ in CHAPTERS:
        graph.add_edge(prev, f"chapter_{order}")
        prev = f"chapter_{order}"

    graph.add_edge("chapter_8", "assemble")
    graph.add_edge("assemble", "save_report")
    graph.add_edge("save_report", END)

    return graph.compile()


def run_report_workflow(params: dict) -> dict:
    graph = build_report_graph()
    state = ReportState(
        analysis_result_id=params["analysis_result_id"],
        topic=params.get("topic", ""),
        analysis_summary="",
        analysis_viewpoints=[],
        analysis_sentiment={},
        chapters=[],
        report_id_str="",
        status="pending",
        error="",
    )
    return graph.invoke(state)
```

---

## 6. LangGraph 工作流 — 图谱构建

### 6.1 图结构

```
START → load_report → extract_entities → extract_relations → deduplicate → save_graph → END
```

### 6.2 节点实现（关键代码）

```python
# backend/app/engine/graph_graph.py  (分析模块下的图谱构建)

from langgraph.graph import StateGraph, END
from typing import TypedDict
import json

class GraphState(TypedDict):
    report_id: str
    report_content: str            # 报告全文 Markdown
    entities: list[dict]           # [{"label":"...","type":"person",...}, ...]
    relations: list[dict]          # [{"source_label":"...","target_label":"...","type":"publish",...}, ...]
    status: str
    error: str

ENTITY_TYPES = ["person", "organization", "event", "topic", "source", "platform", "stance"]  # 7 类
RELATION_TYPES = ["publish", "source_from", "involve_event", "belong_to_topic",
                  "express_viewpoint", "hold_stance", "relate_to"]  # 7 种


def load_report_node(state: GraphState) -> GraphState:
    """加载报告全文。"""
    import asyncio
    from app.core.database import async_session
    from sqlalchemy import text

    async def _load():
        async with async_session() as session:
            result = await session.execute(
                text("""
                    SELECT rc.chapter_order, rc.content
                    FROM reports r
                    JOIN report_chapters rc ON rc.report_id = r.id
                    WHERE r.report_id = :rid
                    ORDER BY rc.chapter_order
                """),
                {"rid": state["report_id"]},
            )
            rows = result.fetchall()
            return "\n\n".join(f"## {r.chapter_order}\n{r.content}" for r in rows)

    loop = asyncio.new_event_loop()
    try:
        state["report_content"] = loop.run_until_complete(_load())
    finally:
        loop.close()

    return state


def extract_entities_node(state: GraphState) -> GraphState:
    """LLM 抽取实体。"""
    from app.engine.llm import get_llm
    from app.engine.prompts import ENTITY_EXTRACTION_PROMPT

    llm = get_llm()
    prompt = ENTITY_EXTRACTION_PROMPT.format(
        content=state["report_content"][:8000],  # 截断防止超 token
        entity_types=", ".join(ENTITY_TYPES),
    )
    resp = llm.invoke(prompt)

    try:
        entities = json.loads(resp.content)
    except json.JSONDecodeError:
        entities = []

    state["entities"] = entities
    return state


def extract_relations_node(state: GraphState) -> GraphState:
    """LLM 抽取关系。"""
    from app.engine.llm import get_llm
    from app.engine.prompts import RELATION_EXTRACTION_PROMPT

    llm = get_llm()
    prompt = RELATION_EXTRACTION_PROMPT.format(
        content=state["report_content"][:8000],
        entities=json.dumps(state["entities"], ensure_ascii=False),
        relation_types=", ".join(RELATION_TYPES),
    )
    resp = llm.invoke(prompt)

    try:
        relations = json.loads(resp.content)
    except json.JSONDecodeError:
        relations = []

    state["relations"] = relations
    return state


def deduplicate_node(state: GraphState) -> GraphState:
    """实体去重——按 label + type 合并。"""
    seen = {}
    deduped = []
    for e in state["entities"]:
        key = (e.get("label", ""), e.get("type", ""))
        if key not in seen:
            seen[key] = e
            deduped.append(e)
    state["entities"] = deduped
    return state


def save_graph_node(state: GraphState) -> GraphState:
    """写入 graph_nodes + graph_edges 表。"""
    import asyncio
    from app.core.database import async_session
    from sqlalchemy import text
    import uuid

    async def _save():
        async with async_session() as session:
            # 先查出 report UUID
            result = await session.execute(
                text("SELECT id FROM reports WHERE report_id = :rid"),
                {"rid": state["report_id"]},
            )
            report_uuid = result.scalar_one()

            # 写入节点
            node_id_map: dict[str, str] = {}  # label → node_uuid
            for e in state["entities"]:
                nid = str(uuid.uuid4())
                node_id_map[e["label"]] = nid
                await session.execute(
                    text("""
                        INSERT INTO graph_nodes (id, report_id, node_label, node_type, properties)
                        VALUES (:id, :report_id, :label, :type, :props)
                    """),
                    {
                        "id": nid,
                        "report_id": report_uuid,
                        "label": e["label"],
                        "type": e["type"],
                        "props": json.dumps(e.get("properties", {})),
                    },
                )

            # 写入边
            for r in state["relations"]:
                src_id = node_id_map.get(r.get("source_label", ""))
                tgt_id = node_id_map.get(r.get("target_label", ""))
                if src_id and tgt_id:
                    await session.execute(
                        text("""
                            INSERT INTO graph_edges (id, report_id, source_node_id, target_node_id, relation_type)
                            VALUES (:id, :report_id, :src, :tgt, :type)
                        """),
                        {
                            "id": str(uuid.uuid4()),
                            "report_id": report_uuid,
                            "src": src_id,
                            "tgt": tgt_id,
                            "type": r["type"],
                        },
                    )

            await session.commit()

    loop = asyncio.new_event_loop()
    try:
        loop.run_until_complete(_save())
    finally:
        loop.close()

    state["status"] = "completed"
    return state


def build_graph_workflow() -> StateGraph:
    graph = StateGraph(GraphState)

    graph.add_node("load_report", load_report_node)
    graph.add_node("extract_entities", extract_entities_node)
    graph.add_node("extract_relations", extract_relations_node)
    graph.add_node("deduplicate", deduplicate_node)
    graph.add_node("save_graph", save_graph_node)

    graph.set_entry_point("load_report")
    graph.add_edge("load_report", "extract_entities")
    graph.add_edge("extract_entities", "extract_relations")
    graph.add_edge("extract_relations", "deduplicate")
    graph.add_edge("deduplicate", "save_graph")
    graph.add_edge("save_graph", END)

    return graph.compile()


def run_graph_workflow(params: dict) -> dict:
    graph = build_graph_workflow()
    state = GraphState(
        report_id=params["report_id"],
        report_content="",
        entities=[],
        relations=[],
        status="pending",
        error="",
    )
    return graph.invoke(state)
```

---

## 7. LLM 调用封装

```python
# backend/app/engine/llm.py

from langchain_openai import ChatOpenAI
from langfuse.callback import CallbackHandler
from app.core.config import settings

_llm_instance = None

def get_llm() -> ChatOpenAI:
    """获取全局 LLM 实例（单例）。如果 LangFuse 配置了就挂载 callback。"""
    global _llm_instance
    if _llm_instance is None:
        callbacks = []
        if settings.LANGFUSE_PUBLIC_KEY:
            langfuse_handler = CallbackHandler(
                public_key=settings.LANGFUSE_PUBLIC_KEY,
                secret_key=settings.LANGFUSE_SECRET_KEY.get_secret_value()
                    if settings.LANGFUSE_SECRET_KEY else "",
                host=settings.LANGFUSE_HOST,
            )
            callbacks.append(langfuse_handler)

        _llm_instance = ChatOpenAI(
            model=settings.LLM_MODEL_NAME,
            temperature=settings.LLM_TEMPERATURE,
            max_tokens=settings.LLM_MAX_TOKENS,
            openai_api_key=settings.LLM_API_KEY.get_secret_value()
                if settings.LLM_API_KEY else "sk-placeholder",
            openai_api_base=settings.LLM_API_BASE,
            callbacks=callbacks,
        )
    return _llm_instance
```

**降级策略：** LangFuse callback 初始化失败时不阻塞 LLM 调用——仅记录日志，callbacks 列表为空。

---

## 8. Prompt 模板设计

```python
# backend/app/engine/prompts.py

SUMMARY_PROMPT = """你是一个专业的舆情分析师。请基于以下采集数据，对主题"{topic}"生成来源摘要。

要求：
1. 按平台分类归纳
2. 提取每个平台的核心讨论方向
3. 标注信息热度（高/中/低）
4. 总字数控制在 500 字以内

采集数据：
{texts}

请输出结构化的来源摘要（Markdown 格式）。"""

VIEWPOINT_PROMPT = """基于以下来源摘要，提炼"{topic}"的核心观点。

要求：
1. 每个观点包含：观点表述、出现频次、代表来源
2. 区分不同立场（支持/反对/中立）
3. 输出 JSON 数组格式:
[{{"viewpoint":"...", "count": N, "stance":"support|oppose|neutral", "sources":["..."]}}]

来源摘要：
{summaries}"""

SENTIMENT_PROMPT = """基于以下来源摘要，分析"{topic}"的公众情绪倾向。

要求：
1. 输出 JSON 格式，包含正面/中性/负面比例（小数，合计为 1）
2. 各情绪方向的关键词列表
3. 整体情绪走势判断（上升/下降/平稳）

格式: {{"positive":0.x, "neutral":0.x, "negative":0.x, "keywords":{{"positive":[],"neutral":[],"negative":[]}}, "trend":"stable|rising|falling"}}

来源摘要：
{summaries}"""

CHAPTER_PROMPTS = {
    1: """请根据以下信息生成舆情报告"事件概述"章节。
主题: {topic}
分析摘要: {summary}

要求: 用 300 字左右概括整个舆情事件的核心情况，包含事件背景、涉及主体、影响范围。""",

    2: """请根据以下信息生成舆情报告"时间线"章节。
主题: {topic}
分析摘要: {summary}

要求: 按时间顺序列出关键事件节点，每个节点标注时间、事件描述和影响程度。""",

    3: """请根据以下信息生成舆情报告"传播渠道"章节。
主题: {topic}
分析摘要: {summary}

要求: 分析各平台的传播情况，包括传播量级、关键传播节点、传播路径特征。""",

    4: """请根据以下信息生成舆情报告"主要观点"章节。
主题: {topic}
观点提炼: {viewpoints}

要求: 分类整理各方核心观点，标注每个观点的支持者群体和影响力。""",

    5: """请根据以下信息生成舆情报告"情绪倾向"章节。
主题: {topic}
情绪分析: {sentiment}

要求: 分析公众情绪分布，说明情绪变化趋势及可能的触发因素。""",

    6: """请根据以下信息生成舆情报告"风险判断"章节。
主题: {topic}
分析摘要: {summary}
情绪分析: {sentiment}

要求: 评估潜在的舆情风险，按风险等级（高/中/低）分类说明，给出预警建议。""",

    7: """请根据以下信息生成舆情报告"重点证据"章节。
主题: {topic}
分析摘要: {summary}

要求: 引用关键来源内容作为证据支撑，每条证据标注来源类型和可信度评估。""",

    8: """请根据以下信息生成舆情报告"结论摘要"章节。
主题: {topic}
全部章节内容: {summary}
观点: {viewpoints}
情绪: {sentiment}

要求: 提炼核心结论（3-5 条），给出后续关注方向的建议。""",
}

ENTITY_EXTRACTION_PROMPT = """你是一个信息抽取专家。请从以下舆情报告中抽取实体。

支持的实体类型: {entity_types}

报告内容:
{content}

请输出 JSON 数组，每个实体包含:
- label: 实体名称
- type: 实体类型（必须从支持类型中选择）
- properties: 附加属性字典

只输出 JSON 数组，不要包含其他文本。"""

RELATION_EXTRACTION_PROMPT = """你是一个信息抽取专家。请基于以下实体列表和报告内容，抽取实体间的关系。

支持的关系类型: {relation_types}
已知实体: {entities}

报告内容:
{content}

请输出 JSON 数组，每个关系包含:
- source_label: 源实体名称（必须在已知实体列表中）
- target_label: 目标实体名称（必须在已知实体列表中）
- type: 关系类型（必须从支持类型中选择）

只输出 JSON 数组，不要包含其他文本。"""
```

---

## 9. 异常处理与降级策略

| 场景 | 处理策略 |
|------|---------|
| LLM API 超时 | 重试 1 次（间隔 5s），仍失败则标记任务 `failed` |
| LLM 返回非 JSON（观点/情绪/实体提取） | 使用正则兜底提取，或返回空数组/默认值 |
| 采集数据为空 | 标记任务 `completed`，结果字段为空，前端展示"暂无相关数据" |
| LangFuse 不可用 | 忽略 callback 初始化异常，LLM 调用不受影响 |
| workfow 内节点异常 | 异常传播到调度器的 `_run_task`，标记任务 `failed` |
| 报告章节生成部分失败 | 单章失败不阻塞其他章节，失败章节标记 `[生成失败]` |

---

## 10. 文件清单

| 文件 | 职责 |
|------|------|
| `backend/app/engine/__init__.py` | 包初始化 |
| `backend/app/engine/scheduler.py` | EngineScheduler 调度器 |
| `backend/app/engine/llm.py` | LLM 实例获取 + LangFuse callback |
| `backend/app/engine/prompts.py` | 所有 Prompt 模板 |
| `backend/app/engine/search_graph.py` | 主题检索 LangGraph 工作流 |
| `backend/app/engine/report_graph.py` | 报告生成 LangGraph 工作流 |
| `backend/app/engine/graph_graph.py` | 图谱构建 LangGraph 工作流 |
