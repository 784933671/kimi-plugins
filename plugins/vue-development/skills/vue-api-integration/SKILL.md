---
name: vue-api-integration
description: Vue 3 + JavaScript 接口接入规范。用于封装或使用 Axios/Fetch 请求、处理 loading/error/empty 状态、token 鉴权、接口异常捕获、请求取消、列表分页、详情加载、提交保存和接口字段适配。
---

# Vue API Integration

## 工作原则

- 优先复用项目已有 request 实例、接口模块、状态管理和消息提示工具。
- 页面组件只编排状态和交互，不在组件里散落原始 URL、鉴权头和响应码判断。
- 所有异步请求必须有异常捕获和兜底状态。
- 接口返回结构在接口层或适配层统一处理，避免模板里到处写兼容判断。

## 接口分层

- `api/`：只负责请求路径、方法、参数和响应返回。
- `utils/request`：负责 baseURL、token、拦截器、通用错误。
- `composables/`：封装列表、详情、提交等可复用业务状态。
- `views/`：负责调用、展示和交互闭环。

## 页面请求状态

每个页面级请求至少维护：

- `loading`：首次加载、刷新或提交中。
- `error`：请求失败原因。
- `empty`：请求成功但无数据。
- `data/list/detail`：实际渲染数据。

```js
import { ref } from 'vue'

export function useList(fetcher) {
  const loading = ref(false)
  const error = ref('')
  const list = ref([])

  async function load(params = {}) {
    try {
      loading.value = true
      error.value = ''
      const res = await fetcher(params)
      list.value = Array.isArray(res?.list) ? res.list : []
    } catch (err) {
      error.value = err?.message || '数据加载失败'
      list.value = []
    } finally {
      loading.value = false
    }
  }

  return { loading, error, list, load }
}
```

## 请求参数

- 提交前清理空字符串、空数组、空对象和无效日期。
- 分页参数统一命名，和项目现有接口约定保持一致。
- 日期范围转换在请求前完成，不把组件库的日期对象直接传给接口。
- 不要把整个表单对象直接传入接口；先构造 payload，避免多传 UI 临时字段。

## 错误处理

- 401/登录过期：交给统一请求层处理。
- 403/无权限：给出明确提示或进入无权限状态。
- 业务错误：显示后端 message；没有 message 时使用兜底文案。
- 字段错误：能映射到表单字段时回填字段；不能映射时全局提示。
- 网络错误：提示稍后重试，并保留用户输入。

## 提交请求

- 提交前完成本地校验。
- 使用 `submitting` 防重复提交。
- 成功后按业务刷新列表、关闭弹窗、返回上一页或跳转详情。
- 失败时不清空表单，不关闭弹窗。

```js
async function save() {
  if (submitting.value) return

  try {
    submitting.value = true
    await saveApi(buildPayload())
    await refreshAfterSave()
  } catch (err) {
    showError(err?.message || '保存失败，请稍后重试')
  } finally {
    submitting.value = false
  }
}
```

## 请求取消和竞态

- 搜索、筛选、路由切换可能触发竞态时，使用 AbortController 或项目已有取消机制。
- 只允许最后一次请求结果落到页面状态。
- 组件卸载后不要继续写入状态。

Vue 3.5+ 提供原生副作用清理机制。在 `watch`/`watchEffect` 中用 `onWatcherCleanup` 注册取消回调，当侦听器重新运行或组件卸载时自动调用：

```js
import { ref, watch, onWatcherCleanup } from 'vue'

const keyword = ref('')
const list = ref([])

// watch 回调的第三个参数是清理函数注册器（Vue 3.5+）
watch(keyword, (value, oldValue, onCleanup) => {
  const controller = new AbortController()

  fetchList({ keyword: value, signal: controller.signal })
    .then((res) => {
      list.value = res?.list ?? []
    })
    .catch(() => {})

  // 下次 watch 触发或组件卸载时自动取消上一次请求
  onCleanup(() => controller.abort())
})
```

`watchEffect` 中用导入的 `onWatcherCleanup` 效果相同：

```js
import { ref, watchEffect, onWatcherCleanup } from 'vue'

const keyword = ref('')

watchEffect(() => {
  const controller = new AbortController()
  fetchList({ keyword: keyword.value, signal: controller.signal })
    .then(/* ... */)
    .catch(() => {})

  // watchEffect 重新执行或组件卸载时自动调用
  onWatcherCleanup(() => controller.abort())
})
```

对于非侦听器场景的副作用（如手动启动的轮询、SSE），用 `onScopeDispose` 在作用域销毁时清理：

```js
import { onScopeDispose } from 'vue'

const controller = new AbortController()
startPolling(controller.signal)

onScopeDispose(() => controller.abort())
```

`useList` 也应在组件卸载时取消进行中的请求，避免写入已卸载组件的状态：

```js
import { ref, onScopeDispose } from 'vue'

export function useList(fetcher) {
  const loading = ref(false)
  const error = ref('')
  const list = ref([])
  let currentController = null

  async function load(params = {}) {
    currentController?.abort()
    const controller = new AbortController()
    currentController = controller

    try {
      loading.value = true
      error.value = ''
      const res = await fetcher({ ...params, signal: controller.signal })
      list.value = Array.isArray(res?.list) ? res.list : []
    } catch (err) {
      if (err?.name !== 'AbortError') {
        error.value = err?.message || '数据加载失败'
        list.value = []
      }
    } finally {
      loading.value = false
    }
  }

  onScopeDispose(() => currentController?.abort())

  return { loading, error, list, load }
}
```

## 接口字段适配

后端字段命名（snake_case、嵌套结构）与前端模型（camelCase、扁平结构）不一致时，在接口层做适配，不要让模板和组件到处写兼容判断。

```js
// api/user.js —— 只关心请求与字段映射
export async function fetchUser(id) {
  const res = await request.get(`/users/${id}`)
  return {
    id: res.user_id,
    name: res.user_name,
    avatar: res.profile?.avatar_url ?? '',
    createdAt: res.created_at,
  }
}
```

列表场景统一在 composable 里适配，组件只消费规整数据：

```js
// composables/useUserList.js
import { useList } from './useList'

const adapter = (raw) => ({
  id: raw.user_id,
  name: raw.user_name,
  status: raw.status_code,
})

export function useUserList() {
  const { loading, error, list, load } = useList(async (params) => {
    const res = await fetchUserListApi(params)
    return {
      list: (res?.items ?? []).map(adapter),
      total: res?.total ?? 0,
    }
  })

  return { loading, error, list, load }
}
```

当字段映射规则会复用时，把 adapter 抽成独立函数；只在单处使用时内联即可。适配完成后，模板和组件不再出现 `item.user_name` 这类后端字段。

## 完成检查

- 组件没有散落重复的请求异常处理。
- loading、error、empty、success 状态完整。
- 接口失败不会导致页面空白、按钮卡死或用户输入丢失。
- 请求参数和响应数据已在边界处适配。
