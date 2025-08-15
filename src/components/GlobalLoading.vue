<!-- src/components/GlobalLoading.vue -->
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useLoadingStore } from '@/stores/loadingStore'

const store = useLoadingStore()
const { active, message } = storeToRefs(store)
</script>

<template>
  <transition name="glfade" appear>
    <!-- ลอยมุมขวาบน แบบไม่บังจอ -->
    <div
      v-if="active"
      class="fixed top-4 right-4 z-[9999] flex items-center gap-3
             rounded-lg bg-black/70 text-white shadow-lg backdrop-blur px-3 py-2"
      role="status" aria-live="polite"
    >
      <!-- วงหมุนเล็ก -->
      <span
        class="inline-block h-4 w-4 animate-spin rounded-full
               border-2 border-white/70 border-t-transparent"
        aria-hidden="true"
      />
      <span class="text-sm font-medium">{{ message }}</span>
    </div>
  </transition>
</template>

<style scoped>
.glfade-enter-active, .glfade-leave-active {
  transition: opacity .18s ease, transform .18s ease;
}
.glfade-enter-from, .glfade-leave-to {
  opacity: 0; transform: translateY(-4px);
}
</style>
