<script setup lang="ts">
defineOptions({ inheritAttrs: false });
import { onMounted, onUnmounted, ref, watch, type ComponentPublicInstance } from "vue";

function buildDisplacementSVG(w: number, h: number, bezel: number): string {
  const edgeWidth = Math.min(Math.max(Math.abs(bezel), 1), Math.max(Math.min(w, h) / 2, 1));
  const radius = Math.min(26, Math.max(Math.min(w, h) / 2 - 1, 0));
  const halfWidth = w / 2;
  const halfHeight = h / 2;
  const cornerX = Math.max(halfWidth - radius, 0);
  const cornerY = Math.max(halfHeight - radius, 0);
  const step = Math.max(1, Math.ceil(Math.max(w, h) / 96));
  const cells: string[] = [];

  const clamp = (value: number) => Math.max(0, Math.min(value, 1));
  const smoothstep = (value: number) => value * value * (3 - 2 * value);

  for (let y = 0; y < h; y += step) {
    const cellHeight = Math.min(step, h - y);
    const sampleY = y + cellHeight / 2 - halfHeight;

    for (let x = 0; x < w; x += step) {
      const cellWidth = Math.min(step, w - x);
      const sampleX = x + cellWidth / 2 - halfWidth;
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
      const easedStrength = smoothstep(smoothstep(edgeProgress)) * 0.92;
      const red = Math.round(128 + normalX * 127 * easedStrength);
      const green = Math.round(128 + normalY * 127 * easedStrength);

      cells.push(
        `<rect x="${x}" y="${y}" width="${cellWidth}" height="${cellHeight}" fill="rgb(${red},${green},128)"/>`,
      );
    }
  }

  return `<svg xmlns="http://www.w3.org/2000/svg" width="${w}" height="${h}" viewBox="0 0 ${w} ${h}">
    <defs>
      <filter id="soften" x="-10%" y="-10%" width="120%" height="120%">
        <feGaussianBlur stdDeviation="${Math.max(step * 0.35, 0.35)}"/>
      </filter>
    </defs>
    <rect width="${w}" height="${h}" fill="#808080"/>
    <g filter="url(#soften)">${cells.join("")}</g>
  </svg>`;
}

let idCounter = 0;

type Props = {
  blur?: number;
  refraction?: number;
  bezel?: number;
  css?: string;
};

const props = withDefaults(defineProps<Props>(), {
  blur: 0,
  bezel: 20,
  refraction: 40,
  css: "bg-white/1 rounded-xl w-fit p-3 px-10",
});

const filterId = `lg-filter-${idCounter++}`;
const glassEl = ref<HTMLElement | null>(null);
const lgMap = ref<SVGFEImageElement | null>(null);
let resizeObserver: ResizeObserver | null = null;
let lastWidth = 0;
let lastHeight = 0;

function updateFilter(width: number, height: number) {
  lastWidth = width;
  lastHeight = height;
  const w = Math.round(width);
  const h = Math.round(height);
  const dataUri =
    "data:image/svg+xml;utf8," + encodeURIComponent(buildDisplacementSVG(w, h, props.bezel));
  const map = lgMap.value;
  if (!map) return;
  map.setAttribute("width", String(w));
  map.setAttribute("height", String(h));
  map.setAttributeNS("http://www.w3.org/1999/xlink", "href", dataUri);
  map.setAttribute("href", dataUri);
}

function setGlassEl(el: Element | ComponentPublicInstance | null) {
  glassEl.value = el instanceof HTMLElement ? el : null;
}

function setLgMap(el: Element | ComponentPublicInstance | null) {
  lgMap.value = el instanceof SVGFEImageElement ? el : null;
}

function getBorderBoxSize(entry: ResizeObserverEntry) {
  const borderBoxSize = entry.borderBoxSize;
  if (borderBoxSize) {
    const size = Array.isArray(borderBoxSize) ? borderBoxSize[0] : borderBoxSize;
    return { width: size.inlineSize, height: size.blockSize };
  }

  const element = entry.target as HTMLElement;
  return { width: element.offsetWidth, height: element.offsetHeight };
}

onMounted(() => {
  if (!glassEl.value) return;
  resizeObserver = new ResizeObserver((entries) => {
    for (const entry of entries) {
      const size = getBorderBoxSize(entry);
      updateFilter(size.width, size.height);
    }
  });
  resizeObserver.observe(glassEl.value, { box: "border-box" });
  updateFilter(glassEl.value.offsetWidth, glassEl.value.offsetHeight);
});

watch(
  () => props.bezel,
  () => {
    if (lastWidth && lastHeight) updateFilter(lastWidth, lastHeight);
  },
);

onUnmounted(() => {
  resizeObserver?.disconnect();
});

const glass = { filterId, setGlassEl, setLgMap };
</script>

<template>
  <div
    :ref="glass.setGlassEl"
    class="glass-liquid select-none flex items-center justify-center"
    :class="props.css"
    v-bind="$attrs"
    :style="{
      backdropFilter: `blur(${props.blur}px) url(#${glass.filterId})`,
      WebkitBackdropFilter: `blur(${props.blur}px) url(#${glass.filterId})`,
    }"
  >
    <slot />
  </div>

  <svg width="0" height="0" style="position: absolute" aria-hidden="true">
    <filter
      :id="glass.filterId"
      x="0"
      y="0"
      width="100%"
      height="100%"
      color-interpolation-filters="sRGB"
    >
      <feImage :ref="glass.setLgMap" x="0" y="0" result="displacementMap" />
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
  /* background: rgba(255, 255, 255, 0.05); */
  box-shadow:
    inset 0 0 0 1px rgba(255, 255, 255, 0.1),
    inset 1.5px 1.5px 0 rgba(255, 255, 255, 0.1),
    inset 0 0 12px rgba(255, 255, 255, 0.1),
    0 8px 32px rgba(0, 0, 0, 0.1);
}
</style>
