<template>
  <div id="page-queries" :class="css">
    <split-pane-container>
      <split-pane-item :width="500" :min-width="300" v-if="app_params.sidebar">
        <pane-query v-model:app_params="app_params" :conn="conn">
        </pane-query>
      </split-pane-item>

      <split-pane-item>
        <pane-inspect :app_params="app_params" :conn="conn">
        </pane-inspect>
      </split-pane-item>
    </split-pane-container>
  </div>
</template>

<script>
export default { name: "page-queries" };
</script>

<script setup>
import { defineProps, defineModel, computed } from 'vue';

const props = defineProps({
  conn: { type: Object, required: true },
  app_params: { type: Object, required: true },
});

const app_params = defineModel("app_params");

const css = computed(() => {
  let result = ["page-content"];
  if (!app_params.value.sidebar) {
    result.push("page-queries-no-sidebar");
  }
  return result;
});

</script>

<style scoped>
#page-queries {
  height: calc(100vh - var(--header-height) - var(--footer-height) - 3 * var(--gap));
}

@media screen and (max-width: 800px) {
  #page-queries :deep(.split-pane-container) {
    flex-direction: column;
  }
}
</style>

<style>
div.queries-left-pane {
  height: calc(100vh - var(--header-height) - var(--footer-height) - 3 * var(--gap));
}

div.queries-right-pane {
  height: calc(100vh - var(--header-height) - var(--footer-height) - 3 * var(--gap));
}

@media screen and (max-width: 800px) {
  div.queries-left-pane {
    height: calc(40vh - var(--header-height) - var(--gap));
  }

  div.queries-right-pane {
    height: calc(60vh - var(--footer-height) - 3 * var(--gap));
  }
}
</style>
