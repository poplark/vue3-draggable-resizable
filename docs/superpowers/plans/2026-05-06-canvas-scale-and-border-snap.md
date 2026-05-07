# 画布缩放和边界吸附实现计划

> **对于 agent 工作者**：必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐步实现此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标**：添加画布缩放（50%-200%）功能，由用户控制视觉呈现，以及基于缩放比例的动态阈值智能边界吸附。

**架构**：通过 provide/inject 集中式管理缩放。DraggableContainer 提供缩放数据但不应用 CSS transform。用户在父元素中应用视觉缩放。Vue3DraggableResizable 注入缩放用于吸附计算。

**技术栈**：Vue 3.0.0 (Composition API), TypeScript 3.9.3

---

## 任务 1：在 DraggableContainer 添加 scale prop

**文件**：
- 修改：`src/components/DraggableContainer.ts:14-39`

- [ ] **步骤 1：在 props 对象中添加 scale prop**

位置：在 `referenceLineColor` prop 之后（第 39 行）

```typescript
referenceLineColor: {
  type: String,
  default: '#f00'
},
scale: {
  type: Number,
  default: 1.0,
  validator: (value: number) => {
    return value >= 0.5 && value <= 2.0
  }
}
```

- [ ] **步骤 2：向子组件提供 scale**

位置：在 `setup()` 函数中，在现有 provide 语句之后（第 75 行）

```typescript
provide('adsorbCols', props.adsorbCols || [])
provide('adsorbRows', props.adsorbRows || [])
provide('scale', toRef(props, 'scale'))
```

- [ ] **步骤 3：运行开发服务器并验证无错误**

运行：`npm run dev`
预期：开发服务器启动，无 TypeScript 错误

- [ ] **步骤 4：提交**

```bash
git add src/components/DraggableContainer.ts
git commit -m "feat: 在 DraggableContainer 添加 scale prop

添加 scale prop（0.5-2.0，默认 1.0）并通过 inject 提供给子组件。
视觉缩放由用户在父元素中处理。
"
```

---

## 任务 2：在 Vue3DraggableResizable 添加 snap props

**文件**：
- 修改：`src/components/Vue3DraggableResizable.ts:31-131`

- [ ] **步骤 1：添加 snapToBorder prop**

位置：在 `lockAspectRatio` prop 之后（第 130 行）

```typescript
lockAspectRatio: {
  type: Boolean,
  default: false
},
snapToBorder: {
  type: Boolean,
  default: false
}
```

- [ ] **步骤 2：添加 snapThreshold prop**

位置：在 `snapToBorder` prop 之后

```typescript
snapToBorder: {
  type: Boolean,
  default: false
},
snapThreshold: {
  type: Number,
  default: 10,
  validator: (value: number) => value > 0
}
```

- [ ] **步骤 3：在 setup() 中添加 scale 的 inject**

位置：在 `setup()` 函数开头（第 153 行），在 `containerProps` 声明之后

```typescript
setup(props, { emit }) {
    const containerProps = initState(props, emit)
    const scale = inject<Ref<number>>('scale', ref(1.0))
```

- [ ] **步骤 4：将新 props 传递给 initResizeHandle**

位置：在 `initResizeHandle` 的调用中（第 184-190 行），添加 scale、snapToBorder 和 snapThreshold

当前代码：
```typescript
const resizeHandle = initResizeHandle(
  containerProps,
  limitProps,
  parentSize,
  props,
  emit
)
```

更改为：
```typescript
const resizeHandle = initResizeHandle(
  containerProps,
  limitProps,
  parentSize,
  props,
  emit,
  scale
)
```

- [ ] **步骤 5：运行开发服务器并验证无错误**

运行：`npm run dev`
预期：开发服务器启动，无 TypeScript 错误

- [ ] **步骤 6：提交**

```bash
git add src/components/Vue3DraggableResizable.ts
git commit -m "feat: 添加 snapToBorder 和 snapThreshold props

为智能边界吸附功能添加 props：
- snapToBorder：启用/禁用边界吸附（默认 false）
- snapThreshold：基础吸附距离（像素，默认 10px）
从 DraggableContainer 注入 scale 用于动态阈值计算
"
```

---

## 任务 3：修改 hooks.ts 中的 initResizeHandle

**文件**：
- 修改：`src/components/hooks.ts:392-553`

- [ ] **步骤 1：更新 initResizeHandle 函数签名**

位置：第 392 行，函数定义

当前代码：
```typescript
export function initResizeHandle(
  containerProps: any,
  limitProps: any,
  parentSize: any,
  props: any,
  emit: any
) {
```

更改为：
```typescript
export function initResizeHandle(
  containerProps: any,
  limitProps: any,
  parentSize: any,
  props: any,
  emit: any,
  scale: Ref<number>
) {
```

- [ ] **步骤 2：找到 handleResize 函数**

在 `initResizeHandle` 中找到 `handleResize` 函数（大约在第 450-500 行）。这是处理调整大小时鼠标移动事件的地方。

- [ ] **步骤 3：在 handleResize 开头添加动态吸附阈值计算**

位置：在 `handleResize` 函数开头

```typescript
const handleResize = (e: MouseEvent | TouchEvent) => {
  // 基于缩放计算动态吸附阈值
  const dynamicThreshold = props.snapToBorder ? props.snapThreshold / scale.value : 0
```

- [ ] **步骤 4：找到计算新宽度/高度的位置**

查找基于鼠标移动更新 `width` 和 `height` 的代码。这通常在增量计算之后。

- [ ] **步骤 5：在宽度/高度计算后添加边界吸附逻辑**

位置：在计算宽度和高度之后，添加吸附逻辑：

```typescript
// 应用到右边界吸附
if (props.snapToBorder && handle.direction.includes('r')) {
  const distanceToRight = parentSize.width.value - (left + width)
  if (distanceToRight <= dynamicThreshold && distanceToRight > 0) {
    width = parentSize.width.value - left
  }
}

// 应用到下边界吸附
if (props.snapToBorder && handle.direction.includes('b')) {
  const distanceToBottom = parentSize.height.value - (top + height)
  if (distanceToBottom <= dynamicThreshold && distanceToBottom > 0) {
    height = parentSize.height.value - top
  }
}
```

注意：`handle.direction` 包含方向字符串，如 'mr'（中右）、'br'（右下）等。检查是否包含 'r' 表示右边缘，'b' 表示下边缘。

- [ ] **步骤 6：运行开发服务器并手动测试**

运行：`npm run dev`
测试：在浏览器中，尝试在靠近右/下边缘处调整 `snapToBorder=true` 的组件大小
预期：组件应该在足够近时吸附到边缘

- [ ] **步骤 7：提交**

```bash
git add src/components/hooks.ts
git commit -m "feat: 实现带动态阈值的智能边界吸附

- 计算动态吸附阈值：baseThreshold / scale
- 检测与右边界和下边界的接近程度
- 当距离在阈值内时吸附组件边缘到边界
- 仅在 snapToBorder prop 为 true 时激活
"
```

---

## 任务 4：更新 types.ts（如果需要）

**文件**：
- 修改：`src/components/types.ts`

- [ ] **步骤 1：检查 ContainerProvider 接口是否存在**

读取文件，查看 `ContainerProvider` 接口是否需要更新 scale 字段。

当前接口（大约在第 20-30 行）：
```typescript
interface ContainerProvider {
  updatePosition: UpdatePosition
  getPositionStore: GetPositionStore
  disabled: Ref<boolean>
  adsorbParent: Ref<boolean>
  adsorbCols: number[]
  adsorbRows: number[]
  setMatchedLine: SetMatchedLine
}
```

- [ ] **步骤 2：如果接口存在并被使用，添加 scale**

如果接口存在并且正在被使用，添加：

```typescript
interface ContainerProvider {
  updatePosition: UpdatePosition
  getPositionStore: GetPositionStore
  disabled: Ref<boolean>
  adsorbParent: Ref<boolean>
  adsorbCols: number[]
  adsorbRows: number[]
  setMatchedLine: SetMatchedLine
  scale?: Ref<number>
}
```

注意：使用 `?` 使其成为可选的，因为现有代码可能不会使用它。

- [ ] **步骤 3：运行开发服务器并验证无错误**

运行：`npm run dev`
预期：无 TypeScript 错误

- [ ] **步骤 4：提交**

```bash
git add src/components/types.ts
git commit -m "feat: 向 ContainerProvider 接口添加 scale"
```

---

## 任务 5：在 App.vue 中创建测试示例

**文件**：
- 修改：`src/App.vue`

- [ ] **步骤 1：添加缩放控制按钮和 DraggableContainer**

替换整个模板：

```vue
<template>
  <div id="app">
    <div style="margin-bottom: 20px;">
      <button @click="scale = 0.5">50%</button>
      <button @click="scale = 1.0">100%</button>
      <button @click="scale = 1.5">150%</button>
      <button @click="scale = 2.0">200%</button>
      <span style="margin-left: 20px;">当前：{{ scale * 100 }}%</span>
    </div>

    <div style="margin-bottom: 20px;">
      <label>
        <input type="checkbox" v-model="snapToBorder"> 启用边界吸附
      </label>
    </div>

    <!-- 用户在这里应用视觉缩放 -->
    <div :style="{ transform: `scale(${scale})`, transformOrigin: 'top left', width: '600px', height: '600px' }">
      <div class="parent">
        <DraggableContainer :scale="scale">
          <Vue3DraggableResizable
            :initW="100"
            :initH="100"
            v-model:x="x"
            v-model:y="y"
            v-model:w="w"
            v-model:h="h"
            v-model:active="active"
            :draggable="true"
            :resizable="true"
            :parent="true"
            :snapToBorder="snapToBorder"
            :snapThreshold="10"
            @activated="print('activated')"
            @deactivated="print('deactivated')"
            @drag-start="print('drag-start', $event)"
            @resize-start="print('resize-start', $event)"
            @dragging="print('dragging', $event)"
            @resizing="print('resizing', $event)"
            @drag-end="print('drag-end', $event)"
            @resize-end="print('resize-end', $event)"
          >
            这是一个测试示例
          </Vue3DraggableResizable>
        </DraggableContainer>
      </div>
    </div>
  </div>
</template>
```

- [ ] **步骤 2：向 data() 添加 scale 和 snapToBorder**

更新 data() 返回对象（第 56-65 行）：

```typescript
data() {
  return {
    x: 100,
    y: 100,
    h: 100,
    w: 100,
    active: false,
    draggable: true,
    resizable: true,
    scale: 1.0,
    snapToBorder: false
  };
},
```

- [ ] **步骤 3：运行开发服务器并测试所有功能**

运行：`npm run dev`

**测试 1：缩放**
- 点击 50%、100%、150%、200% 按钮
- 预期：容器视觉上缩放，逻辑坐标保持不变

**测试 2：边界吸附（scale = 100%）**
- 勾选"启用边界吸附"复选框
- 拖动组件靠近右边缘
- 拖动调整大小手柄（右侧或右下侧）靠近右边缘
- 预期：组件在足够近时吸附到边缘

**测试 3：动态吸附阈值（scale = 50%）**
- 设置缩放为 50%
- 启用边界吸附
- 靠近边缘调整大小
- 预期：吸附距离为 20px（10 / 0.5）而不是 10px

**测试 4：动态吸附阈值（scale = 200%）**
- 设置缩放为 200%
- 启用边界吸附
- 靠近边缘调整大小
- 预期：吸附距离为 5px（10 / 2.0）而不是 10px

- [ ] **步骤 4：提交**

```bash
git add src/App.vue
git commit -m "test: 添加缩放和吸附功能的综合测试示例

测试：
- 缩放控制（50%、100%、150%、200%）
- 边界吸附启用/禁用
- 基于缩放的动态吸附阈值
- 用户在父元素中应用的视觉缩放
"
```

---

## 任务 6：更新 README.md

**文件**：
- 修改：`README.md`

- [ ] **步骤 1：添加画布缩放功能文档**

位置：在 "Props" 部分之后，"Events" 之前（大约第 430 行）

```markdown
### 画布缩放功能

画布缩放功能允许您缩放整个容器，同时保持逻辑坐标不变。视觉缩放由您在父元素中应用。

**用法：**

```vue
<template>
  <div :style="{ transform: `scale(${scale})`, transformOrigin: 'top left' }">
    <DraggableContainer :scale="scale">
      <Vue3DraggableResizable
        v-model:x="x"
        v-model:y="y"
        v-model:w="w"
        v-model:h="h"
      >
        内容
      </Vue3DraggableResizable>
    </DraggableContainer>
  </div>
</template>

<script>
export default {
  data() {
    return {
      scale: 0.8,
      x: 100, y: 100, w: 100, h: 100
    }
  }
}
</script>
```

**DraggableContainer Props:**

#### scale

类型：`Number`<br>
默认值：`1.0`<br>
范围：`0.5 - 2.0`

画布的缩放因子。此值提供给子组件用于吸附距离等计算。您应该使用 CSS transform 自己应用视觉缩放。

```
```

- [ ] **步骤 2：添加边界吸附功能文档**

位置：在画布缩放文档之后

```markdown
### 边界吸附功能

调整大小时自动将组件边缘吸附到容器边界。吸附距离根据缩放因子自动调整。

**用法：**

```vue
<template>
  <DraggableContainer :scale="scale">
    <Vue3DraggableResizable
      v-model:x="x"
      v-model:y="y"
      v-model:w="w"
      v-model:h="h"
      :snapToBorder="true"
      :snapThreshold="15"
    >
      内容
    </Vue3DraggableResizable>
  </DraggableContainer>
</template>
```

**Vue3DraggableResizable Props:**

#### snapToBorder

类型：`Boolean`<br>
默认值：`false`

启用调整大小时自动吸附到容器边界。

#### snapThreshold

类型：`Number`<br>
默认值：`10`<br>

基础吸附距离阈值（像素）。实际阈值计算为 `snapThreshold / scale`，因此吸附在不同缩放级别下正常工作。

示例：
- scale = 1.0, snapThreshold = 10 → 在 10px 处吸附
- scale = 0.5, snapThreshold = 10 → 在 20px 处吸附
- scale = 2.0, snapThreshold = 10 → 在 5px 处吸附
```
```

- [ ] **步骤 3：提交**

```bash
git add README.md
git commit -m "docs: 添加画布缩放和边界吸附功能文档

- 为 DraggableContainer 记录 scale prop
- 为 Vue3DraggableResizable 记录 snapToBorder 和 snapThreshold props
- 添加使用示例
- 解释动态吸附阈值计算
"
```

---

## 任务 7：构建并验证库

**文件**：
- 构建输出：`dist/Vue3DraggableResizable.js`

- [ ] **步骤 1：构建库**

运行：`npm run build`
预期：构建成功无错误，生成 dist 文件

- [ ] **步骤 2：检查 TypeScript 声明**

运行：`npm run build:dts`
预期：在 `typings/` 中成功生成类型定义

- [ ] **步骤 3：提交**

```bash
git add -A
git commit -m "build: 验证带新功能的库构建成功"
```

---

## 任务 8：最终测试和边界情况

**文件**：
- 浏览器中的手动测试

- [ ] **步骤 1：测试缩放验证**

在 App.vue 中测试 [0.5, 2.0] 范围之外的值：

```javascript
scale: 0.1  // 应该警告并使用默认值
scale: 3.0  // 应该警告并使用默认值
```

预期：Vue 在控制台中警告，使用默认值 1.0

- [ ] **步骤 2：测试快速鼠标移动**

测试：快速拖动调整大小手柄越过吸附区域
预期：吸附仍然触发（鼠标事件触发足够频繁）

- [ ] **步骤 3：测试多个组件**

修改 App.vue 使其有多个 Vue3DraggableResizable 组件
预期：每个独立工作，吸附对所有组件有效

- [ ] **步骤 4：测试向后兼容性**

创建一个不使用新 props 的测试（无 scale、无 snapToBorder）：
```vue
<DraggableContainer>
  <Vue3DraggableResizable v-model:x="x" v-model:y="y" v-model:w="w" v-model:h="h">
    旧用法
  </Vue3DraggableResizable>
</DraggableContainer>
```

预期：完全像以前一样工作（scale=1.0, snapToBorder=false）

- [ ] **步骤 5：更新 CLAUDE.md 功能状态**

编辑 CLAUDE.md 第 67 行：
```markdown
**当前功能状态**：
- ✅ 组件拖拽功能（带参考线对齐）
- ✅ 组件调整大小功能
- ✅ 画布缩放功能（已实现）
- ✅ 组件大小调整时的边界吸附（已实现）
```

- [ ] **步骤 6：最终提交**

```bash
git add CLAUDE.md
git commit -m "docs: 更新 CLAUDE.md 功能状态

标记画布缩放和边界吸附功能为已完成。
所有测试通过，向后兼容性已验证。
"
```

---

## 实现完成！

**总结：**
- ✅ DraggableContainer 通过 prop 和 inject 提供 scale
- ✅ 用户使用 CSS transform 控制视觉缩放
- ✅ Vue3DraggableResizable 具有 snapToBorder 和 snapThreshold props
- ✅ 基于缩放的动态阈值智能边界吸附
- ✅ App.vue 中的综合测试示例
- ✅ README.md 中的文档
- ✅ 向后兼容（所有新 props 都有默认值）
- ✅ 库构建成功

**后续步骤：**
1. 打标签发布：`git tag -a v1.7.0 -m "添加画布缩放和智能边界吸附功能"`
2. 推送到 GitHub：`git push origin main --tags`
3. 发布到 npm：`npm publish`
