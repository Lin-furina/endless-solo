---
title: HeartEcho 前端：先把契约、请求和登录态搭稳
published: 2026-10-30
description: 三人小组作业里我负责的前端基础设施——类型契约、请求层封装和路由守卫是怎么设计的，以及踩过的几个坑。
tags: [Vue3, TypeScript, 前端工程, 项目复盘]
category: 项目复盘
pinned: false
draft: false
---

HeartEcho（心回响）是一个 AI 情感陪伴 Web 应用，三人团队四周做完。核心想法是做一个"像真人一样"的 AI 伴侣——它记得你说过的话，有情绪，会主动找你聊天，还能自个儿发朋友圈。

我负责前端全链路。但真开始写页面之前，我先花了一天时间摸索：把环境装起来，再把骨架搭好——**定类型契约、封请求层、配路由守卫**。事后看，这一天花得最值。

这篇就讲这三块，以及各自踩过的坑。

## 一、先定契约，再写页面

前后端并行开发时最怕的不是需求变更，是**字段名对不上**——你写 `username`，后端写 `user_name`，联调时才发现，改一圈。

所以第一件事是把后端的响应结构原样镜像到 TypeScript 里：

```ts
/** 后端统一响应体，对应 Go 侧 `Response[T]` */
export interface ApiResponse<T = unknown> {
  code: number
  message: string
  data: T
  timestamp: number
}
```

就这么简单一个泛型，但它定了三件事：

1. **所有接口的返回都是这个形状**，前端不用为每个接口猜结构
2. `data` 是泛型，各接口自己填自己的业务类型
3. 失败时 `data` 是 `null`，所以**不要直接访问**，统一交给请求层解包

文件顶部我写了一句狠话：

> 本文件是所有接口响应的类型源头，请求层与业务层统一从这里 import，不允许在别处重复定义响应体结构。

### 错误码也镜像一份

后端遵循「一 code 一 msg」：一个错误码只对应一句固定文案。前端跟着建了一张对照表：

```ts
export const ErrorCode = {
  Success: 200,
  ErrUnauthorized: 4010,
  ErrTokenInvalid: 4011,
  ErrTokenExpired: 4012,
  ErrRefreshInvalid: 4014,
  // ...
} as const
```

两个细节：

- **常量名和后端逐字一致**，含 `Err` 前缀。这样两边比对的时候，可以直接拿文件名 diff，不用在脑子里做翻译。
- `as const` 不是为了好看，是为了让值变成字面量类型，后面 `ErrorCodeValue` 就能做穷尽检查——漏写一个 case 编译器会告诉你。

## 二、请求层：把麻烦全关进一个文件

如果不在请求层做封装，每个接口调用大概长这样：

```ts
const res = await axios.get('/personas')
if (res.data.code !== 200) { ElMessage.error(res.data.message); return }
const list = res.data.data
```

每个调用重复一遍，而且 token 过期、刷新、重试全得自己处理。所以我封了一个 `http<T>()`，**调用方拿到的直接是业务数据**：

```ts
async function http<T>(config: AxiosRequestConfig): Promise<T> {
  const response = await instance.request<ApiResponse<T>>(config)
  return response.data.data
}
```

调用方写 `const list = await request.get<Persona[]>('/personas')`，`code` 和 `data` 的拆解在底层做完了。

真正麻烦的是 token 刷新。这里踩了三个坑。

### 坑一：刷新请求自己触发刷新，无限循环

最直觉的写法是在响应拦截器里判断：如果 `code === 4012`（token 过期），就调刷新接口，成功后重放原请求。

问题在于——**刷新接口本身也是用这个 axios 实例发的**。如果刷新时 token 也过期了（或者后端判定异常），它会再次返回 4012，于是又触发刷新……无限循环。

解法是开一个**不挂任何拦截器的"裸"实例**，专门用来刷新：

```ts
/** 刷新令牌专用的「裸」实例：不挂任何拦截器。
 *  这样刷新请求永远不会被自己的拦截器再处理一遍，从根上杜绝死循环。 */
const refreshClient: AxiosInstance = axios.create({
  baseURL, timeout: 15000,
  headers: { 'Content-Type': 'application/json' },
})
```

这比在拦截器里加 `if (url === '/auth/refresh') return` 之类的判断干净得多——**从结构上就不可能出现循环**，而不是靠条件绕开。

### 坑二：五个请求同时过期，刷新五次

页面初始化时经常并发好几个请求。如果 token 恰好过期，五个请求会同时收到 4012，然后各自发起一次刷新——五次刷新请求，四次是浪费，而且可能互相覆盖 token。

用一个并发锁解决：

```ts
let refreshPromise: Promise<string> | null = null

export async function refreshAccessToken(): Promise<string> {
  // 已经有人在刷新 —— 复用它的结果，排队等就行
  if (refreshPromise !== null) {
    return refreshPromise
  }

  const pending = doRefresh()
  refreshPromise = pending

  try {
    return await pending
  } finally {
    // 无论成功还是失败，都要把锁摘掉；否则后面的刷新会被永久卡住
    refreshPromise = null
  }
}
```

`finally` 那一句很关键。如果只写 `try` 不写 `finally`，一次刷新失败之后 `refreshPromise` 会永远不为 null，后面所有请求都卡在"等一个已经死掉的 Promise"上。

### 坑三：重放不能无限重放

token 刷新成功后要重放原请求，但万一重放还是 4012（比如后端刚把这个 token 拉黑了），会再次触发刷新、再次重放。给请求配置打个标记，只重放一次：

```ts
type RetryConfig = InternalAxiosRequestConfig & { _retry?: boolean }

if (code === ErrorCode.ErrTokenExpired && originalConfig._retry !== true) {
  originalConfig._retry = true
  try {
    await refreshAccessToken()
    return await instance.request(originalConfig)
  } catch {
    forceLogout()
  }
}
```

### 顺带一提：为什么 store 要在函数体里获取

```ts
instance.interceptors.request.use((config) => {
  // 延迟获取 store：Pinia 的 store 只能在 app 挂载之后使用。
  // 写在拦截器函数体里，既保证时机正确，也避免与 auth.ts 形成模块级循环依赖。
  const auth = useAuthStore()
  // ...
})
```

如果在文件顶层 `const auth = useAuthStore()`，一是 Pinia 还没初始化会报错，二是 `request.ts` 和 `auth.ts` 会互相 import 形成循环依赖——表现是某个模块莫名其妙变成 `undefined`，排查起来很痛苦。

## 三、路由守卫：缺省就当需要登录

守卫逻辑很短，但有一处值得说：

```ts
router.beforeEach((to) => {
  const authStore = useAuthStore()

  // 缺省视为需要登录：新增页面漏写标记时宁可多拦一次，也不能少一道防线
  if (to.meta.requiresAuth !== false && !authStore.isLogin) {
    return { name: 'Login', query: { redirect: to.fullPath } }
  }

  // 已登录还去登录页，直接送回聊天页
  if (authStore.isLogin && (to.name === 'Login' || to.name === 'Register')) {
    return { name: 'Chat' }
  }

  return true
})
```

注意判断条件写的是 `!== false` 而不是 `=== true`。

差别在于：**加了新页面忘记写 `requiresAuth: true` 时**，前者会拦住（安全），后者会放行（危险）。凡是涉及权限的判断，"缺省拒绝"永远比"缺省允许"好——漏拦一次是数据泄露，多拦一次只是多点一次登录。

## 回头看

这三个模块加起来不到 400 行，但它们决定了后面所有页面写起来顺不顺：

- **契约文件**让联调时不用反复确认字段名，出问题直接 diff 两边定义
- **请求层**让业务代码里看不到 `if (res.code !== 200)`
- **路由守卫**让"忘记加鉴权"这种低级错误不会发生

如果重来一次，我可能会把错误码表做成从后端生成，而不是手写镜像——手写总有漏更新的时候。不过在四周的周期里，手写一份、在注释里标明"权威来源是 errcode.go，改前先确认后端"，已经是性价比最高的方案了。
