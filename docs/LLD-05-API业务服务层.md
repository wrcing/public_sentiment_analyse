# 舆情分析平台详细设计说明书 — API 业务服务层

---

## 1. 引言

### 1.1 设计范围

本文档是 API 后端业务服务层的详细设计说明书，依据 HLD 第 3.2.2 节（API 后端模块）、第 4.3 节（SSE 实时推送）和第 5.1 节（REST + SSE 接口清单）编写。覆盖：

- 路由注册与中间件链
- 控制台聚合服务
- 采集任务管理 API
- 检索/报告/图谱 API
- 系统状态与配置管理 API
- SSE 实时推送（事件总线 + 广播机制）
- Token 消耗统计 API

### 1.2 与 HLD 的对应关系

| HLD 章节 | 本 LLD 覆盖 |
|----------|------------|
| 3.2.2 API 后端模块 — 各子模块 | 全部 8 个子模块的详细设计 |
| 4.3 SSE 实时推送数据路径 | 事件总线、消息类型与广播 |
| 5.1 前端 ↔ API 后端接口清单 | 所有 REST 端点的请求/响应格式 |

---

## 2. 数据库表设计

本模块独占以下两张表。

### 2.1 系统配置表

```sql
CREATE TABLE system_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_key      VARCHAR(100) NOT NULL UNIQUE,
    config_value    TEXT NOT NULL,                             -- JSON 编码的值
    description     VARCHAR(255),
    updated_by      VARCHAR(100),                              -- 修改人用户名
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_system_configs_key ON system_configs(config_key);
```

**预置配置项：**

| config_key | 默认值 | 说明 |
|-----------|-------|------|
| `llm.api_base` | `"https://api.openai.com/v1"` | LLM API 地址 |
| `llm.model_name` | `"gpt-4o"` | 模型名称 |
| `llm.temperature` | `0.3` | 生成温度 |
| `llm.max_tokens` | `4096` | 单次调用最大 Token |
| `collector.poll_interval` | `5` | 采集轮询间隔（秒） |
| `collector.platforms` | `["bilibili","weibo","xiaohongshu","douyin","zhihu","toutiao"]` | 启用的采集平台 |
| `token.warning_threshold` | `80000` | Token 预警阈值 |
| `token.hard_limit` | `100000` | Token 硬上限 |

### 2.2 Token 消耗记录表

```sql
CREATE TABLE token_usage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module          VARCHAR(50) NOT NULL,                      -- search | report | graph
    input_tokens    INTEGER NOT NULL DEFAULT 0,
    output_tokens   INTEGER NOT NULL DEFAULT 0,
    model_name      VARCHAR(100),
    cost_usd        NUMERIC(10, 6) DEFAULT 0,                 -- 估算费用（美元）
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_token_usage_module ON token_usage(module);
CREATE INDEX idx_token_usage_created_at ON token_usage(created_at DESC);
```

### 2.3 ORM 模型

```python
# backend/app/models/config.py

class SystemConfig(Base):
    __tablename__ = "system_configs"
    id          = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    config_key  = mapped_column(String(100), unique=True, nullable=False, index=True)
    config_value = mapped_column(Text, nullable=False)
    description = mapped_column(String(255), nullable=True)
    updated_by  = mapped_column(String(100), nullable=True)
    created_at  = mapped_column(server_default="now()")
    updated_at  = mapped_column(server_default="now()")

# backend/app/models/token_usage.py

class TokenUsage(Base):
    __tablename__ = "token_usage"
    id            = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    module        = mapped_column(String(50), nullable=False, index=True)
    input_tokens  = mapped_column(Integer, nullable=False, default=0)
    output_tokens = mapped_column(Integer, nullable=False, default=0)
    model_name    = mapped_column(String(100), nullable=True)
    cost_usd      = mapped_column(Numeric(10, 6), default=0)
    created_at    = mapped_column(server_default="now()")
```

---

## 3. 路由注册

```python
# backend/app/main.py (create_app 内的路由注册)

from app.api.auth import router as auth_router
from app.api.dashboard import router as dashboard_router
from app.api.collection import router as collection_router
from app.api.search import router as search_router
from app.api.reports import router as reports_router
from app.api.graph import router as graph_router
from app.api.status import router as status_router
from app.api.config import router as config_router
from app.api.token_usage import router as token_usage_router
from app.api.sse import router as sse_router

app.include_router(auth_router,          prefix="/api/auth",       tags=["认证"])
app.include_router(dashboard_router,      prefix="/api/dashboard",  tags=["控制台"])
app.include_router(collection_router,     prefix="/api/collection", tags=["采集管理"])
app.include_router(search_router,         prefix="/api/search",     tags=["主题检索"])
app.include_router(reports_router,        prefix="/api/reports",    tags=["报告"])
app.include_router(graph_router,          prefix="/api/graph",      tags=["图谱"])
app.include_router(status_router,         prefix="/api/status",     tags=["系统状态"])
app.include_router(config_router,         prefix="/api/config",     tags=["配置管理"])
app.include_router(token_usage_router,    prefix="/api/token-usage", tags=["Token管理"])
app.include_router(sse_router,            prefix="",                tags=["实时推送"])
```

---

## 4. 控制台聚合服务

### 4.1 聚合逻辑

```python
# backend/app/services/dashboard_service.py

async def get_app_status_summary(db: AsyncSession) -> dict:
    """
    聚合所有单功能应用的当前状态。
    
    返回:
        {
            "collector": {"status": "running", "task_count": 3, "last_output": "..."},
            "search":    {"status": "idle",    "task_count": 0, "last_output": "..."},
            "report":    {"status": "idle",    "task_count": 0, "last_output": "..."},
            "graph":     {"status": "idle",    "task_count": 0, "last_output": "..."},
        }
    """
    # 采集服务状态——查最近一个任务
    collector_task = await db.execute(
        select(CollectionTask).order_by(CollectionTask.created_at.desc()).limit(1)
    )
    last_collection = collector_task.scalar_one_or_none()

    # 检索状态——查最近一个分析结果
    search_task = await db.execute(
        select(AnalysisResult).order_by(AnalysisResult.created_at.desc()).limit(1)
    )
    last_search = search_task.scalar_one_or_none()

    # 报告状态——查最近一个报告
    report_task = await db.execute(
        select(Report).order_by(Report.created_at.desc()).limit(1)
    )
    last_report = report_task.scalar_one_or_none()

    # 图谱状态——查最近一个图谱
    graph_task = await db.execute(
        text("SELECT report_id, created_at FROM graph_nodes ORDER BY created_at DESC LIMIT 1")
    )
    last_graph = graph_task.fetchone()

    return {
        "collector": {
            "status": last_collection.status if last_collection else "idle",
            "task_count": 0,  # 可通过 count 查询优化
            "last_output": last_collection.name if last_collection else None,
        },
        "search": {
            "status": last_search.status if last_search else "idle",
            "task_count": 0,
            "last_output": last_search.topic if last_search else None,
        },
        "report": {
            "status": last_report.status if last_report else "idle",
            "task_count": 0,
            "last_output": last_report.title if last_report else None,
        },
        "graph": {
            "status": "completed" if last_graph else "idle",
            "task_count": 0,
            "last_output": last_graph.report_id if last_graph else None,
        },
    }
```

### 4.2 API 端点

```python
# backend/app/api/dashboard.py

router = APIRouter(dependencies=[Depends(require_role("operator", "admin"))])

@router.get("/status")
async def dashboard_status(db: AsyncSession = Depends(get_db)):
    """获取所有应用当前状态汇总。"""
    return await get_app_status_summary(db)

@router.get("/summary")
async def dashboard_summary(db: AsyncSession = Depends(get_db)):
    """获取各应用最近一次输出摘要。"""
    # 与 status 类似，额外返回最近输出的内容摘要（如报告标题、检索主题等）
    status = await get_app_status_summary(db)
    return {
        "collector_last": status["collector"]["last_output"],
        "search_last": status["search"]["last_output"],
        "report_last": status["report"]["last_output"],
        "graph_last": status["graph"]["last_output"],
    }
```

---

## 5. 采集任务管理 API

```python
# backend/app/api/collection.py

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession
from app.api.deps import get_db, require_role
from app.models.user import User
import uuid
import json
from datetime import datetime, timezone

router = APIRouter(dependencies=[Depends(require_role("operator", "admin"))])

# ─── 请求/响应模型 ───

class CreateCollectionTaskRequest(BaseModel):
    name: str = Field(..., max_length=200)
    source_platforms: list[str] = Field(..., min_length=1)  # ["bilibili","weibo",...]
    keywords: list[str] = Field(default_factory=list)

class CollectionTaskResponse(BaseModel):
    id: str
    name: str
    source_platforms: list
    keywords: list
    status: str
    progress: int
    error_message: str | None
    created_at: str
    started_at: str | None
    finished_at: str | None

class CollectionTaskListResponse(BaseModel):
    items: list[CollectionTaskResponse]
    total: int
    page: int
    page_size: int

# ─── 端点 ───

@router.post("/tasks", response_model=CollectionTaskResponse, status_code=201)
async def create_collection_task(
    body: CreateCollectionTaskRequest,
    db: AsyncSession = Depends(get_db),
):
    """创建采集任务——写入一条 status=pending 的记录到 collection_tasks 表。"""
    task = CollectionTask(
        id=uuid.uuid4(),
        name=body.name,
        source_platforms=json.dumps(body.source_platforms),
        keywords=json.dumps(body.keywords),
        status="pending",
        progress=0,
        created_at=datetime.now(timezone.utc),
    )
    db.add(task)
    await db.flush()
    await db.refresh(task)
    return _task_to_response(task)


@router.get("/tasks", response_model=CollectionTaskListResponse)
async def list_collection_tasks(
    status: str | None = Query(None),
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
):
    """查询采集任务列表（支持状态过滤、分页）。"""
    stmt = select(CollectionTask)
    if status:
        stmt = stmt.where(CollectionTask.status == status)

    count_stmt = select(func.count()).select_from(stmt.subquery())
    total = (await db.execute(count_stmt)).scalar()

    stmt = stmt.order_by(CollectionTask.created_at.desc()).offset((page - 1) * page_size).limit(page_size)
    rows = (await db.execute(stmt)).scalars().all()

    return CollectionTaskListResponse(
        items=[_task_to_response(t) for t in rows],
        total=total,
        page=page,
        page_size=page_size,
    )


@router.get("/tasks/{task_id}", response_model=CollectionTaskResponse)
async def get_collection_task(task_id: str, db: AsyncSession = Depends(get_db)):
    """查看单个任务详情与进度。"""
    task = await db.get(CollectionTask, task_id)
    if not task:
        raise HTTPException(status_code=404, detail="任务不存在")
    return _task_to_response(task)


@router.get("/tasks/{task_id}/logs")
async def get_collection_logs(
    task_id: str,
    page: int = Query(1, ge=1),
    page_size: int = Query(50, ge=1, le=200),
    db: AsyncSession = Depends(get_db),
):
    """查看任务运行日志。"""
    stmt = (
        select(CollectionLog)
        .where(CollectionLog.task_id == task_id)
        .order_by(CollectionLog.created_at.desc())
        .offset((page - 1) * page_size)
        .limit(page_size)
    )
    rows = (await db.execute(stmt)).scalars().all()
    return {
        "items": [{"id": str(r.id), "level": r.level, "message": r.message, "created_at": str(r.created_at)} for r in rows],
        "total": len(rows),  # 简化；实际应 count
        "page": page,
        "page_size": page_size,
    }


@router.get("/platforms")
async def get_platforms(db: AsyncSession = Depends(get_db)):
    """获取支持的采集平台列表及启用状态。"""
    stmt = select(SystemConfig).where(SystemConfig.config_key == "collector.platforms")
    result = (await db.execute(stmt)).scalar_one_or_none()
    all_platforms = ["bilibili", "weibo", "xiaohongshu", "douyin", "zhihu", "toutiao"]
    enabled = json.loads(result.config_value) if result else all_platforms
    return {
        "platforms": [{"name": p, "enabled": p in enabled} for p in all_platforms],
    }


# ─── 辅助函数 ───

def _task_to_response(task: CollectionTask) -> CollectionTaskResponse:
    platforms = task.source_platforms
    if isinstance(platforms, str):
        platforms = json.loads(platforms)
    kw = task.keywords
    if isinstance(kw, str):
        kw = json.loads(kw)
    return CollectionTaskResponse(
        id=str(task.id),
        name=task.name,
        source_platforms=platforms if isinstance(platforms, list) else [],
        keywords=kw if isinstance(kw, list) else [],
        status=task.status,
        progress=task.progress or 0,
        error_message=task.error_message,
        created_at=str(task.created_at),
        started_at=str(task.started_at) if task.started_at else None,
        finished_at=str(task.finished_at) if task.finished_at else None,
    )
```

---

## 6. 主题检索 API

```python
# backend/app/api/search.py

router = APIRouter(dependencies=[Depends(require_role("operator", "admin"))])

class SearchRequest(BaseModel):
    topic: str = Field(..., min_length=1, max_length=500)
    source_platforms: list[str] = Field(default_factory=list)

class SearchTaskResponse(BaseModel):
    task_id: str
    topic: str
    status: str
    created_at: str

@router.post("", response_model=SearchTaskResponse, status_code=201)
async def start_search(body: SearchRequest):
    """
    发起主题检索。

    1. 在 analysis_results 表创建一条 pending 记录
    2. 将任务提交到分析引擎调度器
    3. 返回 task_id 供前端轮询/SSE 跟踪
    """
    # 创建 pending 记录
    # 提交到 scheduler
    from app.engine.scheduler import scheduler

    async def on_complete(task):
        # SSE 推送
        from app.api.sse import sse_bus
        await sse_bus.publish("app_output", {
            "app_name": "search",
            "output_text": f"主题 '{body.topic}' 检索完成",
        })

    task = await scheduler.submit("search", {
        "topic": body.topic,
        "source_platforms": body.source_platforms,
    }, on_complete=on_complete)

    return SearchTaskResponse(task_id=task.task_id, topic=body.topic, status="pending", created_at=str(datetime.now(timezone.utc)))


@router.get("/{task_id}")
async def get_search_result(task_id: str, db: AsyncSession = Depends(get_db)):
    """查看检索任务状态与结果。"""
    stmt = select(AnalysisResult).where(AnalysisResult.task_id == task_id)
    result = (await db.execute(stmt)).scalar_one_or_none()
    if not result:
        raise HTTPException(status_code=404, detail="检索任务不存在")

    return {
        "task_id": str(result.task_id),
        "topic": result.topic,
        "status": result.status,
        "summary": result.summary,
        "viewpoints": result.viewpoints,
        "sentiment": result.sentiment,
        "error_message": result.error_message,
        "created_at": str(result.created_at),
        "completed_at": str(result.completed_at) if result.completed_at else None,
    }


@router.get("")
async def list_searches(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
):
    """查询历史检索列表（分页）。"""
    stmt = select(AnalysisResult).order_by(AnalysisResult.created_at.desc())
    count = (await db.execute(select(func.count()).select_from(stmt.subquery()))).scalar()
    rows = (await db.execute(stmt.offset((page - 1) * page_size).limit(page_size))).scalars().all()

    return {
        "items": [
            {
                "id": str(r.id), "task_id": str(r.task_id), "topic": r.topic,
                "status": r.status, "created_at": str(r.created_at),
            }
            for r in rows
        ],
        "total": count, "page": page, "page_size": page_size,
    }


@router.get("/sources/{result_id}")
async def get_sources(result_id: str, db: AsyncSession = Depends(get_db)):
    """查看某条结论的来源追溯——返回关联的采集数据。"""
    # 根据分析结果关联的 collection_data 查询
    result = await db.get(AnalysisResult, result_id)
    if not result:
        raise HTTPException(status_code=404, detail="结果不存在")

    # 按 topic 模糊匹配采集数据
    stmt = (
        select(CollectionData)
        .where(CollectionData.content.ilike(f"%{result.topic}%"))
        .order_by(CollectionData.fetched_at.desc())
        .limit(100)
    )
    rows = (await db.execute(stmt)).scalars().all()
    return {
        "items": [
            {
                "id": str(r.id), "platform": r.platform, "title": r.title,
                "content": r.content[:500], "author": r.author_name,
                "source_url": r.source_url, "fetched_at": str(r.fetched_at),
            }
            for r in rows
        ],
    }
```

---

## 7. 报告 API

```python
# backend/app/api/reports.py

router = APIRouter()

class GenerateReportRequest(BaseModel):
    analysis_result_id: str = Field(...)
    topic: str = Field(...)

@router.post("", status_code=201, dependencies=[Depends(require_role("operator", "admin"))])
async def generate_report(body: GenerateReportRequest):
    """基于检索结果生成报告——提交到分析引擎。"""
    from app.engine.scheduler import scheduler

    async def on_complete(task):
        from app.api.sse import sse_bus
        if task.status == "completed":
            rid = task.params.get("_result", {}).get("report_id_str", "")
            await sse_bus.publish("app_output", {
                "app_name": "report",
                "output_text": f"报告 {rid} 生成完成",
            })

    task = await scheduler.submit("report", {
        "analysis_result_id": body.analysis_result_id,
        "topic": body.topic,
    }, on_complete=on_complete)

    return {"task_id": task.task_id, "status": "pending"}


@router.get("", dependencies=[Depends(require_role("viewer", "operator", "admin"))])
async def list_reports(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
):
    """查询报告列表（分页）。"""
    stmt = select(Report).order_by(Report.created_at.desc())
    count = (await db.execute(select(func.count()).select_from(stmt.subquery()))).scalar()
    rows = (await db.execute(stmt.offset((page - 1) * page_size).limit(page_size))).scalars().all()

    return {
        "items": [
            {"id": str(r.id), "report_id": r.report_id, "title": r.title, "status": r.status, "created_at": str(r.created_at)}
            for r in rows
        ],
        "total": count, "page": page, "page_size": page_size,
    }


@router.get("/{report_id}", dependencies=[Depends(require_role("viewer", "operator", "admin"))])
async def get_report(report_id: str, db: AsyncSession = Depends(get_db)):
    """按 report_id 获取完整报告内容。"""
    stmt = select(Report).where(Report.report_id == report_id)
    report = (await db.execute(stmt)).scalar_one_or_none()
    if not report:
        raise HTTPException(status_code=404, detail="报告不存在")

    chapters_stmt = (
        select(ReportChapter)
        .where(ReportChapter.report_id == report.id)
        .order_by(ReportChapter.chapter_order)
    )
    chapters = (await db.execute(chapters_stmt)).scalars().all()

    return {
        "report_id": report.report_id,
        "title": report.title,
        "status": report.status,
        "created_at": str(report.created_at),
        "chapters": [
            {"order": ch.chapter_order, "title": ch.chapter_title, "content": ch.content, "evidence_refs": ch.evidence_refs}
            for ch in chapters
        ],
    }


@router.get("/{report_id}/evidence", dependencies=[Depends(require_role("viewer", "operator", "admin"))])
async def get_report_evidence(report_id: str, db: AsyncSession = Depends(get_db)):
    """获取报告结论的来源证据追溯。"""
    # 查询报告章节中的 evidence_refs
    stmt = select(Report).where(Report.report_id == report_id)
    report = (await db.execute(stmt)).scalar_one_or_none()
    if not report:
        raise HTTPException(status_code=404, detail="报告不存在")

    chapters_stmt = (
        select(ReportChapter)
        .where(ReportChapter.report_id == report.id, ReportChapter.evidence_refs.isnot(None))
    )
    chapters = (await db.execute(chapters_stmt)).scalars().all()

    all_evidence = []
    for ch in chapters:
        if ch.evidence_refs:
            for ref in (ch.evidence_refs if isinstance(ch.evidence_refs, list) else []):
                all_evidence.append({**ref, "chapter": ch.chapter_title})

    return {"evidence": all_evidence}
```

---

## 8. 图谱 API

```python
# backend/app/api/graph.py

router = APIRouter(dependencies=[Depends(require_role("viewer", "operator", "admin"))])

@router.get("/{report_id}")
async def get_graph(report_id: str, db: AsyncSession = Depends(get_db)):
    """获取指定报告的图谱数据（节点+边）。"""
    # 查 report UUID
    stmt = select(Report).where(Report.report_id == report_id)
    report = (await db.execute(stmt)).scalar_one_or_none()
    if not report:
        raise HTTPException(status_code=404, detail="报告不存在")

    nodes_stmt = select(GraphNode).where(GraphNode.report_id == report.id)
    nodes = (await db.execute(nodes_stmt)).scalars().all()

    edges_stmt = select(GraphEdge).where(GraphEdge.report_id == report.id)
    edges = (await db.execute(edges_stmt)).scalars().all()

    return {
        "report_id": report_id,
        "nodes": [
            {"id": str(n.id), "label": n.node_label, "type": n.node_type, "properties": n.properties}
            for n in nodes
        ],
        "edges": [
            {"id": str(e.id), "source": str(e.source_node_id), "target": str(e.target_node_id), "relation": e.relation_type}
            for e in edges
        ],
    }


@router.post("/{report_id}/query")
async def query_graph(report_id: str, body: dict, db: AsyncSession = Depends(get_db)):
    """
    按节点/关系条件查询图谱子图。

    请求体:
        {"node_types": ["person","event"], "relation_types": ["involve_event"]}
    """
    stmt = select(Report).where(Report.report_id == report_id)
    report = (await db.execute(stmt)).scalar_one_or_none()
    if not report:
        raise HTTPException(status_code=404, detail="报告不存在")

    node_types = body.get("node_types", [])
    relation_types = body.get("relation_types", [])

    nodes_stmt = select(GraphNode).where(GraphNode.report_id == report.id)
    if node_types:
        nodes_stmt = nodes_stmt.where(GraphNode.node_type.in_(node_types))
    nodes = (await db.execute(nodes_stmt)).scalars().all()
    node_ids = {n.id for n in nodes}

    edges_stmt = select(GraphEdge).where(GraphEdge.report_id == report.id)
    if relation_types:
        edges_stmt = edges_stmt.where(GraphEdge.relation_type.in_(relation_types))
    edges = (await db.execute(edges_stmt)).scalars().all()

    # 仅返回与查询节点关联的边
    filtered_edges = [e for e in edges if e.source_node_id in node_ids or e.target_node_id in node_ids]

    return {
        "nodes": [{"id": str(n.id), "label": n.node_label, "type": n.node_type} for n in nodes],
        "edges": [
            {"id": str(e.id), "source": str(e.source_node_id), "target": str(e.target_node_id), "relation": e.relation_type}
            for e in filtered_edges
        ],
    }


@router.get("/latest")
async def get_latest_graph(db: AsyncSession = Depends(get_db)):
    """获取最新报告的图谱数据。"""
    stmt = select(Report).order_by(Report.created_at.desc()).limit(1)
    report = (await db.execute(stmt)).scalar_one_or_none()
    if not report:
        raise HTTPException(status_code=404, detail="暂无报告")

    # 委托给 get_graph
    return await get_graph(report.report_id, db)
```

---

## 9. 系统状态与配置 API

```python
# backend/app/api/status.py

router = APIRouter(dependencies=[Depends(require_role("admin"))])

@router.get("")
async def get_system_status(db: AsyncSession = Depends(get_db)):
    """获取系统级状态（整体状态、各应用、资源概况）。"""
    import psutil

    # 各应用状态
    collector_stmt = select(CollectionTask).order_by(CollectionTask.created_at.desc()).limit(1)
    collector = (await db.execute(collector_stmt)).scalar_one_or_none()

    return {
        "system_status": "online",
        "running_apps": [
            {"name": "collector", "status": collector.status if collector else "idle"},
            {"name": "search", "status": "idle"},
            {"name": "report", "status": "idle"},
            {"name": "graph", "status": "idle"},
        ],
        "resources": {
            "cpu_percent": psutil.cpu_percent(),
            "memory_percent": psutil.virtual_memory().percent,
            "disk_percent": psutil.disk_usage("/").percent,
        },
        "uptime": "...",  # 从进程启动时间获取
    }


@router.get("/errors")
async def get_errors(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
):
    """获取最近错误信息列表。"""
    stmt = (
        select(CollectionLog)
        .where(CollectionLog.level == "ERROR")
        .order_by(CollectionLog.created_at.desc())
        .offset((page - 1) * page_size)
        .limit(page_size)
    )
    rows = (await db.execute(stmt)).scalars().all()
    return {
        "items": [
            {"module_name": "collector", "error_message": r.message, "time": str(r.created_at)}
            for r in rows
        ],
        "total": len(rows), "page": page, "page_size": page_size,
    }


# backend/app/api/config.py

router = APIRouter(dependencies=[Depends(require_role("admin"))])

@router.get("")
async def get_configs(db: AsyncSession = Depends(get_db)):
    """获取当前系统配置（所有 config_key → config_value）。"""
    stmt = select(SystemConfig)
    rows = (await db.execute(stmt)).scalars().all()
    return {
        "configs": {r.config_key: _parse_config_value(r.config_value) for r in rows},
    }


@router.put("")
async def update_config(body: dict, db: AsyncSession = Depends(get_db), current_user: User = Depends(get_current_user)):
    """
    更新系统配置。

    请求体: {"config_key": "new_value", ...}

    仅允许更新已存在的 key；非法值拒绝保存。
    """
    for key, value in body.items():
        stmt = select(SystemConfig).where(SystemConfig.config_key == key)
        config = (await db.execute(stmt)).scalar_one_or_none()
        if not config:
            raise HTTPException(status_code=400, detail=f"无效配置项: {key}")
        config.config_value = json.dumps(value) if not isinstance(value, str) else value
        config.updated_by = current_user.username
        config.updated_at = datetime.now(timezone.utc)

    await db.commit()
    return {"message": "配置已更新"}


def _parse_config_value(raw: str):
    """尝试将 JSON 字符串解析为 Python 对象，失败则返回原字符串。"""
    try:
        return json.loads(raw)
    except (json.JSONDecodeError, TypeError):
        return raw
```

---

## 10. Token 消耗 API

```python
# backend/app/api/token_usage.py

router = APIRouter(dependencies=[Depends(require_role("admin"))])

@router.get("")
async def get_token_usage(
    module: str | None = Query(None),
    days: int = Query(7, ge=1, le=365),
    db: AsyncSession = Depends(get_db),
):
    """查询 Token 消耗统计（按模块/时间维度）。"""
    since = datetime.now(timezone.utc) - timedelta(days=days)

    stmt = select(TokenUsage).where(TokenUsage.created_at >= since)
    if module:
        stmt = stmt.where(TokenUsage.module == module)
    stmt = stmt.order_by(TokenUsage.created_at.desc())

    rows = (await db.execute(stmt)).scalars().all()

    # 按模块聚合
    by_module = {}
    for r in rows:
        if r.module not in by_module:
            by_module[r.module] = {"input_tokens": 0, "output_tokens": 0, "total_tokens": 0, "cost_usd": 0.0}
        by_module[r.module]["input_tokens"] += r.input_tokens
        by_module[r.module]["output_tokens"] += r.output_tokens
        by_module[r.module]["total_tokens"] += r.input_tokens + r.output_tokens
        by_module[r.module]["cost_usd"] += float(r.cost_usd or 0)

    return {
        "period_days": days,
        "by_module": by_module,
        "total": {
            "input_tokens": sum(m["input_tokens"] for m in by_module.values()),
            "output_tokens": sum(m["output_tokens"] for m in by_module.values()),
            "total_tokens": sum(m["total_tokens"] for m in by_module.values()),
            "cost_usd": sum(m["cost_usd"] for m in by_module.values()),
        },
    }


@router.get("/quota")
async def get_quota(db: AsyncSession = Depends(get_db)):
    """获取当前配额状态（预警/硬上限）。"""
    stmt_warning = select(SystemConfig).where(SystemConfig.config_key == "token.warning_threshold")
    stmt_hard = select(SystemConfig).where(SystemConfig.config_key == "token.hard_limit")

    warning = (await db.execute(stmt_warning)).scalar_one_or_none()
    hard = (await db.execute(stmt_hard)).scalar_one_or_none()

    # 查询当月消耗
    month_start = datetime.now(timezone.utc).replace(day=1, hour=0, minute=0, second=0, microsecond=0)
    stmt = select(func.sum(TokenUsage.input_tokens + TokenUsage.output_tokens)).where(TokenUsage.created_at >= month_start)
    current = (await db.execute(stmt)).scalar() or 0

    warning_val = int(warning.config_value) if warning else 80000
    hard_val = int(hard.config_value) if hard else 100000

    return {
        "warning_threshold": warning_val,
        "hard_limit": hard_val,
        "current_month_usage": current,
        "is_warning": current >= warning_val,
        "is_locked": current >= hard_val,
    }


@router.put("/quota")
async def set_quota(body: dict, db: AsyncSession = Depends(get_db)):
    """设置配额（管理员）。请求: {"warning_threshold": 80000, "hard_limit": 100000}"""
    for key, value in body.items():
        config_key = f"token.{'warning_threshold' if key == 'warning_threshold' else 'hard_limit'}"
        stmt = select(SystemConfig).where(SystemConfig.config_key == config_key)
        config = (await db.execute(stmt)).scalar_one_or_none()
        if config:
            config.config_value = str(value)
            config.updated_at = datetime.now(timezone.utc)
    await db.commit()
    return {"message": "配额已更新"}
```

---

## 11. SSE 实时推送

### 11.1 事件总线

```python
# backend/app/sse/bus.py

import asyncio
import json
import logging
from typing import Any

logger = logging.getLogger("sse")

class SSEEventBus:
    """
    全局 SSE 事件总线——基于 asyncio.Queue 的多播。

    - 每个 SSE 客户端连接时注册一个 asyncio.Queue
    - 任意模块通过 publish() 发送事件到所有已注册的队列
    - 客户端断开时自动移除队列
    """

    def __init__(self):
        self._queues: dict[str, asyncio.Queue] = {}
        self._counter = 0

    def register(self) -> tuple[str, asyncio.Queue]:
        """注册新连接，返回 (client_id, queue)。"""
        self._counter += 1
        client_id = f"client_{self._counter}"
        queue: asyncio.Queue = asyncio.Queue(maxsize=256)
        self._queues[client_id] = queue
        logger.info("SSE 客户端 %s 已连接 (当前连接数: %d)", client_id, len(self._queues))
        return client_id, queue

    def unregister(self, client_id: str):
        """移除断开连接。"""
        self._queues.pop(client_id, None)
        logger.info("SSE 客户端 %s 已断开 (当前连接数: %d)", client_id, len(self._queues))

    async def publish(self, event_type: str, data: dict[str, Any]):
        """向所有已连接客户端广播事件。"""
        dead = []
        message = f"event: {event_type}\ndata: {json.dumps(data, ensure_ascii=False)}\n\n"

        for cid, queue in self._queues.items():
            try:
                queue.put_nowait(message)
            except asyncio.QueueFull:
                # 客户端消费太慢，丢弃该事件并标记为 dead
                logger.warning("SSE 客户端 %s 队列已满，标记为断开", cid)
                dead.append(cid)

        for cid in dead:
            self.unregister(cid)


# 全局单例
sse_bus = SSEEventBus()
```

### 11.2 SSE 端点

```python
# backend/app/api/sse.py

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse
from app.sse.bus import sse_bus
import asyncio

router = APIRouter()

@router.get("/sse")
async def sse_endpoint(request: Request):
    """
    SSE 长连接端点。

    客户端通过 EventSource 连接，接收所有消息类型的事件推送。
    断开时 EventSource 会自动重连。
    """
    client_id, queue = sse_bus.register()

    async def event_generator():
        try:
            while True:
                # 检查客户端是否断开
                if await request.is_disconnected():
                    break

                try:
                    message = await asyncio.wait_for(queue.get(), timeout=15.0)
                    yield message
                except asyncio.TimeoutError:
                    # 发送心跳保持连接
                    yield ": heartbeat\n\n"
        finally:
            sse_bus.unregister(client_id)

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",     # Nginx 禁用缓冲
        },
    )
```

### 11.3 消息触发场景汇总

| 消息类型 | 触发位置 | 调用代码 |
|---------|---------|---------|
| `app_status` | 采集服务状态变化 / 分析任务状态变化 | `await sse_bus.publish("app_status", {...})` |
| `app_output` | 分析任务产生阶段性输出 | 分析引擎回调中调用 |
| `forum_log` | 采集服务写入新日志 | 采集日志变更检测到后调用 |
| `system_status` | 系统状态变更 | 状态服务检测到变化后调用 |
| `graph_ready` | 图谱构建完成 | 图谱 workflow 完成回调中调用 |
| `error` | 任意模块触发异常 | 异常捕获处调用 |

---

## 12. 文件清单

| 文件 | 职责 |
|------|------|
| `backend/app/models/config.py` | SystemConfig ORM |
| `backend/app/models/token_usage.py` | TokenUsage ORM |
| `backend/app/api/dashboard.py` | 控制台聚合 API |
| `backend/app/api/collection.py` | 采集任务 CRUD API |
| `backend/app/api/search.py` | 主题检索 API |
| `backend/app/api/reports.py` | 报告 API |
| `backend/app/api/graph.py` | 图谱 API |
| `backend/app/api/status.py` | 系统状态 API |
| `backend/app/api/config.py` | 配置管理 API |
| `backend/app/api/token_usage.py` | Token 消耗 API |
| `backend/app/api/sse.py` | SSE 端点 |
| `backend/app/sse/__init__.py` | 包初始化 |
| `backend/app/sse/bus.py` | SSEEventBus 事件总线 |
| `backend/app/services/dashboard_service.py` | 控制台聚合逻辑 |
| `backend/app/services/collector_manager.py` | 采集进程管理（共享自 LLD-03） |
