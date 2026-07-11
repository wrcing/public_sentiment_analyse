# 舆情分析平台详细设计说明书 — 项目基础设施

---

## 1. 引言

### 1.1 设计范围

本文档是舆情分析平台项目基础设施的详细设计说明书（LLD），依据《概要设计说明书》（HLD）第 2、6 章编写，覆盖以下内容：

- 项目顶层目录结构
- 应用配置管理（多环境、加密敏感项）
- 日志系统（按模块分文件、轮转策略）
- 进程管理（启动脚本、systemd 配置）
- 前端项目骨架（Vite + Vue 3 + Element Plus 初始化）

**不在本 LLD 范围**：具体业务模块的目录结构、数据库表、API 端点——这些在各自的 LLD 中定义。

### 1.2 与 HLD 的对应关系

| HLD 章节 | 本 LLD 覆盖 |
|----------|------------|
| 2.2 技术选型 | 各技术的项目级落地方式 |
| 2.3 系统分层 | backend/frontend 顶层目录 |
| 6.2 基础设施需求 | 进程配置、systemd unit |
| 6.3 高可用与容错 | 日志轮转、崩溃重启 |

---

## 2. 项目顶层目录结构

```
public_sentiment_analyse/
├── backend/
│   ├── pyproject.toml          # uv 项目定义 + 依赖声明
│   ├── alembic.ini             # Alembic 配置
│   ├── alembic/                # 数据库迁移脚本
│   │   ├── env.py
│   │   └── versions/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py             # FastAPI 应用工厂 + 生命周期
│   │   ├── core/
│   │   │   ├── __init__.py
│   │   │   ├── config.py       # 配置加载与管理
│   │   │   ├── database.py     # SQLAlchemy async engine + session
│   │   │   ├── security.py     # 密码哈希、JWT 工具
│   │   │   └── logging_config.py  # 日志初始化
│   │   ├── models/             # SQLAlchemy ORM 模型（按 LLD 分摊）
│   │   ├── api/                # 路由注册（按 LLD 分摊）
│   │   ├── services/           # 业务服务（按 LLD 分摊）
│   │   ├── engine/             # 分析引擎（LLD-04）
│   │   └── sse/                # SSE 事件总线（LLD-05）
│   ├── tests/
│   └── scripts/
│       └── start_collector.sh  # 采集服务启动脚本
├── collector/                  # 网络采集服务（LLD-03）
│   ├── pyproject.toml
│   └── collector/
│       ├── __init__.py
│       ├── main.py
│       ├── poller.py
│       ├── scrapers/
│       └── models.py
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   ├── index.html
│   ├── src/
│   │   ├── main.ts
│   │   ├── App.vue
│   │   ├── router/
│   │   ├── stores/
│   │   ├── api/
│   │   ├── components/
│   │   ├── pages/
│   │   └── utils/
│   └── public/
├── docs/                       # 设计文档
├── data/                       # 本地文件存储（日志/图谱导出）
│   ├── logs/
│   └── graph_exports/
├── deploy/
│   ├── nginx.conf
│   └── systemd/
└── docker-compose.langfuse.yml  # LangFuse 部署
```

---

## 3. 应用配置管理

### 3.1 配置文件结构

```python
# backend/app/core/config.py

from pydantic_settings import BaseSettings
from pydantic import SecretStr
from typing import Optional
from enum import Enum

class Environment(str, Enum):
    DEV = "dev"
    PROD = "prod"

class Settings(BaseSettings):
    # ---- 运行环境 ----
    ENV: Environment = Environment.DEV
    DEBUG: bool = False

    # ---- 服务端点 ----
    HOST: str = "127.0.0.1"
    PORT: int = 8000
    WORKERS: int = 4

    # ---- 数据库 ----
    DATABASE_URL: str  # postgresql+asyncpg://user:pass@host:5432/db

    # ---- JWT ----
    JWT_SECRET_KEY: SecretStr
    JWT_ALGORITHM: str = "HS256"
    JWT_ACCESS_TOKEN_EXPIRE_MINUTES: int = 60

    # ---- LLM ----
    LLM_API_BASE: str = "https://api.openai.com/v1"
    LLM_API_KEY: SecretStr
    LLM_MODEL_NAME: str = "gpt-4o"
    LLM_TEMPERATURE: float = 0.3
    LLM_MAX_TOKENS: int = 4096

    # ---- LangFuse ----
    LANGFUSE_PUBLIC_KEY: Optional[str] = None
    LANGFUSE_SECRET_KEY: Optional[SecretStr] = None
    LANGFUSE_HOST: Optional[str] = None

    # ---- 采集服务 ----
    COLLECTOR_POLL_INTERVAL_SECONDS: int = 5
    COLLECTOR_MAX_RESTART: int = 3

    # ---- 分析引擎 ----
    ENGINE_THREAD_POOL_SIZE: int = 4
    ENGINE_TASK_TIMEOUT_MINUTES: int = 10

    # ---- Token 配额 ----
    TOKEN_WARNING_THRESHOLD: int = 80_000   # 预警阈值
    TOKEN_HARD_LIMIT: int = 100_000          # 硬上限

    # ---- 日志 ----
    LOG_LEVEL: str = "INFO"
    LOG_DIR: str = "../data/logs"
    LOG_RETENTION_DAYS: int = 30

    # ---- 数据清理 ----
    COLLECTION_DATA_RETENTION_DAYS: int = 90

    model_config = {
        "env_file": ".env",
        "env_file_encoding": "utf-8",
    }

settings = Settings()
```

### 3.2 多环境策略

| 环境 | 配置文件 | 加载方式 |
|------|---------|---------|
| 开发 (Windows) | `backend/.env.dev` | 通过 `ENV=dev` 加载 |
| 生产 (Linux) | `backend/.env.prod` | 通过 `ENV=prod` 加载 |

敏感项（`JWT_SECRET_KEY`、`LLM_API_KEY`、`LANGFUSE_SECRET_KEY`）使用 `pydantic.SecretStr` 包装，打印和序列化时自动隐藏。

### 3.3 配置热更新

LLD-05 的配置管理服务会维护一份内存中的动态配置（读取自 `system_configs` 表），运行时优先读取内存配置，仅在启动时使用文件配置兜底。

---

## 4. 数据库连接管理

```python
# backend/app/core/database.py

from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

engine = create_async_engine(
    settings.DATABASE_URL,
    pool_size=20,          # 并发连接
    max_overflow=10,       # 溢出连接
    pool_pre_ping=True,    # 连接前检测有效性
    echo=settings.DEBUG,
)

async_session = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)

async def get_db() -> AsyncSession:
    """FastAPI 依赖注入——每个请求一个 session，请求结束自动关闭。"""
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### 4.1 事务边界约定

- **读操作**：自动提交（`get_db` 在无异常时 commit，实际无变更）
- **写操作**：在 service 层调用 `session.flush()` 获取 ID，由 `get_db` 统一 commit
- **跨表写入**：多个 service 调用在同一请求内共享同一个 session，实现事务原子性
- **采集服务**：自行管理 session，每次轮询独立事务

---

## 5. 日志系统

### 5.1 初始化

```python
# backend/app/core/logging_config.py

import logging
import logging.handlers
from pathlib import Path
from .config import settings

def setup_logging():
    log_dir = Path(settings.LOG_DIR)
    log_dir.mkdir(parents=True, exist_ok=True)

    # 根 logger
    root = logging.getLogger()
    root.setLevel(settings.LOG_LEVEL)

    # 格式
    fmt = logging.Formatter(
        "%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
    )

    # 控制台
    console = logging.StreamHandler()
    console.setFormatter(fmt)
    root.addHandler(console)

    # 按模块分文件 + 每日轮转
    modules = ["api", "engine", "collector", "sse", "app"]
    for mod in modules:
        handler = logging.handlers.TimedRotatingFileHandler(
            filename=log_dir / f"{mod}.log",
            when="midnight",
            interval=1,
            backupCount=settings.LOG_RETENTION_DAYS,
            encoding="utf-8",
        )
        handler.setFormatter(fmt)
        logger = logging.getLogger(mod)
        logger.addHandler(handler)
        logger.setLevel(settings.LOG_LEVEL)
```

### 5.2 各模块 logger 名称

| 模块 | logger 名 | 日志文件 |
|------|----------|---------|
| API 网关层 | `api` | `api.log` |
| 分析引擎 | `engine` | `engine.log` |
| 采集服务 | `collector` | `collector.log` |
| SSE 推送 | `sse` | `sse.log` |
| 应用生命周期 | `app` | `app.log` |

采集服务有自己的日志初始化（独立进程），但复用相同的 `logging_config.py`。

---

## 6. FastAPI 应用工厂

```python
# backend/app/main.py

from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from .core.config import settings
from .core.logging_config import setup_logging
from .core.database import engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动
    setup_logging()
    # 启动采集服务子进程（LLD-03 的进程管理模块）
    # yield 前可做预热（如加载配置到内存）

    yield

    # 关闭
    await engine.dispose()
    # 停止采集服务子进程

def create_app() -> FastAPI:
    app = FastAPI(
        title="舆情分析平台",
        version="0.1.0",
        lifespan=lifespan,
    )

    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"] if settings.DEBUG else ["http://localhost"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # 路由注册（各 LLD 分摊，在此集中 import 和 include）
    # from .api.auth import router as auth_router
    # app.include_router(auth_router, prefix="/api/auth", tags=["认证"])
    # ... 其余路由见 LLD-02 / LLD-05

    return app

app = create_app()
```

### 6.1 uvicorn 启动命令

```bash
# 开发环境
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000

# 生产环境
uvicorn app.main:app --host 127.0.0.1 --port 8000 --workers 4 --log-level info
```

---

## 7. 进程管理

### 7.1 进程清单与启动方式

| 进程 | 启动方式 | 说明 |
|------|---------|------|
| FastAPI (uvicorn) | systemd 或手动 `uvicorn` | 主后端服务 |
| 网络采集服务 | FastAPI lifespan 中 `subprocess.Popen` 拉起 | LLD-03 详述 |
| LangFuse | `docker compose -f docker-compose.langfuse.yml up -d` | 仅 Token 追踪用 |

### 7.2 systemd unit 模板

```ini
# deploy/systemd/public-sentiment-api.service

[Unit]
Description=舆情分析平台 - API 后端
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=app
WorkingDirectory=/opt/public_sentiment/backend
ExecStart=/home/app/.local/bin/uv run uvicorn app.main:app --host 127.0.0.1 --port 8000 --workers 4
Restart=always
RestartSec=10
Environment=ENV=prod
StandardOutput=append:/opt/public_sentiment/data/logs/app.log
StandardError=append:/opt/public_sentiment/data/logs/app.log

[Install]
WantedBy=multi-user.target
```

### 7.3 启动/关机顺序

```
启动: PostgreSQL → LangFuse (docker) → FastAPI (自动拉起采集服务) → Nginx
关机: Nginx stop → FastAPI (等待任务完成，最长 5min) → 采集服务 stop → LangFuse stop → PostgreSQL stop
```

---

## 8. 前端项目骨架

### 8.1 初始化命令

```bash
cd frontend
npm create vite@latest . -- --template vue-ts
npm install vue-router@4 pinia axios element-plus @element-plus/icons-vue
npm install -D @types/node sass
```

### 8.2 入口文件

```typescript
// frontend/src/main.ts

import { createApp } from 'vue'
import { createPinia } from 'pinia'
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'
import App from './App.vue'
import router from './router'

const app = createApp(App)
app.use(createPinia())
app.use(router)
app.use(ElementPlus)
app.mount('#app')
```

### 8.3 Vite 代理配置

```typescript
// frontend/vite.config.ts

import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://127.0.0.1:8000',
        changeOrigin: true,
      },
      '/sse': {
        target: 'http://127.0.0.1:8000',
        changeOrigin: true,
        // SSE 长连接不做超时限制
        proxy: { timeout: 0 },
      },
    },
  },
})
```

---

## 9. Nginx 配置

```nginx
# deploy/nginx.conf

server {
    listen 80;
    server_name localhost;

    # Vue SPA 静态资源
    root /opt/public_sentiment/frontend/dist;
    index index.html;

    # API 代理
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # SSE 代理（长连接）
    location /sse {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_buffering off;            # 关闭缓冲
        proxy_cache off;
        proxy_read_timeout 300s;        # 5 分钟超时
        chunked_transfer_encoding on;
    }

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 10. Python 依赖

### 10.1 backend/pyproject.toml

```toml
[project]
name = "public-sentiment-backend"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115,<1.0",
    "uvicorn[standard]>=0.30",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.29",
    "alembic>=1.13",
    "pydantic>=2.0",
    "pydantic-settings>=2.0",
    "python-jose[cryptography]>=3.3",
    "passlib[bcrypt]>=1.7",
    "langgraph>=0.2",
    "langchain>=0.3",
    "langchain-openai>=0.2",
    "langfuse>=2.50",
    "httpx>=0.27",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "httpx>=0.27",
]
```

### 10.2 collector/pyproject.toml

```toml
[project]
name = "public-sentiment-collector"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.29",
    "httpx>=0.27",
    "pydantic>=2.0",
    "pydantic-settings>=2.0",
    "beautifulsoup4>=4.12",
    "lxml>=5.1",
]
```
