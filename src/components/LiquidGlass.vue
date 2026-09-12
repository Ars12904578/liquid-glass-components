<script setup lang="ts">
defineOptions({ inheritAttrs: false });
import { onMounted, onUnmounted, ref, watch, type ComponentPublicInstance } from "vue";

type Props = {
  blur?: number;
  draggable?: boolean;
  refraction?: number;
  centerRefraction?: number;
  bezel?: number;
};

const props = withDefaults(defineProps<Props>(), {
  blur: 0,
  draggable: false,
  bezel: 20,
  refraction: 10,
  centerRefraction: 0.2,
});

function buildDisplacementMap(
  w: number,
  h: number,
  bezel: number,
  centerRefraction: number,
): string {
  const edgeWidth = Math.min(Math.max(Math.abs(bezel), 1), Math.max(Math.min(w, h) / 2, 1));
  const radius = Math.min(26, Math.max(Math.min(w, h) / 2 - 1, 0));
  const halfWidth = w / 2;
  const halfHeight = h / 2;
  const cornerX = Math.max(halfWidth - radius, 0);
  const cornerY = Math.max(halfHeight - radius, 0);

  const maxDim = 64;
  const minShortAxis = 24;
  const baseScale = Math.min(1, maxDim / Math.max(w, h));
  const cw = Math.max(w <= h ? minShortAxis : 1, Math.round(w * baseScale));
  const ch = Math.max(h <= w ? minShortAxis : 1, Math.round(h * baseScale));

  const invScaleX = w / cw;
  const invScaleY = h / ch;

  const canvas = document.createElement("canvas");
  canvas.width = cw;
  canvas.height = ch;
  const ctx = canvas.getContext("2d");
  if (!ctx) return "";

  const imageData = ctx.createImageData(cw, ch);
  const data = imageData.data;

  const clamp = (value: number) => Math.max(0, Math.min(value, 1));
  const smoothstep = (value: number) => value * value * (3 - 2 * value);

  let idx = 0;
  for (let py = 0; py < ch; py++) {
    const sampleY = (py + 0.5) * invScaleY - halfHeight;

    for (let px = 0; px < cw; px++) {
      const sampleX = (px + 0.5) * invScaleX - halfWidth;
      const absX = Math.abs(sampleX);
      const absY = Math.abs(sampleY);
      let distance: number;
      let normalX = 0;
      let normalY = 0;

      if (absX > cornerX && absY > cornerY) {
        const cornerOffsetX = absX - cornerX;
        const cornerOffsetY = absY - cornerY;
        const cornerDistance = Math.hypot(cornerOffsetX, cornerOffsetY);
        distance = radius - cornerDistance;
        if (cornerDistance > 0) {
          normalX = (cornerOffsetX / cornerDistance) * Math.sign(sampleX);
          normalY = (cornerOffsetY / cornerDistance) * Math.sign(sampleY);
        }
      } else if (absX > cornerX) {
        distance = halfWidth - absX;
        normalX = Math.sign(sampleX);
      } else {
        distance = halfHeight - absY;
        normalY = Math.sign(sampleY);
      }

      if (normalX === 0 && normalY === 0) {
        if (halfWidth - absX < halfHeight - absY) {
          normalX = Math.sign(sampleX);
        } else {
          normalY = Math.sign(sampleY);
        }
      }

      const edgeProgress = clamp(1 - distance / edgeWidth);
      const edgeFactor = smoothstep(smoothstep(edgeProgress));
      const centerStrength = clamp(centerRefraction);
      const centerX = sampleX / Math.max(halfWidth, 1);
      const centerY = sampleY / Math.max(halfHeight, 1);
      const centerFade = 1 - edgeFactor;
      const displacementX = normalX * edgeFactor + centerX * centerStrength * centerFade;
      const displacementY = normalY * edgeFactor + centerY * centerStrength * centerFade;

      data[idx++] = Math.round(128 + displacementX * 127 * 0.92); // R -> x displacement
      data[idx++] = Math.round(128 + displacementY * 127 * 0.92); // G -> y displacement
      data[idx++] = 128; // B (unused)
      data[idx++] = 255; // A
    }
  }

  ctx.putImageData(imageData, 0, 0);
  return canvas.toDataURL("image/png");
}


function createFilterId() {
  let id = "";
  do {
    id = `lg-filter-${globalThis.crypto?.randomUUID?.() ?? Math.random().toString(36).slice(2)}`;
  } while (typeof document !== "undefined" && document.getElementById(id));
  return id;
}

const filterId = createFilterId();
const glassEl = ref<HTMLElement | null>(null);
const lgMap = ref<SVGFEImageElement | null>(null);
const lgFilter = ref<SVGFilterElement | null>(null);
const supportsSvgBackdropFilter =
  typeof CSS !== "undefined" &&
  (CSS.supports("backdrop-filter", "url(#liquid-glass-filter)") ||
    CSS.supports("-webkit-backdrop-filter", "url(#liquid-glass-filter)"));

let resizeObserver: ResizeObserver | null = null;
let intersectionObserver: IntersectionObserver | null = null;
let rafId = 0;
let resizeDebounceId = 0;

let isIntersecting = false;
let renderPending = false;

let lastObservedWidth = 0;
let lastObservedHeight = 0;

let lastRenderedWidth = 0;
let lastRenderedHeight = 0;
let dragStartX = 0;
let dragStartY = 0;
let dragOffsetX = 0;
let dragOffsetY = 0;
let isDragging = false;

function setDragOffset() {
  glassEl.value?.style.setProperty("--glass-drag-x", `${dragOffsetX}px`);
  glassEl.value?.style.setProperty("--glass-drag-y", `${dragOffsetY}px`);
}

function onPointerDown(event: PointerEvent) {
  if (!props.draggable || !glassEl.value) return;
  isDragging = true;
  dragStartX = event.clientX - dragOffsetX;
  dragStartY = event.clientY - dragOffsetY;
  glassEl.value.setPointerCapture(event.pointerId);
  event.preventDefault();
}

function onPointerMove(event: PointerEvent) {
  if (!isDragging) return;
  dragOffsetX = event.clientX - dragStartX;
  dragOffsetY = event.clientY - dragStartY;
  setDragOffset();
}

function stopDragging(event: PointerEvent) {
  if (!isDragging) return;
  isDragging = false;
  if (glassEl.value?.hasPointerCapture(event.pointerId)) {
    glassEl.value.releasePointerCapture(event.pointerId);
  }
}

function updateFilter(width: number, height: number) {
  const w = Math.round(width);
  const h = Math.round(height);
  if (w <= 0 || h <= 0) return;

  lastRenderedWidth = w;
  lastRenderedHeight = h;
  renderPending = false;

  const dataUri = buildDisplacementMap(w, h, props.bezel, props.centerRefraction);
  const map = lgMap.value;
  const filter = lgFilter.value;
  if (!map || !filter || !dataUri) return;
  const filterPadding = Math.max(Math.abs(props.refraction), 1);
  map.setAttribute("width", String(w));
  map.setAttribute("height", String(h));
  map.setAttributeNS("http://www.w3.org/1999/xlink", "href", dataUri);
  map.setAttribute("href", dataUri);
  filter.setAttribute("x", String(-filterPadding));
  filter.setAttribute("y", String(-filterPadding));
  filter.setAttribute("width", String(w + filterPadding * 2));
  filter.setAttribute("height", String(h + filterPadding * 2));
}

function scheduleRender(width: number, height: number) {
  cancelAnimationFrame(rafId);
  rafId = requestAnimationFrame(() => updateFilter(width, height));
}

function requestRender(width: number, height: number) {
  if (!isIntersecting) {
    renderPending = true;
    return;
  }
  scheduleRender(width, height);
}

function requestResizeRender(width: number, height: number) {
  cancelAnimationFrame(rafId);
  cancelAnimationFrame(resizeDebounceId);
  resizeDebounceId = requestAnimationFrame(() => scheduleRender(width, height));
}

function setGlassEl(el: Element | ComponentPublicInstance | null) {
  glassEl.value = el instanceof HTMLElement ? el : null;
}

function setLgMap(el: Element | ComponentPublicInstance | null) {
  lgMap.value = el instanceof SVGFEImageElement ? el : null;
}

function setLgFilter(el: Element | ComponentPublicInstance | null) {
  lgFilter.value = el instanceof SVGFilterElement ? el : null;
}

function getBorderBoxSize(entry: ResizeObserverEntry) {
  const borderBoxSize = entry.borderBoxSize;
  if (borderBoxSize) {
    const size = Array.isArray(borderBoxSize) ? borderBoxSize[0] : borderBoxSize;
    return { width: size.inlineSize, height: size.blockSize };
  }

  const element = entry.target as HTMLElement;
  const bounds = element.getBoundingClientRect();
  return { width: bounds.width, height: bounds.height };
}

onMounted(() => {
  if (!glassEl.value) return;

  resizeObserver = new ResizeObserver((entries) => {
    for (const entry of entries) {
      const size = getBorderBoxSize(entry);
      const w = Math.round(size.width);
      const h = Math.round(size.height);

      if (w === lastObservedWidth && h === lastObservedHeight) continue;
      lastObservedWidth = w;
      lastObservedHeight = h;

      requestResizeRender(w, h);
    }
  });
  resizeObserver.observe(glassEl.value, { box: "border-box" });

  intersectionObserver = new IntersectionObserver(
    (entries) => {
      const entry = entries[entries.length - 1];
      if (!entry) return;
      isIntersecting = entry.isIntersecting;

      if (
        isIntersecting &&
        (renderPending ||
          lastRenderedWidth !== lastObservedWidth ||
          lastRenderedHeight !== lastObservedHeight)
      ) {
        scheduleRender(lastObservedWidth, lastObservedHeight);
      }
    },
    { rootMargin: "200px" },
  );
  intersectionObserver.observe(glassEl.value);

  const initialBounds = glassEl.value.getBoundingClientRect();
  const initialWidth = initialBounds.width;
  const initialHeight = initialBounds.height;
  lastObservedWidth = Math.round(initialWidth);
  lastObservedHeight = Math.round(initialHeight);
});

watch(
  () => [props.bezel, props.refraction, props.centerRefraction],
  () => {
    if (lastObservedWidth && lastObservedHeight) {
      requestRender(lastObservedWidth, lastObservedHeight);
    }
  },
);

onUnmounted(() => {
  resizeObserver?.disconnect();
  intersectionObserver?.disconnect();
  cancelAnimationFrame(rafId);
  cancelAnimationFrame(resizeDebounceId);
});

const glass = { filterId, setGlassEl, setLgMap, setLgFilter };
</script>

<template>
  <div
    :ref="glass.setGlassEl"
    class="glass-liquid select-none flex items-center justify-center"
    v-bind="$attrs"
    @pointerdown="onPointerDown"
    @pointermove="onPointerMove"
    @pointerup="stopDragging"
    @pointercancel="stopDragging"
    :style="{
      backdropFilter: supportsSvgBackdropFilter
        ? `blur(${props.blur}px) url(#${glass.filterId})`
        : `blur(${props.blur}px)`,
      WebkitBackdropFilter: supportsSvgBackdropFilter
        ? `blur(${props.blur}px) url(#${glass.filterId})`
        : `blur(${props.blur}px)`,
    }"
  >
    <slot />
  </div>

  <svg width="0" height="0" style="position: absolute" aria-hidden="true">
    <filter
      :ref="glass.setLgFilter"
      :id="glass.filterId"
      x="0"
      y="0"
      width="0"
      height="0"
      filterUnits="userSpaceOnUse"
      primitiveUnits="userSpaceOnUse"
      color-interpolation-filters="sRGB"
    >
      <feImage
        :ref="glass.setLgMap"
        x="0"
        y="0"
        width="0"
        height="0"
        preserveAspectRatio="none"
        result="displacementMap"
      />
      <feDisplacementMap
        in="SourceGraphic"
        in2="displacementMap"
        :scale="props.refraction"
        xChannelSelector="R"
        yChannelSelector="G"
      />
    </filter>
  </svg>
</template>

<style scoped>
.glass-liquid {
  box-sizing: border-box;
  translate: var(--glass-drag-x, 0px) var(--glass-drag-y, 0px);
  cursor: v-bind("props.draggable ? 'grab' : 'auto'");
  touch-action: v-bind("props.draggable ? 'none' : 'auto'");
  box-shadow:
    inset 0 0 0 1px rgba(255, 255, 255, 0.1),
    inset 1.5px 1.5px 0 rgba(255, 255, 255, 0.1),
    inset 0 0 12px rgba(255, 255, 255, 0.1),
    0 8px 32px rgba(0, 0, 0, 0.1);
}

.glass-liquid {
  will-change: auto;
}
</style>
