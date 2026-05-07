# CLAUDE.md

此文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 项目概述

这是我 clone 的 https://github.com/a7650/vue3-draggable-resizable 这个仓库。

**原始仓库功能**：Vue3 可拖拽、可调整大小、支持组件对齐、自动吸附、参考线对齐的组件库

**扩展目标**：
- 画布放大、缩小功能（保持组件拖拽、对齐功能正常）
- 组件拖拽调整大小时自动吸附到画布边界

## 需求

### 1. 画布缩放功能
- 支持画布的放大、缩小操作
- 提供 API 接口调用（方法级）
- 缩放时组件的拖拽、对齐功能保持正常工作
- 缩放比例 50% - 200%

### 2. 组件自动吸附功能
- 组件拖拽调整大小时自动吸附到画布边界
- 通过 props 控制是否启用此功能

## 技术栈

- **框架**：Vue 3.0.0 (Composition API)
- **语言**：TypeScript 3.9.3
- **构建工具**：Vue CLI 4.5.0 (基于 @vue/cli-service)

## 架构说明

**项目现状**：这是一个已有项目，需要在现有基础上扩展功能

**修改计划**：
- 新增画布缩放相关的 API 接口（可能需要添加到主组件暴露的方法中）
- 新增组件拖拽大小自动吸附功能的 props 控制属性
- 保持向后兼容，不影响现有功能

**关键文件**：

- **主组件**：`src/components/Vue3DraggableResizable.ts` (265行)
  - Props 定义：第 31-131 行
  - 暴露事件：activated, deactivated, drag-start, resize-start, dragging, resizing, drag-end, resize-end, update:w/h/x/y/active

- **容器组件**：`src/components/DraggableContainer.ts` (126行)
  - 提供参考线对齐的上下文环境
  - Props: disabled, adsorbParent, adsorbCols, adsorbRows, referenceLineVisible, referenceLineColor

- **核心逻辑**：`src/components/hooks.ts` (584行)
  - `initDraggableContainer` (第 237-390 行) - 拖拽功能实现
  - `initResizeHandle` (第 392-553 行) - 调整大小功能实现
  - `handleDrag` 函数中的参考线对齐逻辑 (第 294-342 行)

- **工具函数**：`src/components/utils.ts` (92行)
  - `getReferenceLineMap` (第 59-92 行) - 生成参考线映射，实现对齐逻辑

- **类型定义**：`src/components/types.ts` (53行)

**当前功能状态**：
- ✅ 组件拖拽功能（带参考线对齐）
- ✅ 组件调整大小功能
- ✅ 画布缩放功能（已实现）
- ✅ 组件大小调整时的边界吸附（已实现）

## Development Guidelines

- **重要**: 做任何代码改动之前，必须先让用户确认方案，未经确认不得修改代码
- **Testing**: 使用 `test-driven-development` skill 实现新功能
- **Code Style**: 遵循项目现有的代码风格
- **Compatibility**: 确保新功能不影响现有 API 的使用
