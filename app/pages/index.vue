<template>
  <div class="h-screen flex flex-col bg-gray-50 dark:bg-gray-950">
    <UContainer class="flex-1 w-full max-w-none p-4 flex gap-4 overflow-hidden">
      
      <!-- Sidebar / TOC -->
      <div class="w-1/3 min-w-[300px] flex flex-col bg-white dark:bg-gray-900 rounded-lg shadow-sm border border-gray-200 dark:border-gray-800 overflow-hidden">
        <div class="p-4 border-b border-gray-200 dark:border-gray-800">
           <h2 class="text-lg font-semibold text-gray-900 dark:text-white mb-2">History</h2>
           <UInput
              v-model="search"
              icon="i-heroicons-magnifying-glass-20-solid"
              size="sm"
              color="neutral"
              variant="outline"
              placeholder="Filter..."
              class="w-full"
           />
        </div>
        
        <div class="flex-1 overflow-y-auto p-2 scrollbar-thin scrollbar-thumb-gray-200 dark:scrollbar-thumb-gray-700">
            <div v-if="status === 'pending'" class="flex justify-center py-4">
                <UIcon name="i-heroicons-arrow-path" class="w-6 h-6 animate-spin text-primary-500" />
            </div>
             <div v-else-if="status === 'error'" class="text-center py-4 text-red-500">
                <p>Failed to load.</p>
            </div>
            
            <nav v-else class="space-y-1">
                <!-- Data Empty State -->
                <div v-if="!tocStructure.length" class="text-center text-gray-400 text-sm py-8">
                    No results found
                </div>

                <!-- Custom TOC -->
                <div v-for="year in tocStructure" :key="year.label" class="space-y-1">
                    <!-- Year Header -->
                     <button 
                        @click="scrollToLog(year.firstLogId)"
                        class="w-full flex items-center gap-2 px-2 py-1.5 text-sm font-semibold rounded-md transition-colors hover:bg-gray-100 dark:hover:bg-gray-800"
                        :class="year.expanded ? 'text-gray-900 dark:text-gray-100' : 'text-gray-500 dark:text-gray-400'"
                     >
                        <UIcon 
                            :name="year.expanded ? 'i-heroicons-chevron-down' : 'i-heroicons-chevron-right'" 
                            class="w-4 h-4 transition-transform duration-200" 
                        />
                        <span>{{ year.label }}</span>
                     </button>

                     <!-- Months -->
                     <div v-show="year.expanded" class="pl-4 space-y-1 border-l border-gray-100 dark:border-gray-800 ml-3">
                        <div v-for="month in year.children" :key="month.label" class="space-y-1">
                            <button 
                                @click="scrollToLog(month.firstLogId)"
                                class="w-full flex items-center gap-2 px-2 py-1 text-sm font-medium rounded-md transition-colors hover:bg-gray-100 dark:hover:bg-gray-800"
                                :class="month.expanded ? 'text-gray-900 dark:text-gray-100' : 'text-gray-500 dark:text-gray-500'"
                            >
                                <span class="capitalize">{{ month.label }}</span>
                            </button>

                            <!-- Days -->
                            <div v-show="month.expanded" class="pl-4 space-y-1 border-l border-gray-100 dark:border-gray-800 ml-1">
                                <div v-for="day in month.children" :key="day.label" class="space-y-1">
                                     <!-- Day Header (Optional, often just wrapper) -->
                                    <div class="px-2 py-0.5 text-xs font-bold text-gray-400 uppercase tracking-wider mt-2 mb-1">
                                        {{ day.label }}
                                    </div>
                                    
                                    <!-- Log Items -->
                                    <div class="space-y-0.5">
                                        <button 
                                            v-for="log in day.children" 
                                            :key="log.id"
                                            @click="scrollToLog(log.id)"
                                            class="w-full text-left px-2 py-1.5 text-sm rounded-md truncate transition-all duration-200 border-l-2"
                                            :class="activeLogId === log.id 
                                                ? 'bg-primary-50 dark:bg-primary-900/10 text-primary-600 dark:text-primary-400 border-primary-500 font-medium' 
                                                : 'text-gray-600 dark:text-gray-400 border-transparent hover:bg-gray-50 dark:hover:bg-gray-800 hover:text-gray-900 dark:hover:text-gray-200'"
                                        >
                                            {{ log.label }}
                                        </button>
                                    </div>
                                </div>
                            </div>
                        </div>
                     </div>
                </div>
            </nav>
        </div>
      </div>

      <!-- Main Content -->
      <div class="flex-1 flex flex-col bg-white dark:bg-gray-900 rounded-lg shadow-sm border border-gray-200 dark:border-gray-800 overflow-hidden relative">
        <div class="flex-1 overflow-y-auto p-0 scroll-smooth" ref="mainScroll">
             <div v-if="processedLogs.length === 0" class="h-full flex flex-col items-center justify-center text-gray-400">
                 <UIcon name="i-heroicons-inbox" class="w-16 h-16 mb-4 opacity-50" />
                 <p>No messages to display</p>
             </div>
             <div v-else class="divide-y divide-gray-100 dark:divide-gray-800">
                 <div v-for="log in processedLogs" :key="log.id" :id="log.id" class="p-8 pb-16 min-h-[50vh] flex flex-col gap-8" data-log-item>
                    <!-- Header Metadata -->
                    <div class="flex flex-wrap items-center justify-between gap-4 pb-4">
                       <div class="flex items-center gap-3">
                          <UBadge color="primary" variant="subtle" size="sm">{{ log.header }}</UBadge>
                          <span class="text-sm text-gray-500 font-mono">{{ log.formattedTime }}</span>
                       </div>
                       <div class="flex gap-2">
                           <UBadge v-for="prod in log.products" :key="prod" color="neutral" variant="outline" size="xs">{{ prod }}</UBadge>
                       </div>
                    </div>

                    <!-- User Prompt -->
                    <div class="flex justify-end pl-12 group">
                        <div class="flex flex-col items-end gap-1 max-w-[85%]">
                             <div class="flex items-center gap-2 opacity-60 mr-1">
                                 <span class="text-xs font-medium">You</span>
                                 <UIcon name="i-heroicons-user" class="w-3 h-3" />
                             </div>
                             <div class="bg-gray-100 dark:bg-gray-800 text-gray-900 dark:text-white rounded-2xl rounded-tr-md px-5 py-3 shadow-sm text-base leading-relaxed whitespace-pre-wrap">
                                {{ log.cleanTitle }}
                             </div>
                        </div>
                    </div>

                    <!-- Assistant Response -->
                    <div class="flex justify-start pr-12">
                        <div class="flex flex-col items-start gap-1 w-full max-w-[90%]">
                             <div class="flex items-center gap-2 opacity-60 ml-1">
                                 <UIcon name="i-heroicons-sparkles" class="w-3 h-3 text-primary-500" />
                                 <span class="text-xs font-medium">Gemini</span>
                             </div>
                             
                             <div v-if="log.responseHtml" class="w-full bg-white dark:bg-gray-900 border border-gray-200 dark:border-gray-800 rounded-2xl rounded-tl-md p-6 shadow-sm overflow-hidden">
                                 <div class="prose dark:prose-invert max-w-none prose-neutral prose-sm sm:prose-base">
                                     <div v-html="log.responseHtml" class="w-full overflow-hidden break-words" />
                                 </div>
                             </div>
                             <div v-else class="text-gray-500 italic px-4">
                                No content to display for this item.
                             </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
      </div>

    </UContainer>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, nextTick, onMounted } from 'vue'

interface HtmlItem {
  html: string
}

interface LogItem {
  id: string
  header: string
  title: string
  time: string
  products: string[]
  activityControls: string[]
  safeHtmlItem?: HtmlItem[]
  cleanTitle?: string
  formattedTime?: string
  responseHtml?: string
}

// Fetch the data
const { data: rawLogs, status, error } = await useFetch<LogItem[]>('/MyActivity.json', {
  server: false,
  lazy: true
})

// Processed logs
const processedLogs = computed(() => {
  if (!rawLogs.value) return []
  return rawLogs.value
    .map((item, index) => ({
      ...item,
      id: `log-${index}`,
      cleanTitle: item.title ? item.title.replace(/^Prompted\s*/, '') : 'No Title',
      formattedTime: item.time ? new Date(item.time).toLocaleDateString(undefined, {
        year: 'numeric',
        month: 'short',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      }) : 'Unknown Date',
      responseHtml: item.safeHtmlItem?.[0]?.html || ''
    }))
    .sort((a, b) => new Date(b.time || 0).getTime() - new Date(a.time || 0).getTime())
})

// Scroll & Interaction
const search = ref('')
const activeLogId = ref<string | null>(null)
const mainScroll = ref<HTMLElement | null>(null)
let observer: IntersectionObserver | null = null

const scrollToLog = (id?: string) => {
  if (!id) return
  const el = document.getElementById(id)
  if (el) {
    el.scrollIntoView({ behavior: 'smooth', block: 'start' })
  }
}

// Intersection Observer
onMounted(() => {
  watch(() => processedLogs.value.length, () => {
       if (processedLogs.value.length > 0) {
           nextTick(initObserver)
       }
  }, { immediate: true })
})

const initObserver = () => {
    if (observer) observer.disconnect()
    
    observer = new IntersectionObserver((entries) => {
        // Find visible entries
        const visible = entries.filter(e => e.isIntersecting)
        if (visible.length > 0) {
             // Prefer the one closest to the top (first in the list usually)
             // IntersectionObserver entries order is not guaranteed to be visual order always, 
             // but usually strictly ordered by DOM.
             activeLogId.value = visible[0].target.id
        }
    }, {
        root: mainScroll.value,
        threshold: 0.1, // Trigger when 10% visible
        rootMargin: '-10% 0px -70% 0px' // Bias selection towards the top of the viewport
    })
    
    const items = document.querySelectorAll('[data-log-item]')
    items.forEach(el => observer?.observe(el))
}

// Derived State for Expansion
// Identify the "Active" hierarchy based on activeLogId
const activeHierarchy = computed(() => {
    if (!activeLogId.value) return { year: null, month: null, day: null }
    const log = processedLogs.value.find(l => l.id === activeLogId.value)
    if (!log || !log.time) return { year: null, month: null, day: null }
    
    const date = new Date(log.time)
    return {
        year: date.getFullYear().toString(),
        month: date.toLocaleString('default', { month: 'long' }),
        day: date.getDate().toString().padStart(2, '0')
    }
})

// TOC Construction
const tocStructure = computed(() => {
  if (!processedLogs.value.length) return []

  const query = search.value.toLowerCase()
  const groups: Record<string, Record<string, Record<string, any[]>>> = {}
  
  // Group logs
  processedLogs.value.forEach(log => {
      // Filter if search is active
      if (query && !log.cleanTitle?.toLowerCase().includes(query)) {
          return
      }

      if (!log.time) return
      const date = new Date(log.time)
      const year = date.getFullYear().toString()
      const month = date.toLocaleString('default', { month: 'long' })
      const day = date.getDate().toString().padStart(2, '0')
      
      if (!groups[year]) groups[year] = {}
      if (!groups[year][month]) groups[year][month] = {}
      if (!groups[year][month][day]) groups[year][month][day] = []
      
      groups[year][month][day].push({
          id: log.id,
          label: log.cleanTitle || 'No Title'
      })
  })

  // Build Hierarchy
  // Sort Years Desc
  return Object.keys(groups).sort((a, b) => b.localeCompare(a)).map(year => {
      const yearIsActive = activeHierarchy.value.year === year
      const yearHasMatch = query.length > 0 // If filtered, we have matches by definition of the loop above
      
      const childrenMonths = Object.keys(groups[year]).map(month => {
          const monthIsActive = activeHierarchy.value.month === month
          
          const childrenDays = Object.keys(groups[year][month])
            .sort((a,b) => parseInt(b) - parseInt(a))
            .map(day => ({
                label: `Day ${day}`,
                children: groups[year][month][day], // Logs
                // In day view, we always just show the list if the month is open
            }))
            
          // If first log of month is needed
          const firstLogId = childrenDays[0]?.children?.[0]?.id

          return {
              label: month,
              children: childrenDays,
              expanded: query.length > 0 || (yearIsActive && monthIsActive),
              firstLogId
          }
      })
      
      // If first log of year is needed
      const firstLogId = childrenMonths[0]?.firstLogId

      return {
          label: year,
          children: childrenMonths,
          expanded: query.length > 0 || yearIsActive,
          firstLogId
      }
  })
})
</script>
