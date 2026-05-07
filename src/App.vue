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

<script>
import { defineComponent } from "vue";
import Vue3DraggableResizable from "./components/Vue3DraggableResizable";
import DraggableContainer from "./components/DraggableContainer";

export default defineComponent({
  components: { DraggableContainer, Vue3DraggableResizable },
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
  mounted() {},
  methods: {
    print(val, e) {
      console.log(val, e)
    },
  },
});
</script>
<style lang="less" scoped>
.parent {
  width: 600px;
  height: 600px;
  position: relative;
  border: 1px solid #000;
  user-select: none;
  ::v-deep {
    .vdr-container {
      border-color: #999;
    }
  }
}
</style>
