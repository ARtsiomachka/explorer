<template>
  <div id="page-entities" :class="pageCss">
    <split-pane-container>
      <split-pane-item :width="300" :min-width="200" v-if="appParams.sidebar">
        <pane-tree :conn="conn" v-model:app_params="appParams" @scriptOpen="onScriptOpen" @entityOpen="onEntityOpen"
          ref="pane_tree">
        </pane-tree>
      </split-pane-item>

      <split-pane-item v-if="showCanvas">
        <div id="canvasPlaceholder" :class="canvasCss">
        </div>
      </split-pane-item>

      <split-pane-item v-if="showScript">
        <div :class="scriptCss">
          <pane-scripts :conn="conn" v-model:script="appParams.script" v-model:scripts="appParams.scripts"
            ref="pane_scripts">
          </pane-scripts>
        </div>
      </split-pane-item>

      <split-pane-item :width="500" :min-width="300" v-if="showInspector">
        <div class="page-entities-inspector">
          <pane-inspector :conn="conn" :app_params="appParams" @abort="onAbort" @scriptOpen="onScriptOpen">
          </pane-inspector>
        </div>
      </split-pane-item>
    </split-pane-container>
  </div>
</template>

<script>
export default { name: "page-entities" };
</script>

<script setup>
import { defineProps, defineModel, ref, computed, watch, nextTick } from 'vue';

const pane_tree = ref(null);
const pane_scripts = ref(null);

const props = defineProps({
  conn: { type: Object, required: true },
  app_state: { type: Object, required: true },
});

const appParams = defineModel("app_params");

const showScript = computed(() => {
  if (!props.app_state.has3DCanvas) {
    return false;
  } else if (!appParams.value.script) {
    return false;
  } else if (appParams.value.entity.path) {
    return false;
  } else {
    return true;
  }
});

const showCanvas = computed(() => {
  return props.app_state.has3DCanvas;
});

const showInspector = computed(() => {
  if (!appParams.value.entity.path) {
    return false;
  }
  return true;
});

watch(() => [showCanvas.value, showInspector.value, showScript.value, appParams.value.sidebar], () => {
  nextTick(() => {
    var resizeEvent = new Event('resize');
    window.dispatchEvent(resizeEvent);
  });
});

function onAbort(evt) {
  appParams.value.entity.path = undefined;
  pane_tree.value.unselect();
}

function onScriptOpen(path) {
  if (!props.app_state.has3DCanvas) {
    pane_scripts.value.openScript(path);
    appParams.value.entity.path = path;
  } else {
    appParams.value.entity.path = undefined;
    appParams.value.scripts.length = 0;
    pane_scripts.value.openScript(path);
  }
}

function onEntityOpen(path) {
  if (props.app_state.has3DCanvas) {
    pane_scripts.value.closeScripts();
  }
}

const pageCss = computed(() => {
  let classes = ["page-content"];
  if (appParams.value.entity.path || showScript.value) {
    classes.push("page-entities-show-inspector");
  }
  if (appParams.value.sidebar == true) {
    classes.push("page-entities-show-sidebar");
  }
  return classes;
});

const scriptCss = computed(() => {
  let classes = ["page-entities-script"];
  return classes;
})

const canvasCss = computed(() => {
  let classes = ["page-entities-canvas"];
  if (!showCanvas.value) {
    classes.push("page-entities-canvas-hide")
  }
  return classes;
});



</script>

<style scoped>
#page-entities {
  height: calc(100vh - var(--header-height) - var(--footer-height) - 3 * var(--gap));
}

div.page-entities-canvas {
  border-radius: var(--border-radius-medium);
  overflow: hidden;
  height: 100%;
}

div.page-entities-canvas-hide {
  display: none;
}

div.page-entities-script {
  height: 100%;
}

div.page-entities-inspector {
  overflow-x: auto;
  height: 100%;
}

</style>
