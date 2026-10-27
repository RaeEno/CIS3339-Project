<script setup>
// =====================================================================
// WHY THIS COMPONENT EXISTS
// A reusable table: the parent gives it column definitions + row data,
// and it handles sorting, pagination, and click events by itself so
// every page doesn't have to rebuild that logic.
// =====================================================================

// ---- IMPORTS ----
// ref      = a reactive variable (when it changes, the screen updates)
// computed = a value that auto-recalculates when what it depends on changes
// watch    = runs code when a specific value changes
import { ref, computed, watch } from 'vue'

// =====================================================================
// PROPS (data the parent passes IN)
// =====================================================================
// Task F3 Props: columns array and rows array
const props = defineProps({
  // columns: describes each column, e.g. [{ key: 'name', label: 'Name' }]
  //   key   = which field to read from each row
  //   label = the text shown in the header
  columns: {
    type: Array,
    required: true
  },
  // rows: the actual data, e.g. [{ id: 1, name: 'Sam' }, ...]
  rows: {
    type: Array,
    required: true,
    default: () => [] // empty list if the parent sends nothing
  }
})

// =====================================================================
// EMITS (events sent OUT to the parent)
// =====================================================================
// Tells the parent which row was clicked (e.g., to open a details page)
const emit = defineEmits(['row-click'])

// =====================================================================
// 1. SORTING
// Why: lets users click a header to order the data.
// Cycle on repeated clicks: Ascending -> Descending -> Unsorted
// =====================================================================

// Which column is currently sorted (null = none)
const sortKey = ref(null)
// 'asc', 'desc', or null
const sortDirection = ref(null)

// Runs when a header is clicked
const toggleSort = (key) => {
  if (sortKey.value !== key) {
    // New column clicked -> start at ascending
    sortKey.value = key
    sortDirection.value = 'asc'
  } else if (sortDirection.value === 'asc') {
    // Same column, was ascending -> go descending
    sortDirection.value = 'desc'
  } else if (sortDirection.value === 'desc') {
    // Same column, was descending -> clear the sort
    sortKey.value = null
    sortDirection.value = null
  }
}

// The rows AFTER sorting is applied
const sortedRows = computed(() => {
  // No sort active -> return rows in original order
  if (!sortKey.value || !sortDirection.value) {
    return props.rows
  }

  // [...props.rows] makes a COPY so we never change the parent's data
  return [...props.rows].sort((a, b) => {
    // ?? '' = if the value is missing (null/undefined), treat as empty text
    const valA = a[sortKey.value] ?? ''
    const valB = b[sortKey.value] ?? ''

    // Return -1 (a first), 1 (b first), or 0 (equal)
    // Flips depending on asc vs desc
    if (valA < valB) return sortDirection.value === 'asc' ? -1 : 1
    if (valA > valB) return sortDirection.value === 'asc' ? 1 : -1
    return 0
  })
})

// =====================================================================
// 2. PAGINATION
// Why: showing hundreds of rows at once is overwhelming and slow,
// so we show 10 per page.
// =====================================================================

const currentPage = ref(1)
const pageSize = ref(10) // rows per page

// If the parent sends new rows (e.g., after a search/filter),
// jump back to page 1 so the user isn't stuck on a page that no longer exists
watch(
  () => props.rows,
  () => {
    currentPage.value = 1
  }
)

// Total number of rows
const totalRows = computed(() => sortedRows.value.length)

// Total pages = rows / page size, rounded UP. "|| 1" keeps it at 1 when empty
const totalPages = computed(() => Math.ceil(totalRows.value / pageSize.value) || 1)

// The slice of rows for the current page only
// Page 1 -> rows 0-9, Page 2 -> rows 10-19, etc.
const paginatedRows = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  return sortedRows.value.slice(start, start + pageSize.value)
})

// Numbers for the footer text: "Showing 11 to 20 of 45 results"
const startRecord = computed(() => (totalRows.value === 0 ? 0 : (currentPage.value - 1) * pageSize.value + 1))
const endRecord = computed(() => Math.min(currentPage.value * pageSize.value, totalRows.value))

// Move forward one page (only if not on the last page)
const nextPage = () => {
  if (currentPage.value < totalPages.value) currentPage.value++
}

// Move back one page (only if not on the first page)
const prevPage = () => {
  if (currentPage.value > 1) currentPage.value--
}
</script>

<template>
  <!-- Outer wrapper: full width -->
  <div class="w-full">
    <!-- Scroll wrapper: lets the table scroll sideways on small screens.
         Also adds the shadow, rounded corners, and border -->
    <div class="overflow-x-auto shadow-md rounded-lg border border-gray-200">
      <table class="min-w-full text-left text-sm text-gray-700 bg-white">

        <!-- ============ TABLE HEADERS ============ -->
        <thead class="bg-gray-100 text-xs uppercase font-semibold text-gray-700">
          <tr>
            <!-- v-for: loops over columns, creating one <th> per column
                 :key gives Vue a unique ID per item so it tracks them correctly
                 @click: clicking a header triggers sorting for that column -->
            <th
              v-for="col in columns"
              :key="col.key"
              @click="toggleSort(col.key)"
              class="px-6 py-3 cursor-pointer select-none hover:bg-gray-200 transition-colors"
            >
              <div class="flex items-center space-x-1">
                <!-- Column title -->
                <span>{{ col.label }}</span>

                <!-- Sort indicator arrow:
                     ▲ = ascending, ▼ = descending, ↕ = not sorted -->
                <span class="text-xs">
                  <template v-if="sortKey === col.key && sortDirection === 'asc'">▲</template>
                  <template v-else-if="sortKey === col.key && sortDirection === 'desc'">▼</template>
                  <template v-else>↕</template>
                </span>
              </div>
            </th>
          </tr>
        </thead>

        <!-- ============ TABLE BODY ============ -->
        <tbody class="divide-y divide-gray-200">

          <!-- EMPTY STATE (v-if): shows only when there are no rows.
               colspan makes the message stretch across all columns -->
          <tr v-if="paginatedRows.length === 0">
            <td :colspan="columns.length" class="px-6 py-8 text-center text-gray-500 italic">
              No results found.
            </td>
          </tr>

          <!-- DATA ROWS (v-for): one <tr> per row on the current page.
               v-else = only render these when the empty state is NOT showing.
               :key uses the row's unique ID (_id or id)
               @click sends the clicked row up to the parent -->
          <tr
            v-else
            v-for="row in paginatedRows"
            :key="row._id || row.id"
            @click="emit('row-click', row)"
            class="hover:bg-blue-50 cursor-pointer transition-colors"
          >
            <!-- Inner loop: one <td> cell per column.
                 row[col.key] looks up that column's value in this row -->
            <td v-for="col in columns" :key="col.key" class="px-6 py-4">
              {{ row[col.key] }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- ============ PAGINATION FOOTER ============ -->
    <div class="flex items-center justify-between px-4 py-3 bg-white border-t border-gray-200 mt-2 sm:px-6">

      <!-- Left side: "Showing 1 to 10 of 45 results" -->
      <div class="text-sm text-gray-700">
        Showing <span class="font-medium">{{ startRecord }}</span> to
        <span class="font-medium">{{ endRecord }}</span> of
        <span class="font-medium">{{ totalRows }}</span> results
      </div>

      <!-- Right side: Previous / Page X of Y / Next -->
      <div class="flex space-x-2">
        <!-- Previous button: disabled (greyed out) on page 1 -->
        <button
          @click="prevPage"
          :disabled="currentPage === 1"
          class="px-3 py-1 text-sm border rounded-md disabled:opacity-40 disabled:cursor-not-allowed hover:bg-gray-100"
        >
          Previous
        </button>

        <!-- Current page indicator -->
        <span class="px-3 py-1 text-sm text-gray-600">
          Page {{ currentPage }} of {{ totalPages }}
        </span>

        <!-- Next button: disabled on the last page or when there are no rows -->
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