---
name: element-plus-forms
description: Element Plus PC 端表单开发规范。用于 Vue 3 + JavaScript 项目中实现、修复或重构 el-form、el-form-item、rules、弹窗新增/编辑表单、查询表单、动态校验、异步校验、提交 loading、防重复提交和接口错误回填。
---

# Element Plus Forms

## 工作原则

- 优先复用项目已有表单组件、请求工具、字典工具和校验方法。
- 使用 Vue 3 Composition API，但代码按 JavaScript 项目编写，不加入类型标注。
- 表单状态、校验规则、提交状态、弹窗状态分开维护，避免把所有逻辑塞进模板。
- 所有提交类异步请求必须 `try/catch/finally`，失败时给出兜底提示并恢复 loading。

## 基础结构

- `el-form` 必须绑定 `ref`、`:model`、`:rules`。
- 每个需要校验的 `el-form-item` 必须设置 `prop`，且 `prop` 与 `model` 字段一致。
- `rules` 保持稳定，避免在模板内临时拼接复杂规则。
- 数字输入优先用 `v-model.number` 或提交前显式转换，避免字符串数字进入接口。
- 表单默认值集中定义，用工厂函数生成，避免新增/编辑互相污染。

```vue
<script setup>
import { reactive, ref, nextTick } from 'vue'

const formRef = ref()
const visible = ref(false)
const submitting = ref(false)

const createDefaultForm = () => ({
  name: '',
  status: '',
  sort: 0,
})

const form = reactive(createDefaultForm())

const rules = {
  name: [{ required: true, message: '请输入名称', trigger: 'blur' }],
  status: [{ required: true, message: '请选择状态', trigger: 'change' }],
}

function assignForm(data = {}) {
  Object.assign(form, createDefaultForm(), data)
}
</script>
```

## 新增/编辑弹窗

- 新增：打开前写入默认值，`nextTick` 后清除历史校验。
- 编辑：先回填接口数据，再 `clearValidate()`，不要让旧错误跟随弹窗打开。
- 不要直接替换 `reactive` 表单对象；用 `Object.assign` 保持响应式引用稳定。
- 关闭弹窗时根据业务选择 `resetFields()` 或重新写入默认值，避免编辑数据残留。

```js
async function openCreate() {
  assignForm()
  visible.value = true
  await nextTick()
  formRef.value?.clearValidate()
}

async function openEdit(row) {
  assignForm(row)
  visible.value = true
  await nextTick()
  formRef.value?.clearValidate()
}
```

## 弹窗可见性用 defineModel

弹窗组件的 `visible` 是父子双向同步的典型场景，Vue 3.4+ 用 `defineModel` 替代手写 `props + emit('update:visible')`：

```vue
<!-- EditDialog.vue -->
<script setup>
import { reactive, ref, nextTick } from 'vue'

// 父组件用 v-model:visible 绑定，无需手动维护 update:visible 事件
const visible = defineModel('visible', { default: false })

const formRef = ref()
const form = reactive({ name: '', status: '' })

async function handleClose() {
  visible.value = false
}
</script>

<template>
  <el-dialog v-model="visible" title="编辑" @closed="handleClose">
    <el-form ref="formRef" :model="form">
      <!-- ... -->
    </el-form>
  </el-dialog>
</template>
```

父组件直接 `v-model:visible` 绑定，不再需要监听 close 事件回写状态：

```vue
<script setup>
import { ref } from 'vue'

const dialogVisible = ref(false)
</script>

<template>
  <EditDialog v-model:visible="dialogVisible" @submit="handleSubmit" />
</template>
```

## 提交流程

- 提交前先 `await formRef.value.validate()`。
- 校验失败不请求接口；接口失败不关闭弹窗。
- 使用 `submitting` 禁止重复提交。
- 提交成功后再关闭弹窗、刷新列表或回传事件。

```js
async function handleSubmit() {
  if (!formRef.value || submitting.value) return

  try {
    await formRef.value.validate()
    submitting.value = true
    await saveApi({ ...form })
    visible.value = false
    await loadList()
  } catch (error) {
    handleSubmitError(error)
  } finally {
    submitting.value = false
  }
}
```

## 动态和异步校验

- 字段联动后，用 `clearValidate(field)` 清掉已失效的错误。
- 只校验局部字段时使用 `validateField(field)`。
- 异步唯一性校验必须处理接口异常；异常时返回明确错误，不静默通过。
- 后端返回字段级错误时，优先映射到对应字段；无法映射时用全局消息提示。

### 局部校验示例

联动改变 A 字段后，单独校验 B 字段而非整表：

```js
async function onTypeChange() {
  await nextTick()
  formRef.value?.clearValidate('code')
  // 只校验受影响的字段，避免整表弹错
  formRef.value?.validateField('code')
}
```

### 校验失败滚动定位

长表单校验失败时，Element Plus 默认不滚动到错误字段。用 `scroll-to-error` 让表单自动滚动到第一个错误项：

```vue
<el-form ref="formRef" :model="form" :rules="rules" scroll-to-error>
  <!-- 字段较多时，校验失败会自动滚到首个错误 -->
</el-form>
```

## 文件上传联动校验

`el-upload` 不直接参与 `el-form` 的 rules 校验，需用 `el-form-item` 包裹并手动触发：

- 用 `el-form-item` 的 `prop` 绑定一个存放文件列表的字段（如 `fileList`）。
- 在 `el-upload` 的 `on-change`/`on-remove` 里调用 `formRef.validateField('fileList')` 触发该字段校验。
- rules 里校验数组长度（如 `required` 时至少 1 个文件），不要校验文件对象本身。

```js
const rules = {
  fileList: [
    {
      required: true,
      validator: (_rule, value, callback) => {
        if (!value || value.length === 0) callback(new Error('请上传文件'))
        else callback()
      },
      trigger: 'change',
    },
  ],
}

function onFileChange() {
  formRef.value?.validateField('fileList')
}
```

## 查询表单

- 查询表单和编辑表单分开建模。
- 查询条件提交前清理空字符串、空数组和无效日期。
- 重置查询条件后立即回到第一页并重新请求列表。

## 完成检查

- 校验字段的 `prop` 与 `model` 完全匹配。
- 打开新增/编辑弹窗不会残留旧值或旧校验错误。
- 提交 loading、重复点击、接口失败、校验失败都有闭环。
- 所有异步请求都有异常捕获。
