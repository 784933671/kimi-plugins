---
name: frontend-debugging
description: 前端问题定位与修复流程。用于 Vue、Element Plus、Vant、浏览器、H5 真机、接口、跨域、Source Map、白屏、样式错位、移动端兼容、线上异常和性能卡顿的调试分析。
---

# Frontend Debugging

## 工作原则

- 先复现，再定位；先证据，再修改。
- 每次只改变一个变量，避免同时改接口、样式和状态逻辑。
- 优先使用浏览器 DevTools、Vue Devtools、Network、Console、Performance 的直接证据。
- 修复后必须回到原复现路径验证。

## 定位流程

1. 明确现象：页面、设备、账号、路由、操作步骤、期望结果、实际结果。
2. 缩小范围：判断是接口、状态、组件库、样式、路由、构建还是环境问题。
3. 找证据：查看 Console、Network、Vue 组件状态、DOM、Computed 样式。
4. 做最小修复：只改根因相关代码。
5. 回归验证：覆盖成功、失败、空数据、刷新、返回、重复操作。

## 接口问题

- Network 里确认 URL、method、params、payload、headers、status、response。
- 区分浏览器没发出、请求被拦截、服务端报错、前端解析错误。
- CORS 问题看预检请求、响应头和代理配置，不在业务代码里硬绕。
- 401/403 优先查 token、权限、路由守卫和请求拦截器。
- 接口失败时确认页面是否恢复 loading、保留输入并显示兜底错误。

## Vue 状态问题

- 用 Vue Devtools 查看 props、emits、refs、computed、Pinia store。
- 检查是否直接修改 props、替换 reactive 对象、watch 依赖写错。
- 检查列表 key 是否稳定，避免复用错误 DOM。
- 检查异步请求返回顺序，避免旧请求覆盖新状态。
- 路由参数变化时确认是否有 watch 或 beforeRouteUpdate 处理。

### Vue DevTools 形态与版本

Vue DevTools 有三种形态，按场景选用：

- **浏览器扩展**：最常用，Chrome/Edge/Firefox 应用商店安装。注意 v7+ 仅支持 Vue 3；Vue 2 项目最高只能用 v6（v6 同时兼容 Vue 2/3，是过渡大版本）。
- **vite-plugin-vue-devtools**：Vite 项目可装 `vite-plugin-vue-devtools`，开发时浏览器里自动注入独立面板，功能比扩展更全（组件树、路由、Pinia、Timeline）。
- **Standalone App**：独立 Electron 应用，用于调试移动端 iframe、嵌入式 WebView 等扩展无法覆盖的场景。

如果扩展面板不显示组件树，先确认页面是否真的挂载了 Vue 实例、扩展版本是否匹配 Vue 版本。

## HMR 调试

HMR 不更新、更新后状态错乱、循环热更新是 Vue + Vite 最高频的调试场景。

### HMR 不生效

- 检查组件是否使用了 `<script setup>`：纯 `<script setup>` 的 SFC 默认支持 HMR；普通 `<script>` 中手动 `import` 旧实例会破坏 HMR 边界。
- 确认 `vite.config.js` 没有把相关文件排除在 `optimizeDeps` 之外，或被 `ssr.noExternal` 误配置。
- 检查是否有全局副作用（如直接修改 `document.body`）阻止 Vite 判断可热替换边界。
- 浏览器扩展（尤其是广告拦截器、缓存插件）可能拦截 WebSocket，关闭后重试。

### HMR 后状态错乱

- HMR 保留组件状态时，`ref`/`reactive` 旧值可能与新模板/逻辑不匹配：在 DevTools Components 面板里手动 reset 组件状态复现。
- Pinia store 默认跨 HMR 保留，store 修改后 HMR 不会重置；用 store 的 `$reset()` 或刷新页面验证。
- `onMounted`/`onUnmounted` 中的全局副作用（事件监听、定时器）在 HMR 时不会重新执行，易造成监听器堆积。

### 循环热更新

- 某个文件改动后触发链式更新甚至死循环，通常是该模块在导入时执行了写文件、读 `import.meta.hot` 之外的副作用。
- 查看 Vite 终端日志中的 HMR 传播链路，定位循环起点。

## 内存泄漏与性能

### 内存泄漏排查

- 用 Chrome DevTools **Memory** 面板：拍 Heap snapshot → 操作页面 → 再拍一次，用 Comparison 视图找增量对象。
- 重点排查 **Detached DOM 节点**：组件卸载后 DOM 被闭包引用，会显示为黄色 detached 节点。
- Vue 专属泄漏来源：
  - 未清理的 `setInterval`/`setTimeout`：在 `onUnmounted` 里 `clearInterval`。
  - 未解绑的 `addEventListener`：在 `onUnmounted` 里 `removeEventListener`，或用 Vue 的 `@event` 自动绑定。
  - 遗留的 Pinia 订阅（`store.$subscribe`）、全局 `watch`：组件卸载时不会自动停止，需手动 `stop()` 或在 `effectScope` 内创建。
  - 路由全局守卫（`router.beforeEach`）返回的清理函数未调用。
- Vue 3.5+ 优先用 `effectScope` + `onScopeDispose` 统一管理副作用生命周期。

### 性能卡顿

- 用 Chrome **Performance** 面板录制，看火焰图中长任务（>50ms）和脚本占比。
- 检查 `v-for` 是否稳定 `key`、是否在列表项里做了昂贵的计算（应提为 `computed` 或用 `v-memo`）。
- 大列表用虚拟滚动（`vue-virtual-scroller`、Element Plus `el-table-v2`、Vant `van-list` 分页）。
- 频繁触发的搜索用防抖（`useDebounceFn`、lodash `debounce`）。
- 用 Lighthouse 跑 Core Web Vitals（LCP/CLS/INP），定位首屏和交互性能瓶颈。

## SSR 与 hydration 问题

- **hydration mismatch**：服务端渲染的 HTML 与客户端首屏渲染不一致。Console 会报 `Hydration node mismatch`，DevTools 里 SSR 节点与 CSR 节点对比可见差异。
- 常见原因：依赖 `window`/`document`/`localStorage` 的代码在 SSR 阶段执行、组件渲染依赖时间戳或随机数、客户端-only 的组件未用 `<ClientOnly>` 或 `ClientOnly` 包裹。
- 定位时先注释掉 `onMounted`/`onBeforeMount` 中访问浏览器 API 的代码，确认是否 SSR 阶段触发。
- 用 Vite 的 `ssrLoadModule` 调试 SSR 入口模块加载，或在启动时加 `--debug` 查看 Vite 的模块解析与加载链路。

## 生产环境错误捕获

- 全局错误兜底用 `app.config.errorHandler`，捕获未处理的渲染错误。
- 组件内用 `onErrorCaptured` 捕获后代组件错误，决定是否阻止向上传播。
- 路由错误用 `router.onError`，捕获懒加载 chunk 失败（如 `ChunkLoadError`）并提供重试/刷新恢复。
- 监控接入（Sentry 等）需开启 Source Map 上传或 `debugId`，否则生产堆栈只显示压缩后位置。

```js
// main.js
app.config.errorHandler = (err, instance, info) => {
  console.error('[全局错误]', err, info)
  // 上报到监控平台
}

router.onError((error) => {
  if (/Loading chunk|ChunkLoadError/.test(error.message)) {
    window.location.reload()
  }
})
```

## Element Plus 问题

- 表单问题先查 `model`、`rules`、`prop` 是否一致。
- 弹窗残留通常来自打开时未重置表单或未清校验。
- 表格错位先查列宽、固定列、容器高度和数据 key。
- 下拉、日期、级联组件异常时确认绑定值类型和 options 结构。

## Vant H5 问题

- 真机问题优先用远程调试或抓包复现，不只看桌面模拟器。
- 输入框问题检查键盘类型、maxlength、formatter、readonly 和弹层遮挡。
- 安全区、底部按钮遮挡检查 `safe-area-inset-bottom` 和固定定位容器。
- 滚动穿透、弹层层级问题检查 popup、body 滚动锁定和 z-index。
- iOS 兼容问题优先验证真实 Safari/WebView。

## 样式和布局问题

- 先在 Elements 面板确认 DOM 结构和最终生效样式。
- 检查作用域样式、深度选择器、组件库覆盖顺序。
- 响应式问题检查断点、容器宽度、flex/grid 收缩和文本溢出。
- 不用全局强选择器粗暴覆盖组件库，优先收敛到当前组件或业务类名。

## 白屏和构建问题

- Console 查首个错误，不被后续连锁错误干扰。
- Network 查 JS/CSS 是否 404、MIME 错误、chunk 加载失败。
- 路由 base、静态资源 publicPath、环境变量和部署路径一起核对。
- Source Map 可用时定位源码；不可用时根据 chunk 和路由范围缩小问题。

## 完成检查

- 复现路径已记录。
- 根因有证据支撑。
- 修复范围最小。
- 原问题路径和相邻状态已验证。
