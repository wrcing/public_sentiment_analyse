# 舆情分析平台详细设计说明书 — 前端应用

---

## 1. 引言

### 1.1 设计范围

本文档是前端应用的详细设计说明书，依据 HLD 第 3.2.1 节（控制台前端模块）、第 4.3 节（SSE 实时推送）和 SRS 第 3.1-3.8 节的功能需求编写。覆盖：

- 项目结构与路由设计
- Pinia 状态管理（各 Store 设计）
- 布局组件与通用组件
- 6 个页面组件的详细设计（Props / Events / Slots）
- SSE 客户端（连接管理 / 自动重连 / 事件分发）
- API 调用层封装

### 1.2 与 HLD 的对应关系

| HLD 章节 | 本 LLD 覆盖 |
|----------|------------|
| 3.2.1 控制台前端模块 | 全部 6 个页面 + 布局 |
| 3.2.1 页面路由规划 | 路由配置 + 守卫 |
| 4.3 SSE 实时推送 | 前端 SSE 客户端 |

---

## 2. 项目结构

```
frontend/src/
├── main.ts                         # 应用入口
├── App.vue                         # 根组件
├── router/
│   └── index.ts                    # 路由配置 + 导航守卫
├── stores/
│   ├── auth.ts                     # 认证状态 (LLD-02)
│   ├── dashboard.ts                # 控制台状态
│   ├── collection.ts               # 采集任务状态
│   ├── search.ts                   # 检索状态
│   ├── report.ts                   # 报告状态
│   ├── graph.ts                    # 图谱状态
│   ├── status.ts                   # 系统状态
│   └── sse.ts                      # SSE 连接状态
├── api/
│   ├── client.ts                   # Axios 实例 + 拦截器 (LLD-02)
│   ├── dashboard.ts                # 控制台 API 调用
│   ├── collection.ts               # 采集 API 调用
│   ├── search.ts                   # 检索 API 调用
│   ├── reports.ts                  # 报告 API 调用
│   ├── graph.ts                    # 图谱 API 调用
│   ├── status.ts                   # 状态 API 调用
│   ├── config.ts                   # 配置 API 调用
│   └── token.ts                    # Token 消耗 API 调用
├── components/
│   ├── layout/
│   │   ├── MainLayout.vue          # 主布局（侧边栏 + 顶栏 + 内容区）
│   │   ├── Sidebar.vue             # 侧边导航
│   │   └── HeaderBar.vue           # 顶栏（用户信息、登出）
│   ├── common/
│   │   ├── AppStatusBadge.vue      # 应用状态徽标
│   │   ├── StatusCard.vue          # 状态卡片（控制台用）
│   │   ├── LogViewer.vue           # 日志查看器
│   │   └── EmptyState.vue          # 空状态占位
│   ├── dashboard/
│   │   └── AppCard.vue             # 应用卡片（控制台首页）
│   ├── collection/
│   │   ├── TaskCreateDialog.vue    # 创建采集任务对话框
│   │   └── TaskList.vue            # 采集任务列表
│   ├── search/
│   │   ├── SearchForm.vue          # 检索表单
│   │   └── SearchResult.vue        # 检索结果展示
│   ├── report/
│   │   ├── ReportViewer.vue        # 报告查看器（8 章节切换）
│   │   └── ChapterCard.vue         # 单章节卡片
│   └── graph/
│       ├── GraphCanvas.vue         # 图谱画布（使用 ECharts 或 vis-network）
│       └── GraphFilter.vue         # 图谱节点/关系过滤器
├── pages/
│   ├── LoginPage.vue               # 登录页 (LLD-02)
│   ├── DashboardPage.vue           # 控制台首页
│   ├── SearchPage.vue              # 主题检索页
│   ├── GraphViewerPage.vue         # 图谱查看页
│   ├── StatusPage.vue              # 系统状态页
│   └── ConfigPage.vue              # 配置管理页
├── composables/
│   ├── useSSE.ts                   # SSE 连接 composable
│   └── usePolling.ts               # 轮询 composable（兜底方案）
└── utils/
    ├── format.ts                   # 日期/数字格式化
    └── constants.ts                # 常量定义
```

---

## 3. 路由设计

```typescript
// frontend/src/router/index.ts

import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/auth'

const routes = [
  {
    path: '/login',
    name: 'Login',
    component: () => import('@/pages/LoginPage.vue'),
    meta: { requiresAuth: false, title: '登录' },
  },
  {
    path: '/',
    component: () => import('@/components/layout/MainLayout.vue'),
    meta: { requiresAuth: true },
    children: [
      {
        path: '',
        name: 'Dashboard',
        component: () => import('@/pages/DashboardPage.vue'),
        meta: { title: '控制台' },
      },
      {
        path: 'search',
        name: 'Search',
        component: () => import('@/pages/SearchPage.vue'),
        meta: { title: '主题检索', roles: ['operator', 'admin'] },
      },
      {
        path: 'graph-viewer',
        name: 'GraphViewer',
        component: () => import('@/pages/GraphViewerPage.vue'),
        meta: { title: '图谱查看' },
      },
      {
        path: 'graph-viewer/:report_id',
        name: 'GraphViewerByReport',
        component: () => import('@/pages/GraphViewerPage.vue'),
        meta: { title: '图谱查看' },
      },
      {
        path: 'status',
        name: 'Status',
        component: () => import('@/pages/StatusPage.vue'),
        meta: { title: '系统状态', roles: ['admin'] },
      },
      {
        path: 'config',
        name: 'Config',
        component: () => import('@/pages/ConfigPage.vue'),
        meta: { title: '配置管理', roles: ['admin'] },
      },
    ],
  },
]

const router = createRouter({
  history: createWebHistory(),
  routes,
})

router.beforeEach((to, _from, next) => {
  document.title = `${to.meta.title || '舆情分析平台'} - 舆情分析平台`

  const auth = useAuthStore()
  if (to.meta.requiresAuth !== false && !auth.isLoggedIn) {
    next({ name: 'Login', query: { redirect: to.fullPath } })
  } else if (to.name === 'Login' && auth.isLoggedIn) {
    next({ name: 'Dashboard' })
  } else if (to.meta.roles) {
    const required = to.meta.roles as string[]
    const hasRole = auth.user?.roles.some(r => required.includes(r))
    if (!hasRole) {
      next({ name: 'Dashboard' })  // 无权限 → 回首页
      return
    }
    next()
  } else {
    next()
  }
})

export default router
```

### 3.1 路由-角色-页面对照

| 路由 | 页面 | 角色 |
|------|------|------|
| `/login` | 登录页 | 无限制 |
| `/` | 控制台首页 | operator / admin |
| `/search` | 主题检索页 | operator / admin |
| `/graph-viewer` | 图谱查看页(最新) | viewer / operator / admin |
| `/graph-viewer/:report_id` | 图谱查看页(指定) | viewer / operator / admin |
| `/status` | 系统状态页 | admin |
| `/config` | 配置管理页 | admin |

---

## 4. 状态管理 (Pinia Store)

### 4.1 控制台 Store

```typescript
// frontend/src/stores/dashboard.ts

import { defineStore } from 'pinia'
import { ref } from 'vue'
import { dashboardApi } from '@/api/dashboard'

export const useDashboardStore = defineStore('dashboard', () => {
  const appStatuses = ref<Record<string, { status: string; task_count: number; last_output: string | null }>>({})
  const loading = ref(false)

  async function fetchStatus() {
    loading.value = true
    try {
      const res = await dashboardApi.getStatus()
      appStatuses.value = res.data
    } finally {
      loading.value = false
    }
  }

  // SSE 事件更新
  function updateAppStatus(appName: string, status: string) {
    if (appStatuses.value[appName]) {
      appStatuses.value[appName].status = status
    }
  }

  return { appStatuses, loading, fetchStatus, updateAppStatus }
})
```

### 4.2 采集 Store

```typescript
// frontend/src/stores/collection.ts

export const useCollectionStore = defineStore('collection', () => {
  const tasks = ref<CollectionTask[]>([])
  const currentTask = ref<CollectionTask | null>(null)
  const logs = ref<LogEntry[]>([])
  const platforms = ref<PlatformInfo[]>([])
  const loading = ref(false)

  async function fetchTasks(params: { status?: string; page?: number; page_size?: number } = {}) {
    loading.value = true
    try {
      const res = await collectionApi.listTasks(params)
      tasks.value = res.data.items
      return res.data
    } finally {
      loading.value = false
    }
  }

  async function createTask(data: { name: string; source_platforms: string[]; keywords: string[] }) {
    const res = await collectionApi.createTask(data)
    await fetchTasks()
    return res.data
  }

  async function fetchTaskDetail(taskId: string) {
    const res = await collectionApi.getTask(taskId)
    currentTask.value = res.data
    return res.data
  }

  async function fetchTaskLogs(taskId: string, page = 1) {
    const res = await collectionApi.getTaskLogs(taskId, { page, page_size: 50 })
    logs.value = res.data.items
  }

  async function fetchPlatforms() {
    const res = await collectionApi.getPlatforms()
    platforms.value = res.data.platforms
  }

  return { tasks, currentTask, logs, platforms, loading, fetchTasks, createTask, fetchTaskDetail, fetchTaskLogs, fetchPlatforms }
})
```

### 4.3 检索 Store

```typescript
// frontend/src/stores/search.ts

export const useSearchStore = defineStore('search', () => {
  const searches = ref<SearchItem[]>([])
  const currentResult = ref<SearchResult | null>(null)
  const loading = ref(false)
  const searching = ref(false)   // 检索提交中

  async function startSearch(topic: string, sourcePlatforms: string[] = []) {
    searching.value = true
    try {
      const res = await searchApi.start({ topic, source_platforms: sourcePlatforms })
      return res.data.task_id
    } finally {
      searching.value = false
    }
  }

  async function fetchResult(taskId: string) {
    loading.value = true
    try {
      const res = await searchApi.getResult(taskId)
      currentResult.value = res.data
    } finally {
      loading.value = false
    }
  }

  async function fetchHistory(page = 1) {
    loading.value = true
    try {
      const res = await searchApi.list({ page, page_size: 20 })
      searches.value = res.data.items
    } finally {
      loading.value = false
    }
  }

  async function fetchSources(resultId: string) {
    const res = await searchApi.getSources(resultId)
    return res.data.items
  }

  return { searches, currentResult, loading, searching, startSearch, fetchResult, fetchHistory, fetchSources }
})
```

### 4.4 报告 Store

```typescript
// frontend/src/stores/report.ts

export const useReportStore = defineStore('report', () => {
  const reports = ref<ReportItem[]>([])
  const currentReport = ref<ReportDetail | null>(null)
  const loading = ref(false)

  async function generateReport(analysisResultId: string, topic: string) {
    const res = await reportsApi.generate({ analysis_result_id: analysisResultId, topic })
    return res.data
  }

  async function fetchReportList(page = 1) {
    loading.value = true
    try {
      const res = await reportsApi.list({ page, page_size: 20 })
      reports.value = res.data.items
    } finally {
      loading.value = false
    }
  }

  async function fetchReport(reportId: string) {
    loading.value = true
    try {
      const res = await reportsApi.getReport(reportId)
      currentReport.value = res.data
    } finally {
      loading.value = false
    }
  }

  return { reports, currentReport, loading, generateReport, fetchReportList, fetchReport }
})
```

### 4.5 图谱 Store

```typescript
// frontend/src/stores/graph.ts

export const useGraphStore = defineStore('graph', () => {
  const graphData = ref<{ nodes: GraphNode[]; edges: GraphEdge[] } | null>(null)
  const loading = ref(false)

  async function fetchGraph(reportId?: string) {
    loading.value = true
    try {
      const res = reportId
        ? await graphApi.getGraph(reportId)
        : await graphApi.getLatestGraph()
      graphData.value = res.data
    } finally {
      loading.value = false
    }
  }

  async function queryGraph(reportId: string, query: { node_types?: string[]; relation_types?: string[] }) {
    loading.value = true
    try {
      const res = await graphApi.queryGraph(reportId, query)
      graphData.value = res.data
    } finally {
      loading.value = false
    }
  }

  return { graphData, loading, fetchGraph, queryGraph }
})
```

### 4.6 系统状态 Store

```typescript
// frontend/src/stores/status.ts

export const useStatusStore = defineStore('status', () => {
  const systemStatus = ref<SystemStatus | null>(null)
  const errors = ref<ErrorItem[]>([])

  async function fetchStatus() {
    const res = await statusApi.getStatus()
    systemStatus.value = res.data
  }

  async function fetchErrors(page = 1) {
    const res = await statusApi.getErrors({ page, page_size: 20 })
    errors.value = res.data.items
  }

  // SSE 事件更新
  function updateSystemStatus(data: { system_status: string; running_apps: any[] }) {
    if (systemStatus.value) {
      systemStatus.value.system_status = data.system_status
      systemStatus.value.running_apps = data.running_apps
    }
  }

  return { systemStatus, errors, fetchStatus, fetchErrors, updateSystemStatus }
})
```

### 4.7 SSE Store

```typescript
// frontend/src/stores/sse.ts

import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useSSEStore = defineStore('sse', () => {
  const connected = ref(false)
  const lastEvent = ref<{ type: string; data: any } | null>(null)

  function setConnected(val: boolean) { connected.value = val }
  function handleEvent(type: string, data: any) { lastEvent.value = { type, data } }

  return { connected, lastEvent, setConnected, handleEvent }
})
```

---

## 5. 布局组件

### 5.1 MainLayout.vue

```
┌──────────────────────────────────────────────────┐
│  HeaderBar                        [用户名] [登出] │
├──────────┬───────────────────────────────────────┤
│          │                                       │
│ Sidebar  │     <router-view />                   │
│          │     页面内容区                          │
│ 控制台    │                                       │
│ 主题检索  │                                       │
│ 图谱查看  │                                       │
│ 系统状态  │                                       │
│ 配置管理  │                                       │
│          │                                       │
└──────────┴───────────────────────────────────────┘
```

```vue
<!-- frontend/src/components/layout/MainLayout.vue -->

<script setup lang="ts">
import { useAuthStore } from '@/stores/auth'
import { useSSE } from '@/composables/useSSE'
import Sidebar from './Sidebar.vue'
import HeaderBar from './HeaderBar.vue'

const auth = useAuthStore()

// 在登录用户进入主布局后启动 SSE 连接
useSSE()
</script>

<template>
  <el-container class="main-layout">
    <el-header height="56px">
      <HeaderBar
        :username="auth.user?.display_name || auth.user?.username || ''"
        @logout="auth.logout()"
      />
    </el-header>
    <el-container>
      <el-aside width="200px">
        <Sidebar />
      </el-aside>
      <el-main>
        <router-view />
      </el-main>
    </el-container>
  </el-container>
</template>
```

### 5.2 Sidebar.vue

```vue
<script setup lang="ts">
import { useAuthStore } from '@/stores/auth'
import { useRoute } from 'vue-router'

const auth = useAuthStore()
const route = useRoute()

const menuItems = [
  { path: '/',           title: '控制台',   icon: 'Monitor',   roles: ['operator', 'admin'] },
  { path: '/search',     title: '主题检索',  icon: 'Search',    roles: ['operator', 'admin'] },
  { path: '/graph-viewer', title: '图谱查看', icon: 'Share',   roles: ['viewer', 'operator', 'admin'] },
  { path: '/status',     title: '系统状态',  icon: 'Setting',  roles: ['admin'] },
  { path: '/config',     title: '配置管理',  icon: 'Tools',    roles: ['admin'] },
]

const visibleItems = computed(() =>
  menuItems.filter(item => auth.user?.roles.some(r => item.roles.includes(r)))
)
</script>

<template>
  <el-menu
    :default-active="route.path"
    router
    class="sidebar-menu"
  >
    <el-menu-item v-for="item in visibleItems" :key="item.path" :index="item.path">
      <el-icon><component :is="item.icon" /></el-icon>
      <span>{{ item.title }}</span>
    </el-menu-item>
  </el-menu>
</template>
```

### 5.3 HeaderBar.vue

```vue
<script setup lang="ts">
defineProps<{ username: string }>()
defineEmits<{ logout: [] }>()
</script>

<template>
  <div class="header-bar">
    <span class="logo">舆情分析平台</span>
    <el-dropdown>
      <span class="user-info">{{ username }}</span>
      <template #dropdown>
        <el-dropdown-menu>
          <el-dropdown-item @click="$emit('logout')">退出登录</el-dropdown-item>
        </el-dropdown-menu>
      </template>
    </el-dropdown>
  </div>
</template>
```

---

## 6. 页面组件设计

### 6.1 控制台首页 — DashboardPage.vue

```
┌──────────────────────────────────────────────────┐
│  控制台首页                                       │
│                                                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐ │
│  │ 网络采集 │ │ 主题检索 │ │ 报告生成 │ │ 图谱分析│ │
│  │ running │ │  idle   │ │  idle   │ │  idle  │ │
│  │ 任务: 3 │ │ 任务: 0 │ │ 任务: 0 │ │ 任务: 0│ │
│  │ 最近:.. │ │ 最近:.. │ │ 最近:.. │ │ 最近:..│ │
│  └─────────┘ └─────────┘ └─────────┘ └────────┘ │
│                                                  │
│  最近错误                                        │
│  ┌──────────────────────────────────────────┐   │
│  │ [ERROR] 采集服务异常退出 ...             │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useDashboardStore } from '@/stores/dashboard'
import AppCard from '@/components/dashboard/AppCard.vue'

const store = useDashboardStore()

onMounted(() => store.fetchStatus())
</script>

<template>
  <div class="dashboard-page">
    <h2>控制台</h2>
    <el-row :gutter="20">
      <el-col :span="6" v-for="(info, name) in store.appStatuses" :key="name">
        <AppCard
          :app-name="name"
          :status="info.status"
          :task-count="info.task_count"
          :last-output="info.last_output"
        />
      </el-col>
    </el-row>
  </div>
</template>
```

### 6.2 主题检索页 — SearchPage.vue

```
┌──────────────────────────────────────────────────┐
│  主题检索                                         │
│                                                  │
│  检索表单                                        │
│  ┌──────────────────────────────────────────┐   │
│  │ 主题词: [输入框                    ]      │   │
│  │ 来源范围: [B站] [微博] [小红书] [...]     │   │
│  │ [开始检索]                               │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  检索结果                                        │
│  ┌──────────────────────────────────────────┐   │
│  │ 来源摘要 | 观点提炼 | 情绪判断  (Tab 切换) │   │
│  │                                          │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  历史检索列表 (分页)                              │
│  ┌──────────────────────────────────────────┐   │
│  │ 2024-01-01  XX主题  已完成               │   │
│  │ 2024-01-02  YY主题  已完成               │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useSearchStore } from '@/stores/search'
import { useCollectionStore } from '@/stores/collection'
import SearchForm from '@/components/search/SearchForm.vue'
import SearchResult from '@/components/search/SearchResult.vue'

const searchStore = useSearchStore()
const collectionStore = useCollectionStore()
const activeTab = ref<'summary' | 'viewpoints' | 'sentiment'>('summary')
const currentTaskId = ref<string | null>(null)

async function handleSearch(params: { topic: string; source_platforms: string[] }) {
  const taskId = await searchStore.startSearch(params.topic, params.source_platforms)
  currentTaskId.value = taskId
  // 通过 SSE 或轮询等待完成
}

onMounted(() => {
  searchStore.fetchHistory()
  collectionStore.fetchPlatforms()
})
</script>

<template>
  <div class="search-page">
    <h2>主题检索</h2>

    <SearchForm
      :platforms="collectionStore.platforms"
      :loading="searchStore.searching"
      @submit="handleSearch"
    />

    <SearchResult
      v-if="searchStore.currentResult"
      :result="searchStore.currentResult"
      :active-tab="activeTab"
      @update:active-tab="activeTab = $event"
      @view-sources="searchStore.fetchSources"
    />

    <el-divider />

    <h3>历史检索</h3>
    <el-table :data="searchStore.searches" v-loading="searchStore.loading">
      <el-table-column prop="topic" label="主题词" />
      <el-table-column prop="status" label="状态" />
      <el-table-column prop="created_at" label="时间" />
    </el-table>
  </div>
</template>
```

### 6.3 图谱查看页 — GraphViewerPage.vue

```
┌──────────────────────────────────────────────────┐
│  图谱查看                       报告: RPT-2024... │
│                                                  │
│  过滤条件                                        │
│  实体类型: [人物] [事件] [话题] ...               │
│  关系类型: [涉及事件] [表达观点] ...              │
│  [查询]                                         │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │                                          │   │
│  │         图谱可视化画布                     │   │
│  │    (ECharts Graph / vis-network)         │   │
│  │                                          │   │
│  │       ○ ─── ○                           │   │
│  │       │      │                           │   │
│  │       ○ ─── ○ ─── ○                     │   │
│  │                                          │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  节点详情 (悬浮/点击展示)                         │
└──────────────────────────────────────────────────┘
```

```vue
<script setup lang="ts">
import { onMounted, watch } from 'vue'
import { useRoute } from 'vue-router'
import { useGraphStore } from '@/stores/graph'
import GraphCanvas from '@/components/graph/GraphCanvas.vue'
import GraphFilter from '@/components/graph/GraphFilter.vue'

const route = useRoute()
const store = useGraphStore()

onMounted(() => {
  const reportId = route.params.report_id as string | undefined
  store.fetchGraph(reportId)
})

watch(() => route.params.report_id, (newId) => {
  store.fetchGraph(newId as string | undefined)
})

async function handleFilter(filters: { node_types: string[]; relation_types: string[] }) {
  const reportId = route.params.report_id as string
  if (reportId) {
    await store.queryGraph(reportId, filters)
  }
}
</script>

<template>
  <div class="graph-page">
    <h2>图谱查看</h2>

    <GraphFilter @filter="handleFilter" />

    <GraphCanvas
      :nodes="store.graphData?.nodes || []"
      :edges="store.graphData?.edges || []"
      :loading="store.loading"
    />
  </div>
</template>
```

### 6.4 系统状态页 — StatusPage.vue

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useStatusStore } from '@/stores/status'

const store = useStatusStore()

onMounted(() => {
  store.fetchStatus()
  store.fetchErrors()
})
</script>

<template>
  <div class="status-page">
    <h2>系统状态</h2>

    <!-- 整体状态 -->
    <el-card>
      <template #header>整体状态</template>
      <el-tag :type="store.systemStatus?.system_status === 'online' ? 'success' : 'danger'">
        {{ store.systemStatus?.system_status || '未知' }}
      </el-tag>
    </el-card>

    <!-- 应用状态列表 -->
    <el-card>
      <template #header>各应用运行状态</template>
      <el-table :data="store.systemStatus?.running_apps || []">
        <el-table-column prop="name" label="应用名称" />
        <el-table-column prop="status" label="状态">
          <template #default="{ row }">
            <AppStatusBadge :status="row.status" />
          </template>
        </el-table-column>
      </el-table>
    </el-card>

    <!-- 资源概况 -->
    <el-card v-if="store.systemStatus?.resources">
      <template #header>资源概况</template>
      <el-progress :percentage="store.systemStatus.resources.cpu_percent" :text-inside="true" :stroke-width="20">
        CPU
      </el-progress>
      <el-progress :percentage="store.systemStatus.resources.memory_percent" :text-inside="true" :stroke-width="20">
        内存
      </el-progress>
      <el-progress :percentage="store.systemStatus.resources.disk_percent" :text-inside="true" :stroke-width="20">
        磁盘
      </el-progress>
    </el-card>

    <!-- 最近错误 -->
    <el-card>
      <template #header>最近错误信息</template>
      <el-table :data="store.errors">
        <el-table-column prop="module_name" label="模块" width="120" />
        <el-table-column prop="error_message" label="错误信息" />
        <el-table-column prop="time" label="时间" width="180" />
      </el-table>
    </el-card>
  </div>
</template>
```

### 6.5 配置管理页 — ConfigPage.vue

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { configApi } from '@/api/config'
import { ElMessage } from 'element-plus'

const configs = ref<Record<string, any>>({})
const loading = ref(false)

onMounted(async () => {
  loading.value = true
  try {
    const res = await configApi.getConfigs()
    configs.value = res.data.configs
  } finally {
    loading.value = false
  }
})

async function saveConfig(key: string, value: any) {
  try {
    await configApi.updateConfig({ [key]: value })
    ElMessage.success(`配置 ${key} 已更新`)
  } catch {
    ElMessage.error(`配置 ${key} 更新失败`)
  }
}
</script>

<template>
  <div class="config-page">
    <h2>配置管理</h2>
    <el-form v-loading="loading">
      <el-form-item v-for="(value, key) in configs" :key="key" :label="key">
        <template v-if="typeof value === 'boolean'">
          <el-switch :model-value="value" @change="saveConfig(key, $event)" />
        </template>
        <template v-else-if="Array.isArray(value)">
          <el-input :model-value="value.join(', ')" @blur="saveConfig(key, ($event.target as HTMLInputElement).value.split(',').map(s => s.trim()))" />
        </template>
        <template v-else>
          <el-input :model-value="String(value)" @blur="saveConfig(key, ($event.target as HTMLInputElement).value)" />
        </template>
      </el-form-item>
    </el-form>
  </div>
</template>
```

---

## 7. SSE 客户端

### 7.1 useSSE Composable

```typescript
// frontend/src/composables/useSSE.ts

import { onMounted, onUnmounted, ref } from 'vue'
import { useSSEStore } from '@/stores/sse'
import { useDashboardStore } from '@/stores/dashboard'
import { useStatusStore } from '@/stores/status'

const MAX_RETRY_DELAY = 3000  // SRS: ≤ 3s
const INITIAL_RETRY_DELAY = 1000

export function useSSE() {
  const sseStore = useSSEStore()
  const dashboardStore = useDashboardStore()
  const statusStore = useStatusStore()

  let eventSource: EventSource | null = null
  let retryDelay = INITIAL_RETRY_DELAY
  let retryTimer: ReturnType<typeof setTimeout> | null = null

  function connect() {
    eventSource = new EventSource('/sse')

    eventSource.onopen = () => {
      sseStore.setConnected(true)
      retryDelay = INITIAL_RETRY_DELAY
    }

    eventSource.onerror = () => {
      sseStore.setConnected(false)
      eventSource?.close()
      scheduleReconnect()
    }

    // 注册各消息类型的处理器
    const handlers: Record<string, (data: any) => void> = {
      app_status(data) {
        dashboardStore.updateAppStatus(data.app_name, data.status)
        sseStore.handleEvent('app_status', data)
      },
      app_output(data) {
        sseStore.handleEvent('app_output', data)
      },
      forum_log(data) {
        sseStore.handleEvent('forum_log', data)
      },
      system_status(data) {
        statusStore.updateSystemStatus(data)
        sseStore.handleEvent('system_status', data)
      },
      graph_ready(data) {
        sseStore.handleEvent('graph_ready', data)
      },
      error(data) {
        sseStore.handleEvent('error', data)
        // 可触发全局错误提示
      },
    }

    // 为每种消息类型注册 event listener
    for (const [eventType, handler] of Object.entries(handlers)) {
      eventSource.addEventListener(eventType, (event: MessageEvent) => {
        try {
          const data = JSON.parse(event.data)
          handler(data)
        } catch (e) {
          console.error(`SSE 事件解析失败 [${eventType}]:`, e)
        }
      })
    }
  }

  function scheduleReconnect() {
    retryTimer = setTimeout(() => {
      connect()
      retryDelay = Math.min(retryDelay * 2, MAX_RETRY_DELAY)
    }, retryDelay)
  }

  function disconnect() {
    eventSource?.close()
    eventSource = null
    if (retryTimer) {
      clearTimeout(retryTimer)
      retryTimer = null
    }
  }

  onMounted(connect)
  onUnmounted(disconnect)

  return { disconnect, reconnect: connect }
}
```

### 7.2 各消息类型的 UI 消费

| 消息类型 | 消费的 Store | UI 更新方式 |
|---------|-------------|-----------|
| `app_status` | `dashboardStore` | 控制台首页应用卡片状态徽标实时变色 |
| `app_output` | `searchStore` | 检索结果区增量更新 |
| `forum_log` | `collectionStore` | 日志查看器追加新行 |
| `system_status` | `statusStore` | 状态页实时刷新 |
| `graph_ready` | `graphStore` | 图谱页自动加载新图谱 |
| `error` | 全局提示 | `ElMessage.error()` 弹出错误通知 |

---

## 8. API 调用层

```typescript
// frontend/src/api/dashboard.ts

import { api } from './client'

export const dashboardApi = {
  getStatus:  ()          => api.get('/api/dashboard/status'),
  getSummary: ()          => api.get('/api/dashboard/summary'),
}

// frontend/src/api/collection.ts

export const collectionApi = {
  createTask:  (data: any) => api.post('/api/collection/tasks', data),
  listTasks:   (params?: any) => api.get('/api/collection/tasks', { params }),
  getTask:     (id: string) => api.get(`/api/collection/tasks/${id}`),
  getTaskLogs: (id: string, params?: any) => api.get(`/api/collection/tasks/${id}/logs`, { params }),
  getPlatforms: () => api.get('/api/collection/platforms'),
}

// frontend/src/api/search.ts

export const searchApi = {
  start:      (data: any) => api.post('/api/search', data),
  getResult:  (id: string) => api.get(`/api/search/${id}`),
  list:       (params?: any) => api.get('/api/search', { params }),
  getSources: (resultId: string) => api.get(`/api/search/sources/${resultId}`),
}

// frontend/src/api/reports.ts

export const reportsApi = {
  generate:     (data: any) => api.post('/api/reports', data),
  list:         (params?: any) => api.get('/api/reports', { params }),
  getReport:    (reportId: string) => api.get(`/api/reports/${reportId}`),
  getEvidence:  (reportId: string) => api.get(`/api/reports/${reportId}/evidence`),
}

// frontend/src/api/graph.ts

export const graphApi = {
  getGraph:       (reportId: string) => api.get(`/api/graph/${reportId}`),
  queryGraph:     (reportId: string, data: any) => api.post(`/api/graph/${reportId}/query`, data),
  getLatestGraph: () => api.get('/api/graph/latest'),
}

// frontend/src/api/status.ts

export const statusApi = {
  getStatus: ()          => api.get('/api/status'),
  getErrors: (params?: any) => api.get('/api/status/errors', { params }),
}

// frontend/src/api/config.ts

export const configApi = {
  getConfigs:   ()          => api.get('/api/config'),
  updateConfig: (data: any) => api.put('/api/config', data),
}

// frontend/src/api/token.ts

export const tokenApi = {
  getUsage: (params?: any) => api.get('/api/token-usage', { params }),
  getQuota: ()             => api.get('/api/token-usage/quota'),
  setQuota: (data: any)    => api.put('/api/token-usage/quota', data),
}
```

---

## 9. 通用组件

### 9.1 AppStatusBadge.vue

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{ status: string }>()

const tagType = computed(() => {
  const map: Record<string, string> = {
    running: 'success',
    online: 'success',
    completed: 'success',
    idle: 'info',
    stopped: 'info',
    pending: 'warning',
    starting: 'warning',
    stopping: 'warning',
    failed: 'danger',
    offline: 'danger',
    error: 'danger',
  }
  return map[props.status] || 'info'
})
</script>

<template>
  <el-tag :type="tagType" size="small">{{ status }}</el-tag>
</template>
```

### 9.2 LogViewer.vue

```vue
<script setup lang="ts">
defineProps<{ logs: { level: string; message: string; created_at: string }[] }>()
</script>

<template>
  <div class="log-viewer">
    <div
      v-for="(log, idx) in logs"
      :key="idx"
      :class="['log-line', `log-${log.level.toLowerCase()}`]"
    >
      <span class="log-time">{{ log.created_at }}</span>
      <span class="log-level">[{{ log.level }}]</span>
      <span class="log-msg">{{ log.message }}</span>
    </div>
    <el-empty v-if="!logs.length" description="暂无日志" />
  </div>
</template>
```

---

## 10. 图谱可视化选型

首版推荐使用 **ECharts**（`echarts` npm 包）实现图谱画布。

```
npm install echarts
```

图类型：`type: 'graph'`，支持力导向布局，节点可拖拽，点击事件可获取详情。

```typescript
// GraphCanvas.vue 核心逻辑
import * as echarts from 'echarts'

// 将 graphData 转换为 ECharts 格式
function toEChartsOption(nodes: GraphNode[], edges: GraphEdge[]) {
  return {
    series: [{
      type: 'graph',
      layout: 'force',
      force: { repulsion: 200, edgeLength: 150 },
      roam: true,
      draggable: true,
      data: nodes.map(n => ({
        id: n.id,
        name: n.label,
        category: n.type,
      })),
      links: edges.map(e => ({
        source: e.source,
        target: e.target,
        label: { show: true, formatter: e.relation },
      })),
      categories: [...ENTITY_TYPES.map(t => ({ name: t }))],
    }],
  }
}
```

---

## 11. 文件清单

| 文件 | 职责 |
|------|------|
| `frontend/src/App.vue` | 根组件 |
| `frontend/src/main.ts` | 入口（注册 Pinia/Router/ElementPlus） |
| `frontend/src/router/index.ts` | 路由表 + 导航守卫 |
| `frontend/src/stores/auth.ts` | 认证状态 |
| `frontend/src/stores/dashboard.ts` | 控制台状态 |
| `frontend/src/stores/collection.ts` | 采集任务状态 |
| `frontend/src/stores/search.ts` | 检索状态 |
| `frontend/src/stores/report.ts` | 报告状态 |
| `frontend/src/stores/graph.ts` | 图谱状态 |
| `frontend/src/stores/status.ts` | 系统状态 |
| `frontend/src/stores/sse.ts` | SSE 连接状态 |
| `frontend/src/api/client.ts` | Axios 实例 + 拦截器 |
| `frontend/src/api/dashboard.ts` | 控制台 API |
| `frontend/src/api/collection.ts` | 采集 API |
| `frontend/src/api/search.ts` | 检索 API |
| `frontend/src/api/reports.ts` | 报告 API |
| `frontend/src/api/graph.ts` | 图谱 API |
| `frontend/src/api/status.ts` | 状态 API |
| `frontend/src/api/config.ts` | 配置 API |
| `frontend/src/api/token.ts` | Token API |
| `frontend/src/composables/useSSE.ts` | SSE 连接管理 |
| `frontend/src/components/layout/MainLayout.vue` | 主布局 |
| `frontend/src/components/layout/Sidebar.vue` | 侧边栏导航 |
| `frontend/src/components/layout/HeaderBar.vue` | 顶栏 |
| `frontend/src/components/common/AppStatusBadge.vue` | 状态徽标 |
| `frontend/src/components/common/LogViewer.vue` | 日志查看器 |
| `frontend/src/components/dashboard/AppCard.vue` | 控制台应用卡片 |
| `frontend/src/components/collection/TaskCreateDialog.vue` | 创建采集任务 |
| `frontend/src/components/collection/TaskList.vue` | 采集任务列表 |
| `frontend/src/components/search/SearchForm.vue` | 检索表单 |
| `frontend/src/components/search/SearchResult.vue` | 检索结果 |
| `frontend/src/components/report/ReportViewer.vue` | 报告查看器 |
| `frontend/src/components/report/ChapterCard.vue` | 章节卡片 |
| `frontend/src/components/graph/GraphCanvas.vue` | 图谱画布 |
| `frontend/src/components/graph/GraphFilter.vue` | 图谱过滤器 |
| `frontend/src/pages/LoginPage.vue` | 登录页 |
| `frontend/src/pages/DashboardPage.vue` | 控制台首页 |
| `frontend/src/pages/SearchPage.vue` | 主题检索页 |
| `frontend/src/pages/GraphViewerPage.vue` | 图谱查看页 |
| `frontend/src/pages/StatusPage.vue` | 系统状态页 |
| `frontend/src/pages/ConfigPage.vue` | 配置管理页 |
