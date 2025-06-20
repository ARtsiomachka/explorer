<template>
  <div class="split-pane-container" ref="container">
    <slot></slot>
  </div>
</template>

<script>
export default { name: "split-pane-container" };
</script>

<script setup>
import { ref, onMounted, onUnmounted, provide, nextTick, watch } from 'vue';

const container = ref(null);
const panes = ref([]);
const resizers = ref([]);

provide('splitPaneContainer', {
  registerPane: (paneProps) => {
    panes.value.push(paneProps);
  },
  unregisterPane: (paneProps) => {
    const index = panes.value.indexOf(paneProps);
    if (index > -1) {
      panes.value.splice(index, 1);
    }
  }
});

const createResizer = (leftPane, rightPane, index) => {
  const resizer = document.createElement('div');
  resizer.className = 'split-pane-resizer';

  let isResizing = false;
  let startX = 0;
  let startLeftWidth = 0;
  let startRightWidth = 0;
  let containerWidth = 0;

  const onMouseDown = (e) => {
    isResizing = true;
    startX = e.clientX;
    startLeftWidth = leftPane.offsetWidth;
    startRightWidth = rightPane.offsetWidth;
    containerWidth = container.value.offsetWidth;

    // Calculate total width of all resizers
    const resizerWidth = resizers.value.reduce((acc, r) => acc + r.offsetWidth, 0);
    containerWidth -= resizerWidth;

    // CRITICAL FIX: Capture the original widths of ALL panes at the start of drag
    const allPanes = Array.from(container.value.children).filter(
      child => child.classList.contains('split-pane')
    );

    // Store original widths for all panes - this prevents other panes from changing
    window.originalPaneWidths = allPanes.map(pane => ({
      pane: pane,
      originalWidth: pane.offsetWidth
    }));

    // Debug logging
    console.log('Resize start:', {
      leftPane: leftPane.className || 'no-class',
      rightPane: rightPane.className || 'no-class',
      startLeftWidth,
      startRightWidth,
      containerWidth,
      leftPaneIndex: Array.from(container.value.children).indexOf(leftPane),
      rightPaneIndex: Array.from(container.value.children).indexOf(rightPane),
      allPaneWidths: window.originalPaneWidths.map(({ originalWidth }) => originalWidth)
    });

    document.addEventListener('mousemove', onMouseMove);
    document.addEventListener('mouseup', onMouseUp);
    document.body.style.cursor = 'col-resize';
    document.body.style.userSelect = 'none';

    e.preventDefault();
  };

  const onMouseMove = (e) => {
    if (!isResizing) return;

    const deltaX = e.clientX - startX;

    // Get all panes to ensure we maintain total width
    const allPanes = Array.from(container.value.children).filter(
      child => child.classList.contains('split-pane')
    );

    // Get current widths of all panes
    const currentWidths = allPanes.map(pane => parseInt(pane.style.width) || pane.offsetWidth);
    const totalCurrentWidth = currentWidths.reduce((sum, width) => sum + width, 0);

    // Get actual minimum widths from each pane
    const leftMinWidth = parseInt(window.getComputedStyle(leftPane).minWidth) || 200;
    const rightMinWidth = parseInt(window.getComputedStyle(rightPane).minWidth) || 200;

    // Calculate what the new widths would be
    let proposedLeftWidth = startLeftWidth + deltaX;
    let proposedRightWidth = startRightWidth - deltaX;

    // Enforce minimum width constraints
    if (proposedLeftWidth < leftMinWidth) {
      proposedLeftWidth = leftMinWidth;
      proposedRightWidth = (startLeftWidth + startRightWidth) - leftMinWidth;
    }
    if (proposedRightWidth < rightMinWidth) {
      proposedRightWidth = rightMinWidth;
      proposedLeftWidth = (startLeftWidth + startRightWidth) - rightMinWidth;
    }

    // Double check that both are still valid after adjustments
    if (proposedLeftWidth < leftMinWidth || proposedRightWidth < rightMinWidth) {
      return; // Can't satisfy both constraints
    }

    // Calculate the change in total width
    const newTotalForTwoPanes = proposedLeftWidth + proposedRightWidth;
    const originalTotalForTwoPanes = startLeftWidth + startRightWidth;
    const widthDifference = newTotalForTwoPanes - originalTotalForTwoPanes;

    // If there's a difference, we need to maintain the total container width
    if (Math.abs(widthDifference) > 1) {
      // Adjust to maintain exact total
      const adjustment = widthDifference / 2;
      proposedLeftWidth -= adjustment;
      proposedRightWidth -= adjustment;
    }

    // Apply the new widths to only the two panes being resized
    const leftIndex = allPanes.indexOf(leftPane);
    const rightIndex = allPanes.indexOf(rightPane);

    console.log('Applying widths:', {
      leftIndex,
      rightIndex,
      proposedLeftWidth,
      proposedRightWidth,
      allPanesCount: allPanes.length
    });

    allPanes.forEach((pane, index) => {
      let width;
      if (index === leftIndex) {
        width = proposedLeftWidth;
      } else if (index === rightIndex) {
        width = proposedRightWidth;
      } else {
        // FIXED: Use the original width captured at drag start, not current width
        const originalPaneData = window.originalPaneWidths.find(data => data.pane === pane);
        width = originalPaneData ? originalPaneData.originalWidth : pane.offsetWidth;
      }

      console.log(`Pane ${index}: setting width to ${width}px`);

      pane.style.width = width + 'px';
      pane.style.flexGrow = '0';
      pane.style.flexShrink = '0';
      pane.style.flexBasis = width + 'px';
    });
  };

  const onMouseUp = () => {
    isResizing = false;
    document.removeEventListener('mousemove', onMouseMove);
    document.removeEventListener('mouseup', onMouseUp);
    document.body.style.cursor = '';
    document.body.style.userSelect = '';

    // Clean up the original widths data
    if (window.originalPaneWidths) {
      delete window.originalPaneWidths;
    }

    console.log('Drag ended');
  };

  resizer.addEventListener('mousedown', onMouseDown);

  return resizer;
};

const setupResizers = () => {
  // Clear existing resizers
  resizers.value.forEach(resizer => resizer.remove());
  resizers.value = [];

  const paneElements = Array.from(container.value.children).filter(
    child => child.classList.contains('split-pane')
  );

  // If only one pane, no resizers needed
  if (paneElements.length <= 1) {
    // Single pane should fill the container
    if (paneElements.length === 1) {
      paneElements[0].style.width = '100%';
      paneElements[0].style.flex = 'none';
    }
    return;
  }

  // Calculate container width and setup initial pane sizes
  const containerWidth = container.value.offsetWidth;
  const resizerWidth = 6 * (paneElements.length - 1); // Width for all future resizers
  const availableWidth = containerWidth - resizerWidth;

  console.log('Setup resizers:', {
    containerWidth,
    availableWidth,
    paneCount: paneElements.length,
    registeredPanes: panes.value.length,
    paneProps: panes.value.map(props => ({ width: props.width, minWidth: props.minWidth })),
    paneWidths: paneElements.map(pane => ({
      offsetWidth: pane.offsetWidth,
      styleWidth: pane.style.width,
      computedWidth: parseInt(window.getComputedStyle(pane).width)
    }))
  });

  let totalFixedWidth = 0;
  let fixedPanes = [];
  let flexPanes = [];

  // Categorize panes into fixed and flexible using the registered props
  paneElements.forEach((pane, index) => {
    // Get the corresponding registered props for this pane
    const paneProps = panes.value[index];
    let explicitWidth = null;

    if (paneProps && paneProps.width !== null && paneProps.width !== undefined) {
      // Use the original :width prop value
      explicitWidth = typeof paneProps.width === 'number' ? paneProps.width : parseInt(paneProps.width);
      console.log(`Pane ${index} has explicit width from props: ${explicitWidth}px`);
    }

    if (explicitWidth && explicitWidth > 0) {
      totalFixedWidth += explicitWidth;
      fixedPanes.push({ pane, width: explicitWidth });
      console.log(`Pane marked as fixed: ${explicitWidth}px`);
    } else {
      flexPanes.push(pane);
      console.log(`Pane marked as flexible: computed ${parseInt(window.getComputedStyle(pane).width)}px`);
    }
  });

  // Handle case where total fixed width exceeds available width
  if (totalFixedWidth > availableWidth && fixedPanes.length > 0) {
    // Scale down fixed panes proportionally to fit
    const scaleFactor = availableWidth / totalFixedWidth;
    fixedPanes.forEach(({ pane, width }) => {
      const scaledWidth = Math.max(
        parseInt(window.getComputedStyle(pane).minWidth) || 200,
        Math.floor(width * scaleFactor)
      );
      pane.style.width = scaledWidth + 'px';
      pane.style.flexGrow = '0';
      pane.style.flexShrink = '0';
      pane.style.flexBasis = scaledWidth + 'px';
    });
    // Recalculate totalFixedWidth after scaling
    totalFixedWidth = fixedPanes.reduce((sum, { pane }) =>
      sum + parseInt(pane.style.width), 0
    );
  } else {
    // Apply fixed widths as specified
    fixedPanes.forEach(({ pane, width }) => {
      pane.style.width = width + 'px';
      pane.style.flexGrow = '0';
      pane.style.flexShrink = '0';
      pane.style.flexBasis = width + 'px';
    });
  }

  // Distribute remaining width among flexible panes
  if (flexPanes.length > 0) {
    const remainingWidth = Math.max(0, availableWidth - totalFixedWidth);
    const widthPerFlexPane = Math.floor(remainingWidth / flexPanes.length);

    flexPanes.forEach((pane, index) => {
      // Get the minimum width for this specific pane
      const paneMinWidth = parseInt(window.getComputedStyle(pane).minWidth) || 200;

      // Give any remainder to the last flex pane
      const width = index === flexPanes.length - 1
        ? remainingWidth - (widthPerFlexPane * (flexPanes.length - 1))
        : widthPerFlexPane;

      const finalWidth = Math.max(paneMinWidth, width);
      pane.style.width = finalWidth + 'px'; // Ensure minimum width
      pane.style.flexGrow = '0';
      pane.style.flexShrink = '0';
      pane.style.flexBasis = finalWidth + 'px';
    });
  }

  // Create resizers between adjacent panes
  for (let i = 0; i < paneElements.length - 1; i++) {
    const leftPane = paneElements[i];
    const rightPane = paneElements[i + 1];
    const resizer = createResizer(leftPane, rightPane, i);

    resizers.value.push(resizer);
    container.value.insertBefore(resizer, rightPane);
  }
};

const handleWindowResize = () => {
  const paneElements = Array.from(container.value.children).filter(
    child => child.classList.contains('split-pane')
  );

  if (paneElements.length <= 1) {
    if (paneElements.length === 1) {
      paneElements[0].style.width = '100%';
    }
    return;
  }

  // Get current container width and calculate new available width
  const newContainerWidth = container.value.offsetWidth;
  const resizerWidth = resizers.value.length * 6;
  const newAvailableWidth = newContainerWidth - resizerWidth;

  // Get current pane widths and their minimum widths
  const currentWidths = paneElements.map(pane => parseInt(pane.style.width) || pane.offsetWidth);
  const minWidths = paneElements.map(pane => parseInt(window.getComputedStyle(pane).minWidth) || 200);
  const currentTotalWidth = currentWidths.reduce((sum, width) => sum + width, 0);

  // Calculate new widths proportionally
  let newWidths = currentWidths.map((width, index) => {
    const ratio = width / currentTotalWidth;
    return Math.max(minWidths[index], Math.floor(newAvailableWidth * ratio));
  });

  // Adjust if total doesn't match available width (due to rounding and min constraints)
  const totalNewWidth = newWidths.reduce((sum, width) => sum + width, 0);
  if (totalNewWidth !== newAvailableWidth) {
    const difference = newAvailableWidth - totalNewWidth;
    // Add/subtract the difference to the last pane (if it won't violate minimum)
    const lastIndex = newWidths.length - 1;
    const adjustedLastWidth = newWidths[lastIndex] + difference;
    if (adjustedLastWidth >= minWidths[lastIndex]) {
      newWidths[lastIndex] = adjustedLastWidth;
    }
  }

  // Apply new widths
  paneElements.forEach((pane, index) => {
    const width = newWidths[index];
    pane.style.width = width + 'px';
    pane.style.flexGrow = '0';
    pane.style.flexShrink = '0';
    pane.style.flexBasis = width + 'px';
  });

  // Trigger resize event for any external components that need to know about size changes
  // This helps with canvas elements or other components that position themselves based on container sizes
  setTimeout(() => {
    const resizeEvent = new Event('resize');
    window.dispatchEvent(resizeEvent);

    // Also check for specific canvas element and trigger its resize if it exists
    const canvasElement = document.getElementById('canvas');
    if (canvasElement && typeof canvasElement.resize === 'function') {
      canvasElement.resize();
    }

    // Alternative: if canvas has a custom resize event
    if (canvasElement) {
      const canvasResizeEvent = new Event('resize');
      canvasElement.dispatchEvent(canvasResizeEvent);
    }
  }, 0);
};

const initRetryCount = ref(0);
const maxRetries = 50; // Try for up to 5 seconds (50 * 100ms)

const tryInitializeResizers = () => {
  if (!container.value) return false;

  const paneElements = Array.from(container.value.children).filter(
    child => child.classList.contains('split-pane')
  );

  console.log(`Retry ${initRetryCount.value}: Found ${paneElements.length} panes, container width: ${container.value.offsetWidth}`);

  // Check if we have multiple panes (single pane doesn't need resizers) and container has proper dimensions
  if (paneElements.length > 1 && container.value.offsetWidth > 10) {
    console.log('Initializing resizers...');
    setupResizers();
    return true; // Successfully initialized
  } else if (paneElements.length === 1 && container.value.offsetWidth > 10) {
    // Single pane case - just set it to full width, but keep retrying in case more panes appear
    paneElements[0].style.width = '100%';
    paneElements[0].style.flex = 'none';
    console.log('Single pane found, continuing to wait for more panes...');
  }

  // If we haven't found enough panes yet but haven't exceeded retries, try again
  if (initRetryCount.value < maxRetries) {
    initRetryCount.value++;
    setTimeout(tryInitializeResizers, 100);
  } else {
    console.log('Max retries reached, final setup with current panes');
    // Do final setup with whatever panes we have
    setupResizers();
  }

  return false;
};

const resizeObserver = ref(null);

// Watch for changes in the number of panes and reinitialize when needed
const currentPaneCount = ref(0);

watch(
  () => {
    if (!container.value) return 0;
    return Array.from(container.value.children).filter(
      child => child.classList.contains('split-pane')
    ).length;
  },
  (newCount) => {
    if (newCount !== currentPaneCount.value) {
      console.log(`Pane count changed from ${currentPaneCount.value} to ${newCount}, reinitializing...`);
      currentPaneCount.value = newCount;

      // Reset retry count and reinitialize
      initRetryCount.value = 0;
      setTimeout(() => {
        setupResizers();

        // Force a layout update after a brief delay to ensure proper sizing
        setTimeout(() => {
          handleWindowResize();
        }, 50);
      }, 10);
    }
  },
  { flush: 'post' }
);

onMounted(async () => {
  // Wait for Vue to finish rendering
  await nextTick();

  // Start trying to initialize
  setTimeout(tryInitializeResizers, 10);

  // Add window resize listener
  window.addEventListener('resize', handleWindowResize);

  // Add ResizeObserver to catch maximize/minimize and other size changes
  if (window.ResizeObserver && container.value) {
    resizeObserver.value = new ResizeObserver((entries) => {
      // Debounce the resize to avoid excessive calls
      clearTimeout(resizeObserver.value.timeout);
      resizeObserver.value.timeout = setTimeout(() => {
        handleWindowResize();
      }, 100);
    });

    resizeObserver.value.observe(container.value);
  }
});

onUnmounted(() => {
  resizers.value.forEach(resizer => {
    resizer.remove();
  });

  // Clean up ResizeObserver
  if (resizeObserver.value) {
    clearTimeout(resizeObserver.value.timeout);
    resizeObserver.value.disconnect();
    resizeObserver.value = null;
  }

  // Remove window resize listener
  window.removeEventListener('resize', handleWindowResize);
});
</script>

<style scoped>
.split-pane-container {
  display: flex;
  height: 100%;
  width: 100%;
  overflow: hidden;
  /* Prevent overflow */
  box-sizing: border-box;
}

:deep(.split-pane) {
  min-width: 200px;
  height: 100%;
  overflow: hidden;
  box-sizing: border-box;
  flex-shrink: 0;
  flex-grow: 0;
}

:deep(.split-pane-resizer) {
  width: 6px;
  background-color: transparent;
  cursor: col-resize;
  flex-shrink: 0;
  position: relative;
  z-index: 1;
}

:deep(.split-pane-resizer::before) {
  content: '';
  position: absolute;
  left: 50%;
  top: 0;
  bottom: 0;
  width: 1px;
  background-color: var(--border, #ddd);
  transform: translateX(-50%);
}

:deep(.split-pane-resizer:hover::before) {
  background-color: var(--primary, #007acc);
  width: 2px;
}

:deep(.split-pane-resizer:hover) {
  background-color: rgba(0, 122, 204, 0.1);
}
</style>