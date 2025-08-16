<script setup lang="ts">
import NewsCard from '@/components/NewsCard.vue'
import type { NewsItem } from '@/types'
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import { useRouter } from 'vue-router'
import { useNewsStore } from '@/stores/newsStore'

type Filter = 'all' | 'fake' | 'not-fake'

const router = useRouter()
const store = useNewsStore()

const props = defineProps<{
  page: number
  pageSize: number
  filter?: Filter
  q?: string
}>()

const page = computed(() => Number(props.page) || 1)
const pageSize = computed(() => Number(props.pageSize) || 10)
const filter = computed(() => (props.filter ?? 'all') as Filter)
const searchTerm = ref(props.q ?? '')

// ------------------ loaders ------------------
let debounceId: ReturnType<typeof setTimeout> | null = null
let lastKey = ''

function keyFor(pp: number, p: number, f: Filter, q: string) {
  return `p=${p}&s=${pp}&f=${f}&q=${q}`
}

function load(pp = pageSize.value, p = page.value, f = filter.value, q = searchTerm.value.trim()) {
  const key = keyFor(pp, p, f, q)
  if (key === lastKey) return
  lastKey = key
  store.fetchList(pp, p, f, q)
}

// A) เปลี่ยนหน้า/จำนวนต่อหน้า/ฟิลเตอร์ => โหลดทันที (ไม่ debounce)
watch(
  () => [page.value, pageSize.value, filter.value],
  () => load(),
  { immediate: true }
)

// B) ค้นหา => debounce 300ms
watch(
  () => searchTerm.value,
  () => {
    if (debounceId) clearTimeout(debounceId)
    debounceId = setTimeout(() => load(), 300)
  }
)

onBeforeUnmount(() => {
  if (debounceId) clearTimeout(debounceId)
  if (typeof window !== 'undefined') window.removeEventListener('scroll', onScroll)
})

// SWR: แสดงของเก่าไว้ถ้า store ยังไม่เคยมีข้อมูล
const newsItems = computed<NewsItem[]>(() => store.list ?? [])
const hasData = computed(() => (newsItems.value?.length ?? 0) > 0)

const totalNews = computed(() => store.total)
const hasNextPage = computed(() => Math.ceil(totalNews.value / pageSize.value) > page.value)

const pageSizeOptions = [5, 10, 15, 20]
const filterOptions: { value: Filter; label: string }[] = [
  { value: 'all', label: 'All' },
  { value: 'fake', label: 'Fake' },
  { value: 'not-fake', label: 'Not Fake' },
]

function handleSearch() {
  router.push({
    name: 'news-list-view',
    query: {
      page: 1,
      pageSize: pageSize.value,
      filter: filter.value,
      q: searchTerm.value.trim(),
    },
  })
}

/* segmented helpers */
function setFilter(v: Filter) {
  router.push({
    name: 'news-list-view',
    query: {
      page: 1,
      pageSize: pageSize.value,
      filter: v,
      q: searchTerm.value.trim(),
    },
  })
}
function setPageSize(v: number) {
  router.push({
    name: 'news-list-view',
    query: {
      page: 1,
      pageSize: v,
      filter: filter.value,
      q: searchTerm.value.trim(),
    },
  })
}
function filterBtnClass(v: Filter) {
  const base = 'px-4 py-1.5 rounded-full text-sm font-medium transition outline-none focus-visible:ring-2 focus-visible:ring-offset-2 focus-visible:ring-[#AF0000]'
  const inactive = 'text-slate-600 hover:bg-white/90'
  if (filter.value !== v) return `${base} ${inactive}`
  if (v === 'fake') return `${base} bg-red-500 text-white shadow`
  if (v === 'not-fake') return `${base} bg-emerald-500 text-white shadow`
  return `${base} bg-slate-800 text-white shadow` // all
}
function pageBtnClass(n: number) {
  const base = 'px-3 sm:px-4 py-1.5 rounded-full text-sm font-semibold transition outline-none focus-visible:ring-2 focus-visible:ring-offset-2 focus-visible:ring-[#AF0000]'
  return pageSize.value === n
    ? `${base} bg-[#AF0000] text-white shadow`
    : `${base} text-slate-700 hover:bg-white/90`
}

// ---- Scroll-to-top button visibility ----
const showScrollTop = ref(false)
function onScroll() {
  if (typeof window === 'undefined') return
  showScrollTop.value = window.scrollY > 300
}

onMounted(() => {
  if (typeof window !== 'undefined') {
    window.addEventListener('scroll', onScroll, { passive: true })
    onScroll()
  }
})

function scrollToTop() {
  if (typeof window !== 'undefined') {
    window.scrollTo({ top: 0, behavior: 'smooth' })
  }
}
</script>

<template>
  <div class="mx-auto max-w-7xl px-3 sm:px-4 lg:px-6">
    <!-- Header -->
    <header class="pt-6 sm:pt-10">
      <h1 class="text-2xl sm:text-4xl lg:text-5xl font-extrabold text-center tracking-tight text-[#2c3e50]">
        The Social Anti-Fake News System
      </h1>
      <p class="mx-auto mt-2 sm:mt-3 max-w-2xl text-center text-sm sm:text-base text-gray-600">
        Community votes and comments decide the news status.
      </p>
    </header>

    <!-- Search + filters (non-sticky as requested) -->
    <div class="mt-4 sm:mt-6 border border-gray-200 rounded-2xl p-3 sm:p-4 shadow-sm bg-white">
      <!-- Search -->
      <div class="flex flex-col sm:flex-row gap-2 sm:gap-3">
        <label for="search" class="sr-only">Search</label>
        <input
          id="search"
          v-model="searchTerm"
          type="text"
          placeholder="Search news by title or detail…"
          class="w-full px-4 py-2.5 rounded-xl border border-gray-300 bg-white focus:outline-none focus:ring-2 focus:ring-[#AF0000]"
          @keyup.enter="handleSearch"
        />
        <button
          @click="handleSearch"
          class="inline-flex items-center justify-center px-5 py-2.5 rounded-xl bg-[#AF0000] text-white font-medium hover:bg-[#af0000d8] transition-colors"
        >
          Search
        </button>
      </div>

      <!-- Filters -->
      <div class="mt-3 flex flex-wrap items-center gap-3 justify-between">
        <div class="inline-flex flex-wrap items-center gap-1 rounded-full bg-slate-100 p-1 shadow-inner">
          <button
            v-for="opt in filterOptions"
            :key="opt.value"
            type="button"
            :class="filterBtnClass(opt.value)"
            @click="setFilter(opt.value)"
          >
            {{ opt.label }}
          </button>
        </div>

        <div class="flex items-center gap-2 sm:gap-3 md:justify-end">
          <span class="text-slate-600 text-sm">Per page</span>
          <div class="inline-flex items-center gap-1 rounded-full bg-slate-100 p-1 shadow-inner">
            <button
              v-for="n in pageSizeOptions"
              :key="n"
              type="button"
              :class="pageBtnClass(n)"
              @click="setPageSize(n)"
            >
              {{ n }}
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- List / states -->
    <section class="mt-6 sm:mt-8">
      <div v-if="hasData" class="grid grid-cols-1 sm:grid-cols-2 xl:grid-cols-3 2xl:grid-cols-4 gap-4 sm:gap-6">
        <div
          v-for="n in newsItems"
          :key="n.id"
          class="group rounded-2xl ring-1 ring-transparent transform-gpu transition duration-200 ease-out hover:-translate-y-1 hover:ring-gray-200 hover:shadow-md focus-within:ring-gray-200 motion-reduce:transition-none motion-reduce:transform-none"
        >
          <NewsCard :news="n" />
        </div>
      </div>
      <div v-else-if="store.loading" class="text-center text-gray-500 text-lg mt-12">
        <p>Loading news...</p>
      </div>
      <div v-else class="text-center text-gray-500 text-lg mt-12">
        <p>No news found with the current filters.</p>
      </div>

      <!-- แถบแจ้งกำลังอัปเดต (ไม่บังรายการ) -->
      <p v-if="hasData && store.loading" class="mt-3 text-center text-xs text-slate-500" role="status">
        Updating…
      </p>
    </section>

    <!-- ปุ่มเลื่อนไปบนสุด: มินิมอลวงกลมโปร่ง + ไอคอนลูกศรสวยๆ -->
    <button
      v-show="showScrollTop"
      @click="scrollToTop"
      class="fixed bottom-6 right-6 grid h-12 w-12 place-items-center rounded-full border border-[#AF0000]/40 bg-white/80 text-[#AF0000] shadow-sm backdrop-blur transition duration-200 hover:bg-white hover:shadow-md hover:-translate-y-0.5 focus:outline-none focus-visible:ring-2 focus-visible:ring-offset-2 focus-visible:ring-[#AF0000]/50"
      aria-label="Scroll to top"
    >
      <svg viewBox="0 0 24 24" fill="none" class="h-5 w-5" aria-hidden="true">
        <path d="M6 14l6-6 6 6" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
      </svg>
      <span class="sr-only">Back to top</span>
    </button>

    <!-- Pagination -->
    <nav class="mx-auto my-10 sm:my-12 max-w-xl" aria-label="Pagination">
      <div class="flex flex-col items-center gap-4">
        <div class="flex justify-center items-center gap-3 sm:gap-6">
          <router-link
            id="page-prev"
            v-if="page > 1"
            :to="{ name: 'news-list-view', query: { page: page - 1, pageSize: pageSize, filter: filter, q: searchTerm.trim() } }"
            rel="prev"
            class="inline-flex items-center gap-2 py-[10px] px-4 bg-white border border-gray-300 text-gray-700 rounded-[999px] font-medium text-sm hover:bg-gray-50 shadow-sm"
          >
            <span>Previous</span>
          </router-link>

          <div class="text-gray-600 text-sm font-medium px-4">
            <span>Page {{ page }} of {{ Math.ceil(totalNews / pageSize) }}</span>
          </div>

          <router-link
            id="page-next"
            v-if="hasNextPage"
            :to="{ name: 'news-list-view', query: { page: page + 1, pageSize: pageSize, filter: filter, q: searchTerm.trim() } }"
            rel="next"
            class="inline-flex items-center gap-2 py-[10px] px-4 bg-[#AF0000] text-white rounded-[999px] font-medium text-sm hover:bg-[#af0000d8] shadow-sm"
          >
            <span>Next</span>
          </router-link>
        </div>
      </div>
    </nav>
  </div>
</template>
