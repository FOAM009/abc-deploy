<script setup lang="ts">
import { computed } from 'vue'
import { useRouter } from 'vue-router'
import { useNewsStore } from '@/stores/newsStore'
import type { NewsItem } from '@/types'

const props = defineProps<{ news: NewsItem }>()
const router = useRouter()
const store = useNewsStore()

function toDetail() {
  router.push({ name: 'news-detail-view', params: { id: props.news.id } })
}

/**
 * IMPORTANT:
 * ดึงอ็อบเจ็กต์เดียวกับหน้า Detail เสมอ (อ้างอิงเดียวกัน)
 */
const newsRef = computed<NewsItem>(() => store.getById(props.news.id) ?? props.news)

// Votes & status ตาม logic ของ Detail
const votes = computed(() => store.getVotesFromComments(newsRef.value))
const status = computed<'fake' | 'not-fake'>(() => store.getStatusFromComments(newsRef.value))

// badge tone (มินิมอล)
const statusClass = computed(() =>
  status.value === 'fake'
    ? 'bg-red-50 text-red-700 border border-red-200'
    : 'bg-green-50 text-green-700 border border-green-200'
)

// ข้อความสถานะ (ใช้ใน template เพื่อหลีกเลี่ยง .value)
const statusLabel = computed(() => (status.value === 'fake' ? 'Fake' : 'Not Fake'))

// แถบสีด้านล่างตามสถานะ (คงแบบเดิมของคุณไว้)
const accentBgClass = computed(() =>
  status.value === 'fake' ? 'bg-red-500/80' : 'bg-emerald-500/80'
)

// วันที่อ่านง่าย
const displayDate = computed(() => {
  const d = new Date(newsRef.value.date)
  try {
    return new Intl.DateTimeFormat(undefined, { dateStyle: 'medium', timeStyle: 'short' }).format(d)
  } catch {
    return d.toLocaleString()
  }
})
</script>

<template>
  <article
    role="button"
    tabindex="0"
    :aria-label="`Open: ${newsRef.title}`"
    @click="toDetail"
    @keydown.enter="toDetail"
    @keydown.space.prevent="toDetail"
    class="group relative w-full h-full bg-white rounded-xl shadow-sm ring-1 ring-gray-100 p-3 sm:p-4 cursor-pointer transition duration-200 transform-gpu hover:-translate-y-[2px] hover:shadow-md focus:outline-none md:min-h-[220px] lg:min-h-[240px] flex flex-col"
  >
    <!-- Image: แสดงเฉพาะจอ md ขึ้นไป, มือถือซ่อนไว้ให้การ์ดเตี้ยลง -->
    <div class="hidden md:block mb-3 overflow-hidden rounded-lg aspect-[2/1] lg:aspect-[16/9] bg-gray-100">
      <img
        v-if="newsRef.imageUrl"
        :src="newsRef.imageUrl"
        alt=""
        loading="lazy"
        class="h-full w-full object-cover transition duration-300 group-hover:scale-[1.02]"
        @click.stop="toDetail"
      />
      <div v-else class="h-full w-full"></div>
    </div>

    <!-- Title + Badge (พาดหัวกระชับลง) -->
    <div class="flex justify-between items-start gap-2 mb-1.5 sm:mb-2">
      <h2 class="text-[15px] sm:text-base md:text-lg font-semibold leading-snug tracking-tight text-gray-900 line-clamp-2">
        {{ newsRef.title }}
      </h2>
      <span :class="statusClass" class="text-[10px] sm:text-[11px] md:text-xs font-medium px-2 py-0.5 md:py-1 rounded-full whitespace-nowrap select-none">
        {{ statusLabel }}
      </span>
    </div>

    <!-- Summary (มือถือให้สั้นลงมาก) -->
    <p class="text-[13px] sm:text-sm text-gray-700/90 mb-1.5 sm:mb-2 line-clamp-1 sm:line-clamp-2">
      {{ newsRef.summary }}
    </p>

    <!-- Spacer: ให้สูงเท่ากันบน md+ เท่านั้น -->
    <div class="md:grow"></div>

    <!-- Meta: รวมเป็นแถวเดียวเพื่อลดช่องว่าง -->
    <div class="mt-1.5 flex flex-wrap items-center gap-x-3 gap-y-1 text-[10px] sm:text-[11px] md:text-xs text-gray-500">
      <span class="truncate">Reporter: {{ newsRef.reporter }}</span>
      <span class="truncate">votes: F {{ votes.fake }} / NF {{ votes['not-fake'] }}</span>
      <span class="ml-auto whitespace-nowrap">{{ displayDate }}</span>
    </div>

    <!-- Accent bar: คงแบบเดิมตาม Fake/Not Fake -->
    <div aria-hidden="true" class="pointer-events-none absolute bottom-0 left-0 right-0 h-[3px] rounded-b-xl" :class="accentBgClass" />
  </article>
</template>
