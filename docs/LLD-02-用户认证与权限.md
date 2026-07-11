# 舆情分析平台详细设计说明书 — 用户认证与权限模块

---

## 1. 引言

### 1.1 设计范围

本文档是用户认证与权限模块的详细设计说明书，依据 HLD 第 3.2.2 节 API 后端模块（路由/认证中间件、用户/角色管理）及 SRS 第 4.2 节安全需求编写。覆盖：

- 用户注册/登录/登出全流程
- JWT Token 签发与验证
- 基于角色的权限控制（RBAC）
- 认证相关的 API 端点（请求/响应格式）
- 前端登录页面与路由守卫

### 1.2 与 HLD 的对应关系

| HLD 章节 | 本 LLD 覆盖 |
|----------|------------|
| 4.2.1 PostgreSQL | `users`, `roles`, `user_roles` 表 |
| 5.1.1 认证类接口 | `/api/auth/login`, `/api/auth/logout`, `/api/auth/me` |
| 3.2.2 路由/认证中间件 | JWT 验证中间件、RBAC 装饰器 |

---

## 2. 数据库表设计

### 2.1 表 DDL

```sql
-- 用户表
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(50)  NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,              -- bcrypt hash
    display_name    VARCHAR(100),                       -- 显示名称
    email           VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,      -- 是否启用
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 角色表
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(50) NOT NULL UNIQUE,        -- admin / operator / viewer
    description     VARCHAR(255)
);

-- 用户-角色关联表
CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

-- 索引
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
```

### 2.2 预置角色

| 角色名 | 对应 SRS 角色 | 权限说明 |
|--------|-------------|---------|
| `admin` | 系统管理员 | 全部权限：控制台、采集、检索、报告、图谱、配置、状态、Token 管理 |
| `operator` | 操作用户 | 控制台首页、启动采集、发起检索、生成报告 |
| `viewer` | 报告查看人 | 查看报告、图谱、检索结果 |

### 2.3 SQLAlchemy ORM 模型

```python
# backend/app/models/user.py

import uuid
from sqlalchemy import String, Boolean, ForeignKey, Table, Column
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID
from app.core.database import Base

user_roles_table = Table(
    "user_roles",
    Base.metadata,
    Column("user_id", UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), primary_key=True),
    Column("role_id", UUID(as_uuid=True), ForeignKey("roles.id", ondelete="CASCADE"), primary_key=True),
)

class User(Base):
    __tablename__ = "users"

    id:            Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    username:      Mapped[str]       = mapped_column(String(50), unique=True, nullable=False, index=True)
    password_hash: Mapped[str]       = mapped_column(String(255), nullable=False)
    display_name:  Mapped[str | None] = mapped_column(String(100))
    email:         Mapped[str | None] = mapped_column(String(255))
    is_active:     Mapped[bool]      = mapped_column(Boolean, default=True)
    created_at:    Mapped[str]       = mapped_column(server_default="now()")  # 实际类型为 TIMESTAMPTZ
    updated_at:    Mapped[str]       = mapped_column(server_default="now()")

    roles: Mapped[list["Role"]] = relationship(
        secondary=user_roles_table,
        back_populates="users",
        lazy="selectin",
    )

class Role(Base):
    __tablename__ = "roles"

    id:          Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name:        Mapped[str]       = mapped_column(String(50), unique=True, nullable=False)
    description: Mapped[str | None] = mapped_column(String(255))

    users: Mapped[list["User"]] = relationship(
        secondary=user_roles_table,
        back_populates="roles",
        lazy="selectin",
    )
```

---

## 3. JWT 认证流程

### 3.1 Token 签发

```
用户输入 username + password
       │
       ▼
POST /api/auth/login
       │
       ▼
验证用户名存在 ∧ 密码 bcrypt 匹配 ∧ is_active = true
       │ 失败 → 401 "用户名或密码错误"
       ▼
生成 access_token: JWT { sub: user_id, roles: [...], exp: now + 60min }
       │
       ▼
返回 { access_token, token_type: "bearer", user: { id, username, display_name, roles } }
```

### 3.2 Token 验证

```
请求携带 Authorization: Bearer <token>
       │
       ▼
中间件解析 JWT → 验证签名 → 验证过期 → 提取 sub + roles
       │ 任一失败 → 401
       ▼
注入 request.state.user_id, request.state.roles
       │
       ▼
路由处理函数通过 Depends(get_current_user) 获取
```

### 3.3 密码安全

```python
# backend/app/core/security.py

from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def create_access_token(user_id: str, roles: list[str]) -> str:
    from jose import jwt
    from datetime import datetime, timedelta, timezone
    from .config import settings

    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.JWT_ACCESS_TOKEN_EXPIRE_MINUTES)
    payload = {
        "sub": user_id,
        "roles": roles,
        "exp": expire,
        "iat": datetime.now(timezone.utc),
    }
    return jwt.encode(payload, settings.JWT_SECRET_KEY.get_secret_value(), algorithm=settings.JWT_ALGORITHM)

def decode_access_token(token: str) -> dict:
    from jose import jwt, JWTError
    try:
        return jwt.decode(token, settings.JWT_SECRET_KEY.get_secret_value(), algorithms=[settings.JWT_ALGORITHM])
    except JWTError:
        return {}
```

---

## 4. API 端点详细设计

### 4.1 路由文件结构

```python
# backend/app/api/auth.py

from fastapi import APIRouter, Depends, HTTPException, status

router = APIRouter()

# 端点实现在此文件中
```

### 4.2 POST /api/auth/login

```python
# ---- 请求 ----
class LoginRequest(BaseModel):
    username: str = Field(..., min_length=2, max_length=50)
    password: str = Field(..., min_length=6)

# ---- 响应 ----
class UserBrief(BaseModel):
    id: str
    username: str
    display_name: str | None
    roles: list[str]

class LoginResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    user: UserBrief

@router.post("/login", response_model=LoginResponse)
async def login(body: LoginRequest, db: AsyncSession = Depends(get_db)):
    """
    用户登录。

    1. 查询用户
    2. 验证密码
    3. 检查 is_active
    4. 生成 JWT
    5. 返回 token + user info
    """
    stmt = select(User).where(User.username == body.username)
    result = await db.execute(stmt)
    user = result.scalar_one_or_none()

    if not user or not verify_password(body.password, user.password_hash):
        raise HTTPException(status_code=401, detail="用户名或密码错误")

    if not user.is_active:
        raise HTTPException(status_code=403, detail="账户已被禁用")

    roles = [r.name for r in user.roles]
    token = create_access_token(str(user.id), roles)

    return LoginResponse(
        access_token=token,
        user=UserBrief(
            id=str(user.id),
            username=user.username,
            display_name=user.display_name,
            roles=roles,
        ),
    )
```

### 4.3 POST /api/auth/logout

```python
@router.post("/logout")
async def logout(current_user: User = Depends(get_current_user)):
    """
    登出。首版为无状态 JWT，服务端不做额外处理。
    前端删除本地 token 即可。
    """
    return {"message": "已登出"}
```

### 4.4 GET /api/auth/me

```python
@router.get("/me", response_model=UserBrief)
async def get_me(current_user: User = Depends(get_current_user)):
    """
    获取当前登录用户信息。
    """
    return UserBrief(
        id=str(current_user.id),
        username=current_user.username,
        display_name=current_user.display_name,
        roles=[r.name for r in current_user.roles],
    )
```

---

## 5. 权限中间件

### 5.1 获取当前用户

```python
# backend/app/api/deps.py

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.core.database import get_db
from app.core.security import decode_access_token
from app.models.user import User

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    """解析 JWT，返回当前用户 ORM 对象。"""
    payload = decode_access_token(credentials.credentials)
    if not payload or "sub" not in payload:
        raise HTTPException(status_code=401, detail="无效的认证令牌")

    stmt = select(User).where(User.id == payload["sub"])
    result = await db.execute(stmt)
    user = result.scalar_one_or_none()

    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="用户不存在或已禁用")

    return user
```

### 5.2 RBAC 权限检查

```python
# backend/app/api/deps.py (续)

from functools import wraps
from typing import Callable

def require_role(*allowed_roles: str) -> Callable:
    """
    FastAPI 依赖工厂——限制只有指定角色才能访问。

    用法:
        @router.post("/config")
        async def update_config(
            user: User = Depends(require_role("admin")),
        ):
            ...
    """
    async def role_checker(
        current_user: User = Depends(get_current_user),
    ) -> User:
        user_roles = {r.name for r in current_user.roles}
        if not user_roles.intersection(allowed_roles):
            raise HTTPException(status_code=403, detail="权限不足")
        return current_user

    return role_checker
```

### 5.3 角色-端点映射

| 端点前缀 | 最低角色 |
|---------|---------|
| `/api/auth/*` | 无需认证（除 `/me`） |
| `/api/dashboard/*` | `operator` |
| `/api/collection/*` | `operator` |
| `/api/search/*` | `operator` |
| `/api/reports/*`（GET） | `viewer` |
| `/api/reports/*`（POST） | `operator` |
| `/api/graph/*` | `viewer` |
| `/api/status/*` | `admin` |
| `/api/config/*` | `admin` |
| `/api/token-usage/*` | `admin` |

---

## 6. 种子数据

系统首次启动时需要预置角色和一个默认管理员账号。

```python
# backend/app/core/seed.py

async def seed_roles(db: AsyncSession):
    """创建预置角色（幂等——已存在则跳过）。"""
    preset_roles = [
        ("admin",     "系统管理员"),
        ("operator",  "操作用户"),
        ("viewer",    "报告查看人"),
    ]
    for name, desc in preset_roles:
        stmt = select(Role).where(Role.name == name)
        existing = (await db.execute(stmt)).scalar_one_or_none()
        if not existing:
            db.add(Role(name=name, description=desc))

    await db.flush()

async def seed_admin_user(db: AsyncSession):
    """创建默认管理员（用户名 admin，密码通过环境变量 ADMIN_PASSWORD 传入）。"""
    import os
    from app.core.security import hash_password

    stmt = select(User).where(User.username == "admin")
    existing = (await db.execute(stmt)).scalar_one_or_none()
    if existing:
        return

    from .config import settings
    admin_pw = os.getenv("ADMIN_PASSWORD", "admin123")

    admin = User(
        username="admin",
        password_hash=hash_password(admin_pw),
        display_name="系统管理员",
        is_active=True,
    )
    stmt = select(Role).where(Role.name == "admin")
    admin_role = (await db.execute(stmt)).scalar_one()
    admin.roles.append(admin_role)
    db.add(admin)
```

在 `main.py` 的 `lifespan` 启动阶段调用 `seed_roles` 和 `seed_admin_user`。

---

## 7. 前端设计

### 7.1 路由守卫

```typescript
// frontend/src/router/index.ts

import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/auth'

const routes = [
  {
    path: '/login',
    name: 'Login',
    component: () => import('@/pages/LoginPage.vue'),
    meta: { requiresAuth: false },
  },
  {
    path: '/',
    component: () => import('@/layouts/MainLayout.vue'),
    meta: { requiresAuth: true },
    children: [
      {
        path: '',
        name: 'Dashboard',
        component: () => import('@/pages/DashboardPage.vue'),
      },
      // ... 其余路由见 LLD-06
    ],
  },
]

const router = createRouter({
  history: createWebHistory(),
  routes,
})

router.beforeEach((to, _from, next) => {
  const auth = useAuthStore()
  if (to.meta.requiresAuth !== false && !auth.isLoggedIn) {
    next({ name: 'Login', query: { redirect: to.fullPath } })
  } else if (to.name === 'Login' && auth.isLoggedIn) {
    next({ name: 'Dashboard' })
  } else {
    next()
  }
})

export default router
```

### 7.2 Auth Store

```typescript
// frontend/src/stores/auth.ts

import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { api } from '@/api/client'
import router from '@/router'

interface UserBrief {
  id: string
  username: string
  display_name: string | null
  roles: string[]
}

export const useAuthStore = defineStore('auth', () => {
  const user = ref<UserBrief | null>(null)
  const token = ref<string | null>(localStorage.getItem('access_token'))

  const isLoggedIn = computed(() => !!token.value)
  const isAdmin = computed(() => user.value?.roles.includes('admin') ?? false)

  async function login(username: string, password: string) {
    const res = await api.post('/api/auth/login', { username, password })
    token.value = res.data.access_token
    user.value = res.data.user
    localStorage.setItem('access_token', token.value!)
  }

  async function fetchUser() {
    if (!token.value) return
    try {
      const res = await api.get('/api/auth/me')
      user.value = res.data
    } catch {
      logout()
    }
  }

  function logout() {
    token.value = null
    user.value = null
    localStorage.removeItem('access_token')
    router.push({ name: 'Login' })
  }

  return { user, token, isLoggedIn, isAdmin, login, fetchUser, logout }
})
```

### 7.3 登录页面组件

```
┌──────────────────────────────────────────┐
│                                          │
│           舆情分析平台                     │
│                                          │
│    ┌─────────────────────────────────┐   │
│    │ 用户名: [________________]      │   │
│    │ 密码:   [________________]      │   │
│    │                                 │   │
│    │ [        登  录        ]        │   │
│    └─────────────────────────────────┘   │
│                                          │
│    登录失败时显示错误提示 (el-message)      │
└──────────────────────────────────────────┘
```

```vue
<!-- frontend/src/pages/LoginPage.vue -->

<script setup lang="ts">
import { ref } from 'vue'
import { useAuthStore } from '@/stores/auth'
import { useRoute, useRouter } from 'vue-router'

const auth = useAuthStore()
const route = useRoute()
const router = useRouter()

const username = ref('')
const password = ref('')
const loading = ref(false)
const error = ref('')

async function handleLogin() {
  loading.value = true
  error.value = ''
  try {
    await auth.login(username.value, password.value)
    const redirect = (route.query.redirect as string) || '/'
    router.push(redirect)
  } catch (e: any) {
    error.value = e.response?.data?.detail || '登录失败'
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div class="login-page">
    <el-card class="login-card">
      <template #header>
        <h2>舆情分析平台</h2>
      </template>
      <el-form @submit.prevent="handleLogin">
        <el-form-item label="用户名">
          <el-input v-model="username" autocomplete="username" />
        </el-form-item>
        <el-form-item label="密码">
          <el-input v-model="password" type="password" show-password autocomplete="current-password" />
        </el-form-item>
        <el-alert v-if="error" :title="error" type="error" show-icon :closable="false" />
        <el-form-item>
          <el-button type="primary" native-type="submit" :loading="loading" style="width:100%">
            登录
          </el-button>
        </el-form-item>
      </el-form>
    </el-card>
  </div>
</template>
```

### 7.4 API 客户端 (Axios 拦截器)

```typescript
// frontend/src/api/client.ts

import axios from 'axios'
import { useAuthStore } from '@/stores/auth'

export const api = axios.create({
  baseURL: '',
  timeout: 30_000,
})

// 请求拦截——自动附上 JWT
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// 响应拦截——401 时自动跳转登录
api.interceptors.response.use(
  (res) => res,
  (err) => {
    if (err.response?.status === 401) {
      const auth = useAuthStore()
      auth.logout()
    }
    return Promise.reject(err)
  },
)
```

---

## 8. 异常处理

| 场景 | HTTP 状态码 | detail 消息 |
|------|-----------|-----------|
| 用户名不存在或密码错误 | 401 | "用户名或密码错误" |
| 账户被禁用 | 403 | "账户已被禁用" |
| Token 过期 | 401 | "无效的认证令牌" |
| Token 签名无效 | 401 | "无效的认证令牌" |
| 缺少 Authorization 头 | 401 | "未提供认证凭证" |
| 角色权限不足 | 403 | "权限不足" |
| 用户名重复（注册场景） | 409 | "用户名已存在" |

---

## 9. 文件清单

| 文件 | 职责 |
|------|------|
| `backend/app/models/user.py` | User, Role ORM 模型 + user_roles 关联表 |
| `backend/app/core/security.py` | 密码哈希、JWT 签发/验证 |
| `backend/app/core/seed.py` | 种子数据（角色 + 默认管理员） |
| `backend/app/api/auth.py` | 登录/登出/me 端点 |
| `backend/app/api/deps.py` | `get_current_user`、`require_role` 依赖 |
| `frontend/src/pages/LoginPage.vue` | 登录页面 |
| `frontend/src/stores/auth.ts` | 认证状态管理 |
| `frontend/src/api/client.ts` | Axios 实例 + 拦截器 |
| `frontend/src/router/index.ts` | 路由表 + 守卫 |
