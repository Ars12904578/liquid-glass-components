<script setup lang="ts">
import LiquidGlass from "@/components/LiquidGlass.vue";
import { Settings } from "@lucide/vue";
import { ref } from "vue";
const isOpened = ref(false);
function toggle() {
  console.log("clicked!");
  isOpened.value = !isOpened.value;
}
</script>

<template>
  <img
    src="/bg2.jpg"
    alt="bg"
    class="fixed inset-0 w-full h-full object-cover -z-1 brightness-50"
  />

  <div class="fixed inset-0 flex flex-col items-center justify-center">
    <div class="flex-1 w-full h-full grid grid-cols-3">
      <div v-for="value in 9" :key="value" class="flex items-center justify-center w-full h-full">
        <LiquidGlass
          :blur="2"
          draggable
          class="transition-none animate-none w-20 h-20 rounded-4xl bg-white/1"
          @click="toggle"
        >
          <Settings :size="40" class="text-white" />
        </LiquidGlass>
      </div>
    </div>
    <div class="w-full h-30 p-3 flex items-end justify-center">
      <LiquidGlass
        :blur="1"
        :center-refraction="0.2"
        :refraction="50"
        :bezel="10"
        class="flex flex-row rounded-4xl items-center justify-evenly gap-3"
        :class="isOpened ? 'closing bg-white/1 w-40 h-3' : 'opening bg-white/1 w-full h-full'"
        @click="toggle"
      >
        <Settings
          v-for="value in 4"
          :key="value"
          :size="40"
          class="text-white"
          :class="isOpened ? 'opacity-0 closing' : 'opacity-100 opening'"
        />
      </LiquidGlass>
    </div>
    <LiquidGlass class="fixed inset-10 rounded-4xl" :center-refraction="0.2" :blur="2" :refraction="200" :bezel="10" draggable/>
  </div>

  <RouterView />
</template>
