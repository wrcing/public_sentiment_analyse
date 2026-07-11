# 舆情分析平台详细设计说明书 — 网络采集服务

---

## 1. 引言

### 1.1 设计范围

本文档是网络采集服务的详细设计说明书，依据 HLD 第 3.2.3 节（网络采集服务模块）和 SRS 第 3.2 节（网络采集模块）编写。覆盖：

- 独立进程架构与启动方式
- 任务轮询机制（基于 PostgreSQL 的 `FOR UPDATE` 互斥锁）
- 平台采集器抽象接口与各平台实现
- 采集数据与日志的存储设计
- 状态机与异常恢复策略

### 1.2 与 HLD 的对应关系

| HLD 章节 | 本 LLD 覆盖 |
|----------|------------|
| 3.2.3 网络采集服务模块 | 进程架构、轮询流程、平台采集器 |
| 4.2.1 PostgreSQL — 采集任务/数据/日志 | 三个表的 DDL 与 ORM |
| 6.3.1 故障场景 | 采集服务崩溃恢复 |

---

## 2. 数据库表设计

本模块独占以下三张表。

### 2.1 采集任务表

```sql
CREATE TABLE collection_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,                     -- 任务名称（用户自定义）
    source_platforms JSONB NOT NULL DEFAULT '[]',             -- ["bilibili","weibo","xiaohongshu","douyin","zhihu","toutiao"]
    keywords        JSONB NOT NULL DEFAULT '[]',              -- 采集关键词列表
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',   -- pending | running | stopped | failed
    progress        INTEGER NOT NULL DEFAULT 0,               -- 0-100 进度百分比
    error_message   TEXT,                                     -- 失败时的错误信息
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collection_tasks_status ON collection_tasks(status);
CREATE INDEX idx_collection_tasks_created_at ON collection_tasks(created_at DESC);
```

### 2.2 采集数据表

```sql
CREATE TABLE collection_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES collection_tasks(id) ON DELETE CASCADE,
    platform        VARCHAR(30) NOT NULL,                     -- bilibili | weibo | xiaohongshu | douyin | zhihu | toutiao
    source_url      TEXT,                                     -- 原始 URL
    title           TEXT,
    content         TEXT NOT NULL,                            -- 正文/摘要
    author_name     VARCHAR(200),
    publish_time    TIMESTAMPTZ,                              -- 原始发布时间
    raw_json        JSONB,                                    -- 平台返回的完整 JSON 数据
    fetched_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collection_data_task_id ON collection_data(task_id);
CREATE INDEX idx_collection_data_platform ON collection_data(platform);
CREATE INDEX idx_collection_data_fetched_at ON collection_data(fetched_at DESC);
```

### 2.3 采集日志表

```sql
CREATE TABLE collection_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES collection_tasks(id) ON DELETE CASCADE,
    level           VARCHAR(10) NOT NULL DEFAULT 'INFO',     -- INFO | WARN | ERROR
    message         TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collection_logs_task_id ON collection_logs(task_id);
CREATE INDEX idx_collection_logs_created_at ON collection_logs(created_at DESC);
```

### 2.4 ORM 模型

```python
# collector/collector/models.py

from sqlalchemy import String, Integer, Text, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.ext.declarative import declarative_base
import uuid
from datetime import datetime

Base = declarative_base()

class CollectionTask(Base):
    __tablename__ = "collection_tasks"

    id              = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name            = mapped_column(String(200), nullable=False)
    source_platforms = mapped_column(JSONB, nullable=False, default=list)
    keywords        = mapped_column(JSONB, nullable=False, default=list)
    status          = mapped_column(String(20), nullable=False, default="pending")
    progress        = mapped_column(Integer, nullable=False, default=0)
    error_message   = mapped_column(Text, nullable=True)
    created_at      = mapped_column(nullable=False, default=datetime.utcnow)
    started_at      = mapped_column(nullable=True)
    finished_at     = mapped_column(nullable=True)
    updated_at      = mapped_column(nullable=False, default=datetime.utcnow, onupdate=datetime.utcnow)

class CollectionData(Base):
    __tablename__ = "collection_data"

    id          = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    task_id     = mapped_column(UUID(as_uuid=True), ForeignKey("collection_tasks.id", ondelete="CASCADE"), nullable=False)
    platform    = mapped_column(String(30), nullable=False)
    source_url  = mapped_column(Text, nullable=True)
    title       = mapped_column(Text, nullable=True)
    content     = mapped_column(Text, nullable=False)
    author_name = mapped_column(String(200), nullable=True)
    publish_time = mapped_column(nullable=True)
    raw_json    = mapped_column(JSONB, nullable=True)
    fetched_at  = mapped_column(nullable=False, default=datetime.utcnow)

class CollectionLog(Base):
    __tablename__ = "collection_logs"

    id        = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    task_id   = mapped_column(UUID(as_uuid=True), ForeignKey("collection_tasks.id", ondelete="CASCADE"), nullable=False)
    level     = mapped_column(String(10), nullable=False, default="INFO")
    message   = mapped_column(Text, nullable=False)
    created_at = mapped_column(nullable=False, default=datetime.utcnow)
```

---

## 3. 进程架构

### 3.1 启动方式

采集服务由 FastAPI 的 `lifespan` 通过 `subprocess.Popen` 拉起：

```python
# backend/app/main.py (lifespan 内)

import subprocess
import sys
from pathlib import Path

collector_proc = subprocess.Popen(
    [sys.executable, "-m", "collector.main"],
    cwd=Path(__file__).parent.parent / "collector",
    env={**__import__("os").environ, "PYTHONPATH": str(Path(__file__).parent.parent)},
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
)
```

### 3.2 主循环

```python
# collector/collector/main.py

import asyncio
import logging
import signal
import sys

from collector.poller import Poller
from collector.config import settings

logger = logging.getLogger("collector")

class CollectorService:
    def __init__(self):
        self.poller = Poller()
        self._running = False

    async def run(self):
        self._running = True
        logger.info("采集服务启动，轮询间隔: %ds", settings.POLL_INTERVAL_SECONDS)

        while self._running:
            try:
                task = await self.poller.fetch_next_task()
                if task:
                    await self.poller.execute_task(task)
                else:
                    await asyncio.sleep(settings.POLL_INTERVAL_SECONDS)
            except Exception as e:
                logger.error("采集服务异常: %s", e, exc_info=True)
                await asyncio.sleep(settings.POLL_INTERVAL_SECONDS)

    def shutdown(self):
        self._running = False
        logger.info("采集服务收到停止信号")

if __name__ == "__main__":
    service = CollectorService()

    loop = asyncio.new_event_loop()

    # 信号处理
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, service.shutdown)

    try:
        loop.run_until_complete(service.run())
    finally:
        loop.close()
```

---

## 4. 任务轮询与执行

### 4.1 轮询器

```python
# collector/collector/poller.py

import logging
from datetime import datetime, timezone
from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.dialects.postgresql import insert

from collector.config import settings
from collector.models import CollectionTask, CollectionData, CollectionLog
from collector.scrapers import get_scraper

logger = logging.getLogger("collector")

class Poller:
    def __init__(self):
        self.engine = create_async_engine(settings.DATABASE_URL, pool_size=5)
        self.session_factory = async_sessionmaker(self.engine, class_=AsyncSession, expire_on_commit=False)

    async def fetch_next_task(self) -> CollectionTask | None:
        """
        轮询数据库，捡取一个 pending 任务。

        使用 FOR UPDATE 实现互斥：如果同时有多个采集进程，只有一个能抢到同一任务。
        """
        async with self.session_factory() as session:
            async with session.begin():
                stmt = (
                    select(CollectionTask)
                    .where(CollectionTask.status == "pending")
                    .order_by(CollectionTask.created_at)
                    .limit(1)
                    .with_for_update(skip_locked=True)
                )
                result = await session.execute(stmt)
                task = result.scalar_one_or_none()

                if task:
                    task.status = "running"
                    task.started_at = datetime.now(timezone.utc)
                    await session.flush()
                    # 刷新后 detach，确保 session 关闭后仍可访问属性
                    session.expunge(task)

                return task

    async def execute_task(self, task: CollectionTask):
        """执行单个采集任务——并行抓取所有选中平台。"""
        async with self.session_factory() as session:
            # 记录日志
            session.add(CollectionLog(
                task_id=task.id,
                level="INFO",
                message=f"开始采集任务: {task.name}，平台: {task.source_platforms}",
            ))
            await session.flush()

        platforms = task.source_platforms  # ["bilibili", "weibo", ...]
        keywords = task.keywords or []

        # 并行抓取所有平台
        tasks = []
        for platform_name in platforms:
            scraper_cls = get_scraper(platform_name)
            if scraper_cls:
                scraper = scraper_cls()
                tasks.append(scraper.scrape(str(task.id), keywords))

        results = await asyncio.gather(*tasks, return_exceptions=True)

        # 汇总结果写入数据库
        total_items = 0
        async with self.session_factory() as session:
            for i, result in enumerate(results):
                platform = platforms[i]
                if isinstance(result, Exception):
                    session.add(CollectionLog(
                        task_id=task.id,
                        level="ERROR",
                        message=f"平台 {platform} 采集失败: {str(result)}",
                    ))
                else:
                    for item in result:  # result 是 list[dict]
                        session.add(CollectionData(
                            task_id=task.id,
                            platform=platform,
                            source_url=item.get("url"),
                            title=item.get("title"),
                            content=item.get("content", ""),
                            author_name=item.get("author"),
                            publish_time=item.get("publish_time"),
                            raw_json=item.get("raw"),
                        ))
                        total_items += 1

            # 更新任务状态
            task_obj = await session.get(CollectionTask, task.id)
            if task_obj:
                task_obj.status = "stopped"
                task_obj.progress = 100
                task_obj.finished_at = datetime.now(timezone.utc)

            session.add(CollectionLog(
                task_id=task.id,
                level="INFO",
                message=f"采集完成: 共获取 {total_items} 条数据",
            ))

            await session.commit()
```

---

## 5. 平台采集器

### 5.1 抽象接口

```python
# collector/collector/scrapers/__init__.py

from abc import ABC, abstractmethod

class BaseScraper(ABC):
    """平台采集器抽象基类。"""

    # 子类需覆盖的平台标识
    platform_name: str = ""

    @abstractmethod
    async def scrape(self, task_id: str, keywords: list[str]) -> list[dict]:
        """
        执行采集。

        参数:
            task_id:  采集任务 ID
            keywords: 用户指定的关键词列表

        返回:
            list[dict]，每个 dict 包含:
                - url:           str  原始链接
                - title:         str  标题
                - content:       str  正文内容
                - author:        str  作者名
                - publish_time:  str  ISO 8601 时间字符串
                - raw:           dict 平台返回的原始 JSON
        """
        ...


# 平台注册表
_scrapers: dict[str, type[BaseScraper]] = {}

def register_scraper(name: str):
    """装饰器——注册采集器类。"""
    def decorator(cls: type[BaseScraper]):
        _scrapers[name] = cls
        return cls
    return decorator

def get_scraper(name: str) -> type[BaseScraper] | None:
    return _scrapers.get(name)
```

### 5.2 平台实现模板（以 B 站为例）

```python
# collector/collector/scrapers/bilibili.py

import httpx
from . import BaseScraper, register_scraper

@register_scraper("bilibili")
class BilibiliScraper(BaseScraper):
    platform_name = "bilibili"

    async def scrape(self, task_id: str, keywords: list[str]) -> list[dict]:
        """
        B 站采集逻辑。

        首版实现策略（简化）:
        - 调用 B 站搜索 API: https://api.bilibili.com/x/web-interface/search/type?keyword=xxx
        - 解析返回结果，取前 50 条
        - 对每条结果构造标准 dict
        """
        results = []
        async with httpx.AsyncClient(timeout=30) as client:
            for kw in keywords[:5]:  # 最多 5 个关键词，避免过度请求
                try:
                    resp = await client.get(
                        "https://api.bilibili.com/x/web-interface/search/type",
                        params={"keyword": kw, "search_type": "video", "page": 1},
                        headers={
                            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
                            "Referer": "https://www.bilibili.com/",
                        },
                    )
                    resp.raise_for_status()
                    data = resp.json()

                    for item in data.get("data", {}).get("result", [])[:20]:
                        results.append({
                            "url": item.get("arcurl", ""),
                            "title": item.get("title", "").replace('<em class="keyword">', '').replace('</em>', ''),
                            "content": item.get("description", ""),
                            "author": item.get("author", ""),
                            "publish_time": None,  # B 站 API 返回 pubdate 为时间戳，需转换
                            "raw": item,
                        })
                except Exception as e:
                    # 单个关键词失败不阻塞整体
                    continue

        return results
```

其他五个平台（微博、小红书、抖音、知乎、头条）参照上述模板实现，各自继承 `BaseScraper` 并用 `@register_scraper("weibo")` 等注册。首版各平台采集前 20-50 条搜索结果即可。

### 5.3 配置

```python
# collector/collector/config.py

from pydantic_settings import BaseSettings

class CollectorSettings(BaseSettings):
    DATABASE_URL: str                          # 必须与后端指向同一 PostgreSQL
    POLL_INTERVAL_SECONDS: int = 5
    REQUEST_TIMEOUT_SECONDS: int = 30
    MAX_KEYWORDS_PER_TASK: int = 5
    MAX_ITEMS_PER_PLATFORM: int = 50

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

settings = CollectorSettings()
```

---

## 6. 状态机

### 6.1 任务状态流转

```
                 ┌──────────┐
                 │ pending  │  ← 用户通过 API POST /api/collection/tasks 创建
                 └────┬─────┘
                      │ 采集服务捡取 (FOR UPDATE)
                      ▼
                 ┌──────────┐
          ┌──────│ running  │───────┐
          │      └────┬─────┘       │
          │           │ 全部平台完成  │ 采集过程中异常且不可恢复
          │           ▼              ▼
          │      ┌──────────┐   ┌──────────┐
          │      │ stopped  │   │  failed  │
          │      └──────────┘   └──────────┘
          │
          └── 采集服务崩溃，进程管理模块自动重启后：
              采集服务扫描 running 状态的任务 → 重置为 pending → 重新执行
```

### 6.2 崩溃恢复

```python
# collector/collector/poller.py (启动时调用)

async def recover_orphaned_tasks(self):
    """
    启动时将上次未完成的 running 任务重置为 pending。

    这些任务可能是上次采集服务崩溃留下的。
    """
    async with self.session_factory() as session:
        async with session.begin():
            stmt = (
                update(CollectionTask)
                .where(CollectionTask.status == "running")
                .values(status="pending", error_message="上次采集异常中断，已自动重置")
            )
            await session.execute(stmt)
```

### 6.3 最大重启次数

FastAPI 端的进程管理模块追踪采集服务的重启次数。连续崩溃 ≥ 3 次时不再重启，标记为 `failed` 并通过 SSE 推送错误事件。

```python
# backend/app/services/collector_manager.py

class CollectorManager:
    def __init__(self):
        self._proc: subprocess.Popen | None = None
        self._restart_count = 0
        self._max_restart = settings.COLLECTOR_MAX_RESTART  # 默认 3

    def start(self):
        self._launch()

    def _launch(self):
        self._proc = subprocess.Popen([...])
        # 启动一个后台 asyncio task 监控进程状态
        asyncio.create_task(self._monitor())

    async def _monitor(self):
        while True:
            ret = self._proc.poll()
            if ret is not None:  # 进程已退出
                self._restart_count += 1
                if self._restart_count < self._max_restart:
                    logger.warning("采集服务异常退出，第 %d 次重启", self._restart_count)
                    self._launch()
                    # SSE 推送
                    await sse_bus.publish("error", {
                        "module_name": "collector",
                        "error_message": f"采集服务异常退出，第 {self._restart_count} 次重启",
                    })
                else:
                    logger.error("采集服务连续崩溃 %d 次，停止重试", self._max_restart)
                    await sse_bus.publish("app_status", {
                        "app_name": "collector",
                        "status": "failed",
                    })
                    break
            await asyncio.sleep(5)
```

---

## 7. 日志

采集服务使用独立的 logger（`logging.getLogger("collector")`），日志同时写入：

1. **本地文件**：`data/logs/collector.log`（每日轮转，保留 30 天）
2. **数据库 `collection_logs` 表**：供 API 后端查询并通过 SSE 推送给前端

```python
# collector/collector/db_log_handler.py

import logging

class DBLogHandler(logging.Handler):
    """将日志同步写入 collection_logs 表的 handler。"""

    def __init__(self, session_factory, task_id: str | None = None):
        super().__init__()
        self.session_factory = session_factory
        self.task_id = task_id

    def emit(self, record: logging.LogRecord):
        # 异步写入——不阻塞采集主流程
        async def _write():
            async with self.session_factory() as session:
                session.add(CollectionLog(
                    task_id=self.task_id,
                    level=record.levelname,
                    message=self.format(record),
                ))
                await session.commit()

        try:
            loop = asyncio.get_event_loop()
            loop.create_task(_write())
        except RuntimeError:
            pass  # event loop 未就绪则跳过
```

---

## 8. 文件清单

| 文件 | 职责 |
|------|------|
| `collector/pyproject.toml` | uv 项目定义 + 依赖 |
| `collector/collector/__init__.py` | 包初始化 |
| `collector/collector/main.py` | 服务入口 + 主循环 + 信号处理 |
| `collector/collector/config.py` | 采集服务配置 |
| `collector/collector/models.py` | CollectionTask / CollectionData / CollectionLog ORM |
| `collector/collector/poller.py` | 任务轮询、执行、崩溃恢复 |
| `collector/collector/scrapers/__init__.py` | BaseScraper 抽象类 + 注册表 |
| `collector/collector/scrapers/bilibili.py` | B 站采集器 |
| `collector/collector/scrapers/weibo.py` | 微博采集器 |
| `collector/collector/scrapers/xiaohongshu.py` | 小红书采集器 |
| `collector/collector/scrapers/douyin.py` | 抖音采集器 |
| `collector/collector/scrapers/zhihu.py` | 知乎采集器 |
| `collector/collector/scrapers/toutiao.py` | 头条采集器 |
| `collector/collector/db_log_handler.py` | 日志写入数据库 Handler |
| `backend/app/services/collector_manager.py` | 采集进程管理（启动/监控/重启） |
