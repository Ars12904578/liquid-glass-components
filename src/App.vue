<script setup lang="ts">
import LiquidGlass from "@/components/LiquidGlass.vue";
import {
  Ellipsis,
  Wifi,
  Folder,
  Search,
  CircleUserRound,
  BatteryMedium,
  Globe,
  Bot,
  Monitor,
  Play,
  Music,

} from "@lucide/vue";
import { reactive } from "vue";
const Apps = reactive([
  { app: "This PC", icon: Monitor },
  { app: "Files", icon: Folder },
  { app: "Chrome", icon: Globe },
  { app: "ChatGPT", icon: Bot },
  { app: "Youtube", icon: Play },
  { app: "Spotify", icon: Music }
]);
const PinnedApps = reactive([ Search , Folder, Globe, Bot]);
</script>

<template>
  <img
    src="/bg.jpg"
    alt="bg"
    class="fixed inset-0 w-full h-full object-cover -z-1 brightness-50"
  />

  <!-- Top Window -->
  <div class="fixed inset-5 bottom-25 grid grid-flow-col grid-rows-5 xl:grid-rows-6 grid-cols-10 gap-5">
    <!-- Apps -->
    <div
      v-for="(app, index) in Apps"
      :key="`${app.app}-${index}`"
      class="min-w-0 min-h-0 flex flex-col items-center justify-center"
    >
      <LiquidGlass
        :blur="1"
        class="h-full aspect-square flex items-center justify-center rounded-3xl text-white hover:scale-110 active:scale-90"
      >
        <component :is="app.icon" class="w-1/2 h-1/2" />
      </LiquidGlass>

      <span class="font-medium leading-none text-shadow-lg p-1 rounded-full text-sm text-center text-white">
        {{ app.app }}
      </span>
    </div>
  </div>

  <!-- Bottom Window -->
  <div class="fixed bottom-0 w-full h-20 p-1">
    <!-- Taskbar -->
    <div
      class="flex relative p-2 rounded-full items-center justify-between gap-3 w-full h-full bg-black/50"
    >
      <LiquidGlass
        class="h-full aspect-square px-5 hover:scale-105 active:scale-95 bg-white/1 text-white rounded-full flex items-center justify-center"
      >
        <Ellipsis class="" :size="30" />
      </LiquidGlass>

      <!-- Middle -->
      <div class="absolute left-1/2 -translate-x-1/2 flex h-full w-fit items-center gap-2 py-2">
        <LiquidGlass
          v-for="(app, index) in PinnedApps"
          :key="index"
          class="text-white h-full aspect-square hover:scale-105 active:scale-95 rounded-2xl"
        >
          <component :is="app" :size="30" />
        </LiquidGlass>
      </div>

      <!-- Right -->
      <div class="flex gap-1 items-center w-fit h-full">
        <LiquidGlass
          class="h-full hover:scale-105 active:scale-95 bg-white/1 text-white rounded-l-4xl px-2 rounded-r-lg"
        >
          <div
            class="aspect-square rounded-full h-10 w-10 flex items-center justify-center bg-black/50 text-white font-extrabold"
          >
            1
          </div>
        </LiquidGlass>
        <LiquidGlass
          class="h-full w-fit px-4 gap-0 text-lg hover:scale-105 active:scale-95 bg-white/1 text-white rounded-lg flex flex-col items-center justify-center"
        >
          <strong class="leading-none text-shadow-lg">9:18</strong
          ><span class="leading-none text-xs text-shadow-lg">AM</span>
        </LiquidGlass>
        <LiquidGlass
          class="h-full w-fit px-3 gap-2 hover:scale-105 active:scale-95 bg-white/1 text-white rounded-l-lg rounded-r-4xl flex items-center justify-center"
        >
          <Wifi :size="25" />
          <BatteryMedium :size="27" />
          <CircleUserRound :size="25" />
        </LiquidGlass>
      </div>
    </div>
  </div>

  <RouterView />
</template>
