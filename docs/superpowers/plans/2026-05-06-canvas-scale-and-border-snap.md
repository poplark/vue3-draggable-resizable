# Canvas Scale and Border Snap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add canvas scaling (50%-200%) with user-controlled visual presentation and smart border snapping with dynamic threshold based on scale.

**Architecture:** Centralized scale management via provide/inject. DraggableContainer provides scale data but doesn't apply CSS transform. User applies visual scaling in parent element. Vue3DraggableResizable injects scale for snap calculations.

**Tech Stack:** Vue 3.0.0 (Composition API), TypeScript 3.9.3

---

## Task 1: Add scale prop to DraggableContainer

**Files:**
- Modify: `src/components/DraggableContainer.ts:14-39`

- [ ] **Step 1: Add scale prop to props object**

Location: After `referenceLineColor` prop (line 39)

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

- [ ] **Step 2: Provide scale to child components**

Location: In `setup()` function, after existing provide statements (line 75)

```typescript
provide('adsorbCols', props.adsorbCols || [])
provide('adsorbRows', props.adsorbRows || [])
provide('scale', toRef(props, 'scale'))
```

- [ ] **Step 3: Run development server and verify no errors**

Run: `npm run dev`
Expected: Dev server starts without TypeScript errors

- [ ] **Step 4: Commit**

```bash
git add src/components/DraggableContainer.ts
git commit -m "feat: add scale prop to DraggableContainer

Add scale prop (0.5-2.0, default 1.0) and provide it to child components via inject.
Visual scaling is handled by user in parent element.
"
```

---

## Task 2: Add snap props to Vue3DraggableResizable

**Files:**
- Modify: `src/components/Vue3DraggableResizable.ts:31-131`

- [ ] **Step 1: Add snapToBorder prop**

Location: After `lockAspectRatio` prop (line 130)

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

- [ ] **Step 2: Add snapThreshold prop**

Location: After `snapToBorder` prop

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

- [ ] **Step 3: Add inject for scale in setup()**

Location: At the beginning of `setup()` function (line 153), after `containerProps` declaration

```typescript
setup(props, { emit }) {
    const containerProps = initState(props, emit)
    const scale = inject<Ref<number>>('scale', ref(1.0))
```

- [ ] **Step 4: Pass new props to initResizeHandle**

Location: In the call to `initResizeHandle` (line 184-190), add scale, snapToBorder, and snapThreshold

Current code:
```typescript
const resizeHandle = initResizeHandle(
  containerProps,
  limitProps,
  parentSize,
  props,
  emit
)
```

Change to:
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

- [ ] **Step 5: Run development server and verify no errors**

Run: `npm run dev`
Expected: Dev server starts without TypeScript errors

- [ ] **Step 6: Commit**

```bash
git add src/components/Vue3DraggableResizable.ts
git commit -m "feat: add snapToBorder and snapThreshold props

Add props for smart border snapping feature:
- snapToBorder: enable/disable border snap (default false)
- snapThreshold: base snap distance in pixels (default 10px)
Inject scale from DraggableContainer for dynamic threshold calculation
"
```

---

## Task 3: Modify initResizeHandle in hooks.ts

**Files:**
- Modify: `src/components/hooks.ts:392-553`

- [ ] **Step 1: Update initResizeHandle function signature**

Location: Line 392, function definition

Current code:
```typescript
export function initResizeHandle(
  containerProps: any,
  limitProps: any,
  parentSize: any,
  props: any,
  emit: any
) {
```

Change to:
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

- [ ] **Step 2: Locate the handleResize function**

Find the `handleResize` function within `initResizeHandle` (around line 450-500). This is where mouse move events are handled during resizing.

- [ ] **Step 3: Add dynamic snap threshold calculation at the start of handleResize**

Location: At the beginning of `handleResize` function

```typescript
const handleResize = (e: MouseEvent | TouchEvent) => {
  // Calculate dynamic snap threshold based on scale
  const dynamicThreshold = props.snapToBorder ? props.snapThreshold / scale.value : 0
```

- [ ] **Step 4: Find where new width/height are calculated**

Search for the code that updates `width` and `height` based on mouse movement. This is typically after the delta calculations.

- [ ] **Step 5: Add border snap logic after width/height calculation**

Location: After width and height are calculated, add snap logic:

```typescript
// Apply snap to right border
if (props.snapToBorder && handle.direction.includes('r')) {
  const distanceToRight = parentSize.width.value - (left + width)
  if (distanceToRight <= dynamicThreshold && distanceToRight > 0) {
    width = parentSize.width.value - left
  }
}

// Apply snap to bottom border
if (props.snapToBorder && handle.direction.includes('b')) {
  const distanceToBottom = parentSize.height.value - (top + height)
  if (distanceToBottom <= dynamicThreshold && distanceToBottom > 0) {
    height = parentSize.height.value - top
  }
}
```

Note: `handle.direction` contains the direction string like 'mr' (middle-right), 'br' (bottom-right), etc. Check if it includes 'r' for right edge and 'b' for bottom edge.

- [ ] **Step 6: Run development server and test manually**

Run: `npm run dev`
Test: In browser, try resizing a component with `snapToBorder=true` near right/bottom edges
Expected: Component should snap to edges when close enough

- [ ] **Step 7: Commit**

```bash
git add src/components/hooks.ts
git commit -m "feat: implement smart border snapping with dynamic threshold

- Calculate dynamic snap threshold: baseThreshold / scale
- Detect proximity to right and bottom borders
- Snap component edge to border when within threshold
- Only active when snapToBorder prop is true
"
```

---

## Task 4: Update types.ts (if needed)

**Files:**
- Modify: `src/components/types.ts`

- [ ] **Step 1: Check if ContainerProvider interface exists**

Read the file to see if `ContainerProvider` interface needs to be updated with scale field.

Current interface (around line 20-30):
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

- [ ] **Step 2: Add scale to ContainerProvider interface if it exists**

If the interface exists and is being used, add:

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

Note: Make it optional with `?` since existing code may not use it.

- [ ] **Step 3: Run development server and verify no errors**

Run: `npm run dev`
Expected: No TypeScript errors

- [ ] **Step 4: Commit**

```bash
git add src/components/types.ts
git commit -m "feat: add scale to ContainerProvider interface"
```

---

## Task 5: Create test example in App.vue

**Files:**
- Modify: `src/App.vue`

- [ ] **Step 1: Add scale control buttons and DraggableContainer**

Replace the entire template with:

```vue
<template>
  <div id="app">
    <div style="margin-bottom: 20px;">
      <button @click="scale = 0.5">50%</button>
      <button @click="scale = 1.0">100%</button>
      <button @click="scale = 1.5">150%</button>
      <button @click="scale = 2.0">200%</button>
      <span style="margin-left: 20px;">Current: {{ scale * 100 }}%</span>
    </div>

    <div style="margin-bottom: 20px;">
      <label>
        <input type="checkbox" v-model="snapToBorder"> Enable border snap
      </label>
    </div>

    <!-- User applies visual scaling here -->
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
            This is a test example
          </Vue3DraggableResizable>
        </DraggableContainer>
      </div>
    </div>
  </div>
</template>
```

- [ ] **Step 2: Add scale and snapToBorder to data()**

Update the data() return object (line 56-65):

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

- [ ] **Step 3: Run development server and test all features**

Run: `npm run dev`

**Test 1: Scale**
- Click 50%, 100%, 150%, 200% buttons
- Expected: Container visually scales, logical coordinates stay same

**Test 2: Border snap (scale = 100%)**
- Check "Enable border snap" checkbox
- Drag component near right edge
- Drag resize handle (right or bottom-right) near right edge
- Expected: Component snaps to edge when close

**Test 3: Dynamic snap threshold (scale = 50%)**
- Set scale to 50%
- Enable border snap
- Resize near edge
- Expected: Snap distance is 20px (10 / 0.5) instead of 10px

**Test 4: Dynamic snap threshold (scale = 200%)**
- Set scale to 200%
- Enable border snap
- Resize near edge
- Expected: Snap distance is 5px (10 / 2.0) instead of 10px

- [ ] **Step 4: Commit**

```bash
git add src/App.vue
git commit -m "test: add comprehensive test example for scale and snap features

Tests:
- Scale control (50%, 100%, 150%, 200%)
- Border snap enable/disable
- Dynamic snap threshold based on scale
- Visual scaling applied by user in parent element
"
```

---

## Task 6: Update README.md

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add canvas scale feature documentation**

Location: After the "Props" section, before "Events" (around line 430)

```markdown
### Canvas Scale Feature

The canvas scale feature allows you to scale the entire container while maintaining logical coordinates. The visual scaling is applied by you in the parent element.

**Usage:**

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
        Content
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

type: `Number`<br>
default: `1.0`<br>
range: `0.5 - 2.0`

Scale factor for the canvas. This value is provided to child components for calculations like snap distance. You should apply the visual scaling yourself using CSS transform.

```
```

- [ ] **Step 2: Add border snap feature documentation**

Location: After the canvas scale documentation

```markdown
### Border Snap Feature

Automatically snap component edges to container boundaries when resizing. The snap distance automatically adjusts based on the scale factor.

**Usage:**

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
      Content
    </Vue3DraggableResizable>
  </DraggableContainer>
</template>
```

**Vue3DraggableResizable Props:**

#### snapToBorder

type: `Boolean`<br>
default: `false`

Enable automatic snapping to container borders when resizing.

#### snapThreshold

type: `Number`<br>
default: `10`<br>

Base snap distance threshold in pixels. The actual threshold is calculated as `snapThreshold / scale`, so snapping works correctly at different zoom levels.

Example:
- scale = 1.0, snapThreshold = 10 → snap at 10px
- scale = 0.5, snapThreshold = 10 → snap at 20px
- scale = 2.0, snapThreshold = 10 → snap at 5px
```
```

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add canvas scale and border snap feature documentation

- Document scale prop for DraggableContainer
- Document snapToBorder and snapThreshold props for Vue3DraggableResizable
- Add usage examples
- Explain dynamic snap threshold calculation
"
```

---

## Task 7: Build and verify library

**Files:**
- Build output: `dist/Vue3DraggableResizable.js`

- [ ] **Step 1: Build the library**

Run: `npm run build`
Expected: Build succeeds without errors, generates dist files

- [ ] **Step 2: Check TypeScript declarations**

Run: `npm run build:dts`
Expected: Type definitions generated successfully in `typings/`

- [ ] **Step 3: Commit**

```bash
git add -A
git commit -m "build: verify successful library build with new features"
```

---

## Task 8: Final testing and edge cases

**Files:**
- Manual testing in browser

- [ ] **Step 1: Test scale validation**

Test values outside [0.5, 2.0] range in App.vue:

```javascript
scale: 0.1  // Should warn and use default
scale: 3.0  // Should warn and use default
```

Expected: Vue warns in console, uses default 1.0

- [ ] **Step 2: Test rapid mouse movements**

Test: Quickly drag resize handle past the snap zone
Expected: Snap still triggers (mouse events fire frequently enough)

- [ ] **Step 3: Test with multiple components**

Modify App.vue to have multiple Vue3DraggableResizable components
Expected: Each works independently, snap works for all

- [ ] **Step 4: Test backward compatibility**

Create a test without new props (no scale, no snapToBorder):
```vue
<DraggableContainer>
  <Vue3DraggableResizable v-model:x="x" v-model:y="y" v-model:w="w" v-model:h="h">
    Old usage
  </Vue3DraggableResizable>
</DraggableContainer>
```

Expected: Works exactly as before (scale=1.0, snapToBorder=false)

- [ ] **Step 5: Update CLAUDE.md feature status**

Edit CLAUDE.md line 67:
```markdown
**当前功能状态**：
- ✅ 组件拖拽功能（带参考线对齐）
- ✅ 组件调整大小功能
- ✅ 画布缩放功能（已实现）
- ✅ 组件大小调整时的边界吸附（已实现）
```

- [ ] **Step 6: Final commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md feature status

Mark canvas scale and border snap features as completed.
All tests pass, backward compatibility verified.
"
```

---

## Implementation Complete!

**Summary:**
- ✅ DraggableContainer provides scale via prop and inject
- ✅ User controls visual scaling with CSS transform
- ✅ Vue3DraggableResizable has snapToBorder and snapThreshold props
- ✅ Smart border snapping with dynamic threshold based on scale
- ✅ Comprehensive test example in App.vue
- ✅ Documentation in README.md
- ✅ Backward compatible (all new props have defaults)
- ✅ Library builds successfully

**Next Steps:**
1. Tag release: `git tag -a v1.7.0 -m "Add canvas scale and smart border snap features"`
2. Push to GitHub: `git push origin main --tags`
3. Publish to npm: `npm publish`
