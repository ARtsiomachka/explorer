<template>
  <div class="split-pane" :style="paneStyle">
    <slot></slot>
  </div>
</template>

<script>
export default { name: "split-pane-item" };
</script>

<script setup>
import { computed, inject, onMounted, onUnmounted } from 'vue';

const props = defineProps({
  width: {
    type: [String, Number],
    default: null
  },
  minWidth: {
    type: [String, Number],
    default: 200
  }
});

const splitPaneContainer = inject('splitPaneContainer', null);

const paneStyle = computed(() => {
  let style = {};

  if (props.width) {
    style.flex = 'none';
    style.width = typeof props.width === 'number' ? `${props.width}px` : props.width;
  }

  if (props.minWidth) {
    style.minWidth = typeof props.minWidth === 'number' ? `${props.minWidth}px` : props.minWidth;
  }

  return style;
});

onMounted(() => {
  if (splitPaneContainer && splitPaneContainer.registerPane) {
    splitPaneContainer.registerPane(props);
  }
});

onUnmounted(() => {
  if (splitPaneContainer && splitPaneContainer.unregisterPane) {
    splitPaneContainer.unregisterPane(props);
  }
});
</script>

<style scoped>
.split-pane {
  height: 100%;
  overflow: hidden;
}

/* Ensure all direct children of split-pane take full height */
.split-pane> :deep(*) {
  height: 100% !important;
}
</style>