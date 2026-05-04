# FRONT

当然可以！将我们目前构建的 Vue 3 权限系统前端逻辑进行详细梳理，并总结成一个 Markdown 文档，方便您理解和参考。

***

# 🚀 Vue 3 + Spring Boot 权限系统前端逻辑深度解析

本文档详细描述了基于 Vue 3, Pinia, Vue Router 和 Element Plus 构建的权限管理系统前端的**核心逻辑与模块职责**。

## I. 核心技术栈概览

| 模块 | 技术栈 | 职责 |
| :--- | :--- | :--- |
| **框架** | Vue 3 (Composition API) | 构建 UI 界面。 |
| **UI 组件** | Element Plus | 提供美观、专业的后台组件。 |
| **状态管理** | Pinia | 集中管理用户状态 (`Token`, `Permissions`)。 |
| **路由** | Vue Router 4 | 实现前端路由和动态菜单加载。 |
| **通信** | Axios | 负责与 Spring Boot 后端进行 HTTP 通信。 |
| **开发环境** | Vite Proxy | 解决开发时的**跨域问题 (CORS)**。 |

***

## II. 核心流程：动态路由与权限校验

权限系统的核心在于用户登录后，前端如何动态地加载用户有权访问的菜单和页面。

### 流程图

用户访问受保护页面 $ \xrightarrow{1} $ 路由守卫 (`permission.ts`) 拦截 $ \xrightarrow{2} $ 检查 Token/权限 $ \xrightarrow{3} $ 生成路由 $ \xrightarrow{4} $ 页面跳转

### 步骤详解

1. **用户尝试访问：** 用户在浏览器输入 URL (例如 `/dashboard`)。
2. **路由守卫介入 (**`src/permission.ts`**)：**
   * **❓**\*\* 检查 Token：\*\* `if (!userStore.token)` 检查 Pinia 中是否有 Token。如果没有，强制重定向到 `/login`，并将目标路径记在 `?redirect=...` 中。
   * **✅**\*\* Token 存在：\*\* 检查用户是否已加载权限列表 (`if (userStore.permissions.length === 0)`)。如果尚未加载：
     * 调用后端 API 获取当前用户的 **权限代码列表** (`['user:add', 'menu:list']`)。
3. **权限路由生成 (**`src/store/permission.ts`**)：**
   * 守卫拿到权限列表后，调用 `permissionStore.generateRoutes(permissions)`。
   * 该 Store 将前端预先定义好的 **所有动态路由映射表**，通过 `filterAsyncRoutes` 函数进行递归过滤。
   * 过滤逻辑：`if (permissions.includes(route.meta.permissionCode))`
4. **动态添加路由：**
   * 将过滤后得到的 **用户专属路由列表** (`accessRoutes`)，通过 `router.addRoute(route)` 动态添加到 Vue Router 实例中。
5. **最终放行：**
   * 路由添加完成后，调用 `next({ ...to, replace: true })`，确保路由系统重新计算，跳转到用户最初请求的页面 (`/dashboard`)。

***

## III. 模块职责划分 (文件级)

| 文件/模块 | 角色定位 | 核心职责 |
| :--- | :--- | :--- |
| `src/main.ts` | 入口配置 | 注册 Pinia, Router, Element Plus, 并引入路由守卫 (`./permission.ts`)。 |
| `src/router/index.ts` | 静态路由定义 | 存放 `/login`, `/404`, `/dashboard` 等所有用户都可访问的基础路由。 |
| `src/permission.ts` | **路由守卫 (Gatekeeper)** | 全局拦截请求，判断登录状态，触发权限加载流程。 |
| `src/store/user.ts` | **用户身份管理** | 存储并管理 `token` 和 `permissions` 列表，处理 `login/logout` 业务。 |
| `src/store/permission.ts` | **路由构建器** | 接收权限列表，过滤并生成用户可访问的最终路由表。 |
| `src/utils/request.ts` | **HTTP 请求工具** | 封装 Axios，实现请求头自动添加 Token，统一响应错误处理（401, 500 等）。 |
| `vite.config.ts` | **开发环境配置** | 通过配置 `server.proxy`，将前端 `5173/api/login` 转发至后端 `13690/login`，解决跨域问题。 |

***

## IV. 关键逻辑点深度解析

### 1. Token 认证与数据流

| 动作 | 发生位置 | 关键实现 |
| :--- | :--- | :--- |
| **登录请求** | `src/store/user.ts` $ \rightarrow $ `src/api/login.ts` | 调用封装的 `request.post('/login', data)`。 |
| **Token 获取** | `src/store/user.ts` | 根据您的后端返回结构，成功获取后执行：`this.token = res.data`。 |
| **Token 携带** | `src/utils/request.ts` (请求拦截器) | 在每个非登录请求中，自动将 `userStore.token` 加入请求头 (`config.headers['Authorization'] = this.token`)。 |

### 2. 权限控制的两个级别

前端的权限控制通常分为两个级别，以确保安全性和用户体验：

#### A. 路由级权限 (Page-Level)

* **目的:** 阻止用户通过直接输入 URL 访问无权查看的页面。
* **实现:** `src/store/permission.ts` 中的 `filterAsyncRoutes` 函数。它在内存中过滤路由表，然后只将有权访问的路由通过 `router.addRoute` 注册到系统中。无权访问的路由在前端路由表中根本**不存在**。

#### B. 元素级权限 (Element-Level)

* **目的:** 控制页面内部按钮（如“新增”、“删除”）、输入框或组件的显示/隐藏。
* **实现:** 使用 `v-permission` 自定义指令（我们之前讨论过）。
  * **用法:** `<el-button v-permission="'user:add'">新增</el-button>`
  * **逻辑:** 该指令在组件挂载时，会查询 `userStore.permissions` 数组是否包含 `'user:add'` 权限码。如果没有，则直接将该按钮从 DOM 中移除。

***

您希望我下一步为您提供\*\*`v-permission` 自定义指令\*\*的代码实现吗？


> 更新: 2025-12-10 16:19:02  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/nsn3bzm7gnma3gvd>