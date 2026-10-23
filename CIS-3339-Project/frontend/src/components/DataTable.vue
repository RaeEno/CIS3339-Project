<script setup>
import { ref, computed, watch } from 'vue'

// Task F3 Props: columns array and rows array
const props = defineProps({
  columns: {
    type: Array,
    required: true
  },
  rows: {
    type: Array,
    required: true,
    default: () => []
  }
})

// Emits row selection to parent views (e.g., open client/event details)
const emit = defineEmits(['row-click'])

// 1. Sorting: cycles Ascending -> Descending -> Unsorted
const sortKey = ref(null)
const sortDirection = ref(null)

const toggleSort = (key) => {
  if (sortKey.value !== key) {
    sortKey.value = key
    sortDirection.value = 'asc'
  } else if (sortDirection.value === 'asc') {
    sortDirection.value = 'desc'
  } else if (sortDirection.value === 'desc') {
    sortKey.value = null
    sortDirection.value = null
  }
}

const sortedRows = computed(() => {
  if (!sortKey.value || !sortDirection.value) {
    return props.rows
  }

  return [...props.rows].sort((a, b) => {
    const valA = a[sortKey.value] ?? ''
    const valB = b[sortKey.value] ?? ''

    if (valA < valB) return sortDirection.value === 'asc' ? -1 : 1
    if (valA > valB) return sortDirection.value === 'asc' ? 1 : -1
    return 0
  })
})

// 2. Pagination: 10 rows per page
const currentPage = ref(1)
const pageSize = ref(10)

// Reset to page 1 if the underlying data updates (e.g., search/filter)
watch(
  () => props.rows,
  () => {
    currentPage.value = 1
  }
)

const totalRows = computed(() => sortedRows.value.length)
const totalPages = computed(() => Math.ceil(totalRows.value / pageSize.value) || 1)

const paginatedRows = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  return sortedRows.value.slice(start, start + pageSize.value)
})

const startRecord = computed(() => (totalRows.value === 0 ? 0 : (currentPage.value - 1) * pageSize.value + 1))
const endRecord = computed(() => Math.min(currentPage.value * pageSize.value, totalRows.value))

const nextPage = () => {
  if (currentPage.value < totalPages.value) currentPage.value++
}

const prevPage = () => {
  if (currentPage.value > 1) currentPage.value--
}
</script>

<template>
  <div class="w-full">
    <div class="overflow-x-auto shadow-md rounded-lg border border-gray-200">
      <table class="min-w-full text-left text-sm text-gray-700 bg-white">
        <!-- Table Headers -->
        <thead class="bg-gray-100 text-xs uppercase font-semibold text-gray-700">
          <tr>
            <th
              v-for="col in columns"
              :key="col.key"
              @click="toggleSort(col.key)"
              class="px-6 py-3 cursor-pointer select-none hover:bg-gray-200 transition-colors"
            >
              <div class="flex items-center space-x-1">
                <span>{{ col.label }}</span>
                <span class="text-xs">
                  <template v-if="sortKey === col.key && sortDirection === 'asc'">▲</template>
                  <template v-else-if="sortKey === col.key && sortDirection === 'desc'">▼</template>
                  <template v-else>↕</template>
                </span>
              </div>
            </th>
          </tr>
        </thead>

        <!-- Table Rows -->
        <tbody class="divide-y divide-gray-200">
          <!-- Empty State (v-if) -->
          <tr v-if="paginatedRows.length === 0">
            <td :colspan="columns.length" class="px-6 py-8 text-center text-gray-500 italic">
              No results found.
            </td>
          </tr>

          <!-- Data Rows (v-for) -->
          <tr
            v-else
            v-for="row in paginatedRows"
            :key="row._id || row.id"
            @click="emit('row-click', row)"
            class="hover:bg-blue-50 cursor-pointer transition-colors"
          >
            <td v-for="col in columns" :key="col.key" class="px-6 py-4">
              {{ row[col.key] }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Pagination Footer -->
    <div class="flex items-center justify-between px-4 py-3 bg-white border-t border-gray-200 mt-2 sm:px-6">
      <div class="text-sm text-gray-700">
        Showing <span class="font-medium">{{ startRecord }}</span> to
        <span class="font-medium">{{ endRecord }}</span> of
        <span class="font-medium">{{ totalRows }}</span> results
      </div>
      <div class="flex space-x-2">
        <button
          @click="prevPage"
          :disabled="currentPage === 1"
          class="px-3 py-1 text-sm border rounded-md disabled:opacity-40 disabled:cursor-not-allowed hover:bg-gray-100"
        >
          Previous
        </button>
        <span class="px-3 py-1 text-sm text-gray-600">
          Page {{ currentPage }} of {{ totalPages }}
        </span>
        <button
          @click="nextPage"
          :disabled="currentPage === totalPages || totalRows === 0"
          class="px-3 py-1 text-sm border rounded-md disabled:opacity-40 disabled:cursor-not-allowed hover:bg-gray-100"
        >
          Next
        </button>
      </div>
    </div>
  </div>
</template>