# Canvas Scale and Border Snap Design

**Date**: 2026-05-06
**Status**: Approved
**Author**: Claude with user collaboration

## Overview

This document describes the design for two new features to be added to the vue3-draggable-resizable component library:

1. **Canvas Scale Feature**: Allow users to scale the entire DraggableContainer and all its child components (50%-200%)
2. **Smart Border Snap Feature**: Automatically snap component edges to container boundaries when resizing, with intelligent threshold based on scale

## Requirements

### Canvas Scale Feature
- Support canvas zoom in/out (50% - 200%)
- Provide API interface via props
- Maintain dragging and alignment functionality during scaling
- Use CSS transform for visual scaling (logical coordinates remain unchanged)

### Border Snap Feature
- Auto-snap to container boundaries when resizing components
- Control via props (snapToBorder)
- Intelligent snap threshold that adjusts based on scale
- Only snap to right and bottom boundaries

## Architecture

### Design Approach: Centralized Scale Management

We use a centralized approach where `DraggableContainer` manages the scale state and provides it to all child components via Vue's `provide/inject` mechanism.

**Component Relationship**:
```
DraggableContainer (provide scale, containerSize)
  └── Vue3DraggableResizable (inject scale, containerSize)
       └── hooks.ts (initResizeHandle uses scale for smart snap threshold)
```

**Data Flow**:
1. Parent component passes `:scale="0.8"` prop to DraggableContainer
2. DraggableContainer validates scale range (0.5-2.0), provides to children
3. DraggableContainer applies CSS `transform: scale(scale)` to root element
4. Vue3DraggableResizable injects scale
5. hooks.ts resize logic calculates dynamic snap distance: `baseThreshold / scale`

## Component Design

### DraggableContainer Modifications

**New Props**:
```typescript
scale: {
  type: Number,
  default: 1.0,
  validator: (value: number) => {
    return value >= 0.5 && value <= 2.0
  }
}
```

**Provide Logic**:
```typescript
provide('scale', toRef(props, 'scale'))
```

**Style Handling**:
- Apply inline style to root element: `transform: scale(scale)`
- Set `transform-origin: top left` for consistent scaling
- Add `transition` property for smooth scaling (configurable via default)

### Vue3DraggableResizable Modifications

**New Props**:
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

**Inject Logic**:
```typescript
const scale = inject<Ref<number>>('scale', ref(1.0))
```

**Pass to Hooks**:
- Pass `scale`, `snapToBorder`, and `snapThreshold` to `initResizeHandle` function

### hooks.ts Modifications (initResizeHandle)

**Core Logic - Smart Border Snap**:

1. **Calculate Dynamic Snap Threshold**:
```typescript
const dynamicThreshold = props.snapThreshold / scale.value
```

2. **Detect Boundary Proximity**:
```typescript
const distanceToRight = parentSize.width - (left + width)
const distanceToBottom = parentSize.height - (top + height)
```

3. **Trigger Snap**:
```typescript
if (snapToBorder && distanceToRight <= dynamicThreshold) {
  width = parentSize.width - left
}
if (snapToBorder && distanceToBottom <= dynamicThreshold) {
  height = parentSize.height - top
}
```

4. **Apply to Specific Handles**:
Snap logic only activates when resizing towards right or bottom directions (handles: `mr`, `br`, `bm`, `tr`, `bl`)

### types.ts Modifications

**Type Extensions**:
- Extend `ContainerProvider` interface to include `scale` field if needed
- Ensure proper TypeScript typing for new props

## Data Flow Examples

### Example 1: User Sets Scale to 0.5

```
1. Parent component passes :scale="0.5"
   ↓
2. DraggableContainer receives and validates scale
   ↓
3. DraggableContainer.provide('scale', ref(0.5))
   ↓
4. DraggableContainer applies style { transform: 'scale(0.5)' }
   ↓
5. Vue3DraggableResizable.inject('scale') → ref(0.5)
   ↓
6. hooks.ts initResizeHandle receives scale
   ↓
7. Calculates dynamic snap threshold: 10 / 0.5 = 20px
   ↓
8. Component visually shrinks by 50%, logical coordinates unchanged
```

### Example 2: User Resizes Component Near Boundary

```
Scenario: Container 800x600, Component at (100, 100), current size 100x100,
user drags bottom-right handle

1. During drag, component right edge reaches 700px (100px from right boundary)
2. scale = 0.5, dynamicThreshold = 10 / 0.5 = 20px
3. When distance <= 20px, snap triggers
4. Component width automatically adjusts to 700px (snaps to right boundary)
5. User releases mouse, resize-end event fires
```

## Interaction Design

### Scaling Behavior
- Default 0.2s transition animation for smooth UX
- Configurable via future prop (not in initial implementation)
- Logical coordinates (x, y, w, h) remain unchanged
- Only visual representation changes via CSS transform

### Snap Behavior
- Snap triggers only during resize (not during drag/move)
- Snap is instant (no transition animation)
- Only right and bottom boundaries are checked
- `resizing` and `resize-end` events include post-snap values

### Compatibility with Existing Features
- ✅ Drag functionality: Unaffected (logical coordinates unchanged)
- ✅ Reference line alignment: Continues to work, snap distance also scale-aware
- ✅ parent prop: Container restriction still applies
- ✅ lockAspectRatio: No conflict with snap feature

## Edge Cases and Error Handling

### Input Validation
- **scale outside [0.5, 2.0]**: Vue validator intercepts, console warning, defaults to 1.0
- **scale <= 0**: Validator intercepts
- **snapThreshold <= 0**: Validator intercepts
- Invalid props are handled by Vue's built-in validation

### Boundary Scenarios
- **Rapid mouse movement past snap zone**: Continuous mouse events ensure at least one frame will trigger snap
- **Component initially outside boundary**: `parent` prop handles initialization, snap doesn't handle initial placement
- **Multiple components scaling**: Each component independently receives scale via inject

## Testing Strategy

### Unit Tests

**DraggableContainer**:
- ✅ scale prop validation (0.5-2.0 range)
- ✅ scale correctly provided via provide/inject
- ✅ CSS transform correctly applied to root element
- ✅ transform-origin set to top left

**Vue3DraggableResizable**:
- ✅ snapToBorder prop correctly passed to hooks
- ✅ snapThreshold prop validation
- ✅ scale correctly obtained via inject

**hooks.ts initResizeHandle**:
- ✅ Dynamic snap threshold calculation correct
- ✅ Boundary detection logic correct
- ✅ Snap only activates when enabled
- ✅ Only applies to specific handle directions

### Integration Tests

- ✅ Scale to 50%, drag component still works
- ✅ Scale to 50%, reference line alignment works
- ✅ Scale to 50%, resize snap distance correct (20px)
- ✅ Scale to 200%, resize snap distance correct (5px)
- ✅ Enable snapToBorder, resize to right boundary triggers snap
- ✅ Enable snapToBorder, resize to bottom boundary triggers snap
- ✅ Disable snapToBorder, no snap occurs
- ✅ Continuous scaling (0.5 → 1.0 → 1.5 → 2.0) without errors

### Manual Tests

- ✅ Rapid dragging of resize handles
- ✅ Multiple components with different snap settings
- ✅ Cross-browser testing (Chrome, Firefox, Safari)
- ✅ Touch events on mobile devices

## Implementation Plan

### Phase 1: Canvas Scale Feature (High Priority)
1. Add scale prop to DraggableContainer
2. Implement provide logic
3. Apply CSS transform
4. Basic testing

### Phase 2: Smart Border Snap (High Priority)
1. Add snapToBorder and snapThreshold props to Vue3DraggableResizable
2. Implement dynamic snap threshold calculation in hooks.ts
3. Implement boundary detection and snap logic
4. Integration testing

### Phase 3: Optimization and Documentation (Medium Priority)
1. Add unit tests
2. Update README.md
3. Add usage examples
4. Performance optimization if needed

## Backward Compatibility

All changes are **100% backward compatible**:
- All new props have default values
- Default scale=1.0 (no visual change)
- Default snapToBorder=false (no snap behavior)
- Existing functionality completely unaffected

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Scale affects drag experience | Medium | Logical coordinates unchanged, drag logic unaffected |
| Snap conflicts with parent prop | Low | Both restrict to container, logic is consistent |
| Transform affects event coordinates | Low | Use logical coordinates, CSS transform doesn't affect mouse events |
| Performance issues | Low | GPU acceleration, calculation only during resize |

## Files to Modify

1. **src/components/DraggableContainer.ts**
   - Add scale prop and validation
   - Implement provide logic
   - Apply CSS transform styles

2. **src/components/Vue3DraggableResizable.ts**
   - Add snapToBorder and snapThreshold props
   - Add inject logic for scale
   - Pass new props to hooks

3. **src/components/hooks.ts**
   - Modify initResizeHandle function
   - Implement dynamic snap threshold calculation
   - Implement boundary detection and snap logic

4. **src/components/types.ts**
   - Extend ContainerProvider interface if needed

5. **README.md**
   - Document new props
   - Add usage examples for scale feature
   - Add usage examples for snap feature

## Usage Examples

### Canvas Scaling

```vue
<template>
  <DraggableContainer :scale="0.8">
    <Vue3DraggableResizable
      v-model:x="x"
      v-model:y="y"
      v-model:w="w"
      v-model:h="h"
    >
      Content
    </Vue3DraggableResizable>
  </DraggableContainer>
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

### Border Snap

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
      Content
    </Vue3DraggableResizable>
  </DraggableContainer>
</template>
```

### Combined Usage

```vue
<template>
  <div>
    <button @click="scale = 0.5">50%</button>
    <button @click="scale = 1.0">100%</button>
    <button @click="scale = 1.5">150%</button>

    <DraggableContainer :scale="scale">
      <Vue3DraggableResizable
        v-model:x="x"
        v-model:y="y"
        v-model:w="w"
        v-model:h="h"
        :snapToBorder="true"
      >
        Drag me and resize to see snap in action!
      </Vue3DraggableResizable>
    </DraggableContainer>
  </div>
</template>
```

## Conclusion

This design provides a clean, maintainable solution for canvas scaling and intelligent border snapping. The centralized architecture aligns with existing patterns in the codebase, and all changes are backward compatible with comprehensive testing coverage.
