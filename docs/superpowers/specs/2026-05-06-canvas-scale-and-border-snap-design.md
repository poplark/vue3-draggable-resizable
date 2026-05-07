# 画布缩放和边界吸附设计文档

**日期**: 2026-05-06
**状态**: 已批准
**作者**: Claude 与用户协作

## 概述

本文档描述了要添加到 vue3-draggable-resizable 组件库的两个新功能的设计：

1. **画布缩放功能**：允许用户缩放整个 DraggableContainer 及其所有子组件（50%-200%）
2. **智能边界吸附功能**：在调整组件大小时自动吸附到容器边界，基于缩放比例的智能阈值

## 需求

### 画布缩放功能
- 支持画布放大/缩小（50% - 200%）
- 通过 props 提供 API 接口
- 缩放时保持拖拽和对齐功能正常
- 使用 CSS transform 进行视觉缩放（逻辑坐标保持不变）

### 边界吸附功能
- 调整组件大小时自动吸附到容器边界
- 通过 props 控制（snapToBorder）
- 根据缩放比例调整的智能吸附阈值
- 只吸附到右边界和下边界

## 架构

### 设计方案：集中式缩放管理

我们使用集中式方法，`DraggableContainer` 管理缩放状态，并通过 Vue 的 `provide/inject` 机制将其提供给所有子组件。

**组件关系**：
```
DraggableContainer (提供 scale, containerSize)
  └── Vue3DraggableResizable (注入 scale, containerSize)
       └── hooks.ts (initResizeHandle 使用 scale 计算智能吸附阈值)
```

**数据流**：
1. 父组件将 `:scale="0.8"` prop 传递给 DraggableContainer
2. DraggableContainer 验证缩放范围（0.5-2.0），提供给子组件
3. 用户在父元素中应用 CSS `transform: scale()`（视觉呈现）
4. Vue3DraggableResizable 注入 scale
5. hooks.ts 调整大小逻辑计算动态吸附距离：`baseThreshold / scale`

## 组件设计

### DraggableContainer 修改

**新增 Props**：
```typescript
scale: {
  type: Number,
  default: 1.0,
  validator: (value: number) => {
    return value >= 0.5 && value <= 2.0
  }
}
```

**Provide 逻辑**：
```typescript
provide('scale', toRef(props, 'scale'))
```

**注意**：DraggableContainer 不应用 CSS transform。视觉缩放由用户在父元素中处理。

### Vue3DraggableResizable 修改

**新增 Props**：
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

**Inject 逻辑**：
```typescript
const scale = inject<Ref<number>>('scale', ref(1.0))
```

**传递给 Hooks**：
- 将 `scale`、`snapToBorder` 和 `snapThreshold` 传递给 `initResizeHandle` 函数

### hooks.ts 修改（initResizeHandle）

**核心逻辑 - 智能边界吸附**：

1. **计算动态吸附阈值**：
```typescript
const dynamicThreshold = props.snapThreshold / scale.value
```

2. **检测边界接近**：
```typescript
const distanceToRight = parentSize.width - (left + width)
const distanceToBottom = parentSize.height - (top + height)
```

3. **触发吸附**：
```typescript
if (snapToBorder && distanceToRight <= dynamicThreshold) {
  width = parentSize.width - left
}
if (snapToBorder && distanceToBottom <= dynamicThreshold) {
  height = parentSize.height - top
}
```

4. **应用于特定手柄**：
吸附逻辑只在向右或向下方向调整大小时激活（手柄：`mr`、`br`、`bm`、`tr`、`bl`）

### types.ts 修改

**类型扩展**：
- 扩展 `ContainerProvider` 接口以包含 `scale` 字段（如果需要）
- 确保为新 props 提供正确的 TypeScript 类型

## 数据流示例

### 示例 1：用户将缩放设置为 0.5

```
1. 父组件将 :scale="0.5" 传递给 DraggableContainer
   ↓
2. DraggableContainer 接收并验证 scale
   ↓
3. DraggableContainer.provide('scale', ref(0.5))
   ↓
4. 用户在父元素中应用 CSS transform: scale(0.5)（视觉）
   ↓
5. Vue3DraggableResizable.inject('scale') → ref(0.5)
   ↓
6. hooks.ts initResizeHandle 接收 scale
   ↓
7. 计算动态吸附阈值：10 / 0.5 = 20px
   ↓
8. 组件视觉上缩小 50%（通过用户的 CSS），逻辑坐标不变
```

### 示例 2：用户在边界附近调整组件大小

```
场景：容器 800x600，组件位于 (100, 100)，当前大小 100x100，
用户拖动右下角手柄

1. 拖动过程中，组件右边缘到达 700px（距右边界 100px）
2. scale = 0.5, dynamicThreshold = 10 / 0.5 = 20px
3. 当距离 <= 20px 时，触发吸附
4. 组件宽度自动调整为 700px（吸附到右边界）
5. 用户松开鼠标，触发 resize-end 事件
```

## 交互设计

### 缩放行为
- 默认 0.2s 过渡动画以提供流畅的用户体验
- 可通过未来的 props 配置（不在初始实现中）
- 逻辑坐标（x, y, w, h）保持不变
- 仅通过 CSS transform 改变视觉呈现

### 吸附行为
- 吸附仅在调整大小时触发（不在拖拽/移动时）
- 吸附是瞬间的（无过渡动画）
- 只检查右边界和下边界
- `resizing` 和 `resize-end` 事件包含吸附后的值

### 与现有功能的兼容性
- ✅ 拖拽功能：不受影响（逻辑坐标不变）
- ✅ 参考线对齐：继续工作，吸附距离也感知缩放
- ✅ parent prop：容器限制仍然适用
- ✅ lockAspectRatio：与吸附功能无冲突

## 边界情况和错误处理

### 输入验证
- **scale 超出 [0.5, 2.0]**：Vue 验证器拦截，控制台警告，默认为 1.0
- **scale <= 0**：验证器拦截
- **snapThreshold <= 0**：验证器拦截
- 无效的 props 由 Vue 的内置验证处理

### 边界场景
- **快速拖动鼠标跳过吸附区域**：连续的鼠标事件确保至少有一帧会触发吸附
- **组件初始在边界外**：`parent` prop 处理初始化，吸附不处理初始位置
- **多个组件缩放**：每个组件通过 inject 独立接收 scale

## 测试策略

### 单元测试

**DraggableContainer**：
- ✅ scale prop 验证（0.5-2.0 范围）
- ✅ scale 通过 provide/inject 正确提供
- ✅ CSS transform 正确应用到根元素
- ✅ transform-origin 设置为 top left

**Vue3DraggableResizable**：
- ✅ snapToBorder prop 正确传递给 hooks
- ✅ snapThreshold prop 验证
- ✅ scale 通过 inject 正确获取

**hooks.ts initResizeHandle**：
- ✅ 动态吸附阈值计算正确
- ✅ 边界检测逻辑正确
- ✅ 吸附仅在启用时激活
- ✅ 仅适用于特定手柄方向

### 集成测试

- ✅ 缩放到 50%，拖拽组件仍正常工作
- ✅ 缩放到 50%，参考线对齐正常工作
- ✅ 缩放到 50%，调整大小吸附距离正确（20px）
- ✅ 缩放到 200%，调整大小吸附距离正确（5px）
- ✅ 启用 snapToBorder，调整大小到右边界触发吸附
- ✅ 启用 snapToBorder，调整大小到下边界触发吸附
- ✅ 禁用 snapToBorder，不发生吸附
- ✅ 连续缩放（0.5 → 1.0 → 1.5 → 2.0）无错误

### 手动测试

- ✅ 快速拖动调整大小手柄
- ✅ 多个组件具有不同的吸附设置
- ✅ 跨浏览器测试（Chrome、Firefox、Safari）
- ✅ 移动设备上的触摸事件

## 实现计划

### 阶段 1：画布缩放功能（高优先级）
1. 在 DraggableContainer 添加 scale prop
2. 实现 provide 逻辑
3. 应用 CSS transform
4. 基础测试

### 阶段 2：智能边界吸附（高优先级）
1. 在 Vue3DraggableResizable 添加 snapToBorder 和 snapThreshold props
2. 在 hooks.ts 实现动态吸附阈值计算
3. 实现边界检测和吸附逻辑
4. 集成测试

### 阶段 3：优化和文档（中优先级）
1. 添加单元测试
2. 更新 README.md
3. 添加使用示例
4. 性能优化（如果需要）

## 向后兼容性

所有更改都是 **100% 向后兼容**：
- 所有新 props 都有默认值
- 默认 scale=1.0（无视觉变化）
- 默认 snapToBorder=false（无吸附行为）
- 现有功能完全不受影响

## 风险评估

| 风险 | 影响 | 缓解措施 |
|------|--------|----------|
| 缩放影响拖拽体验 | 中 | 逻辑坐标不变，拖拽逻辑不受影响 |
| 吸附与 parent prop 冲突 | 低 | 两者都限制在容器内，逻辑一致 |
| Transform 影响事件坐标 | 低 | 使用逻辑坐标，CSS transform 不影响鼠标事件 |
| 性能问题 | 低 | GPU 加速，仅在调整大小时计算 |

## 需要修改的文件

1. **src/components/DraggableContainer.ts**
   - 添加 scale prop 和验证
   - 实现 provide 逻辑
   - 应用 CSS transform 样式

2. **src/components/Vue3DraggableResizable.ts**
   - 添加 snapToBorder 和 snapThreshold props
   - 为 scale 添加 inject 逻辑
   - 将新 props 传递给 hooks

3. **src/components/hooks.ts**
   - 修改 initResizeHandle 函数
   - 实现动态吸附阈值计算
   - 实现边界检测和吸附逻辑

4. **src/components/types.ts**
   - 扩展 ContainerProvider 接口（如果需要）

5. **README.md**
   - 为新 props 编写文档
   - 添加缩放功能的使用示例
   - 添加吸附功能的使用示例

## 使用示例

### 画布缩放

```vue
<template>
  <!-- 用户在父元素中应用视觉缩放 -->
  <div :style="{ transform: `scale(${scale})`, transformOrigin: 'top left' }">
    <DraggableContainer :scale="0.8">
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

### 边界吸附

```vue
<template>
  <DraggableContainer>
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

### 组合使用

```vue
<template>
  <div>
    <button @click="scale = 0.5">50%</button>
    <button @click="scale = 1.0">100%</button>
    <button @click="scale = 1.5">150%</button>

    <!-- 视觉缩放在这里应用 -->
    <div :style="{ transform: `scale(${scale})`, transformOrigin: 'top left' }">
      <DraggableContainer :scale="scale">
        <Vue3DraggableResizable
          v-model:x="x"
          v-model:y="y"
          v-model:w="w"
          v-model:h="h"
          :snapToBorder="true"
        >
          拖动我并调整大小以查看吸附效果！
        </Vue3DraggableResizable>
      </DraggableContainer>
    </div>
  </div>
</template>
```

## 结论

本设计为画布缩放和智能边界吸附提供了一个干净、可维护的解决方案。集中式架构与代码库中的现有模式一致，所有更改都是向后兼容的，并具有全面的测试覆盖。
