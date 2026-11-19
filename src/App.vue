<script setup>
import { ref, computed, onMounted } from 'vue'

const employees = ref([])
const searchQuery = ref('')
const viewMode = ref('week')
const monthName = ref('Ноябрь')

const templates = [
  { id: 'morning', name: 'Утро', time: '07:00 - 15:00', hours: 8, colorClass: 'bg-[#E5F0FF] text-[#1A56DB]' },
  { id: 'day', name: 'День', time: '10:00 - 18:00', hours: 8, colorClass: 'bg-[#EDE9FE] text-[#5B21B6]' },
  { id: 'evening', name: 'Вечер', time: '15:00 - 23:00', hours: 9, colorClass: 'bg-[#FCE7F3] text-[#BE185D]' },
]

const days = computed(() => {
  if (viewMode.value === 'week') {
    return ['пн 1', 'вт 2', 'ср 3', 'чт 4', 'пт 5', 'сб 6', 'вс 7']
  } else {
    const weekDays = ['пн', 'вт', 'ср', 'чт', 'пт', 'сб', 'вс']
    return Array.from({ length: 30 }, (_, i) => `${weekDays[i % 7]} ${i + 1}`)
  }
})

const filteredEmployees = computed(() => {
  if (!searchQuery.value) return employees.value
  return employees.value.filter(emp =>
      emp.name.toLowerCase().includes(searchQuery.value.toLowerCase())
  )
})

const shifts = ref({})
const selectedCells = ref([])
const draggedTemplate = ref(null)
const customStart = ref('09:00')
const customEnd = ref('17:00')

const activePopup = ref(null)

onMounted(async () => {
  try {
    const response = await fetch('/api/schedule')
    const data = await response.json()
    employees.value = data.employees
    shifts.value = data.shifts
  } catch (error) {
    console.error('Ошибка загрузки:', error)
  }
})

const saveSchedule = async () => {
  try {
    const response = await fetch('/api/schedule', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
      body: JSON.stringify({ shifts: shifts.value })
    })
    if (response.ok) alert('График сохранен как черновик!')
  } catch (error) {
    alert('Ошибка сети.')
  }
}

const publishSchedule = async () => {
  try {
    await fetch('/api/schedule', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
      body: JSON.stringify({ shifts: shifts.value })
    })
    const response = await fetch('/api/schedule/publish', { method: 'POST' })
    const data = await response.json()
    alert(data.message)
  } catch (error) {
    alert('Ошибка при публикации.')
  }
}

// --- ИСПРАВЛЕННАЯ ЛОГИКА КЛИКОВ ---
const handleCellClick = (empId, day, event) => {
  const key = `${empId}-${day}`

  // 1. Если зажат Ctrl — закрываем меню и управляем выделением
  if (event.ctrlKey || event.metaKey) {
    activePopup.value = null // <--- Закрываем меню при мультивыделении
    const index = selectedCells.value.findIndex(c => c.key === key)
    if (index === -1) selectedCells.value.push({ empId, day, key })
    else selectedCells.value.splice(index, 1)
    return
  }

  // 2. Если кликнули по уже выделенной ячейке (без Ctrl) — просто снимаем выделение
  if (isCellSelected(empId, day)) {
    activePopup.value = null
    selectedCells.value = selectedCells.value.filter(c => c.key !== key)
    return
  }

  // 3. Обычный клик по пустой ячейке. Сбрасываем старое выделение, если оно было.
  selectedCells.value = []

  if (shifts.value[key]) {
    // Если смена есть, по обычному клику удаляем её
    delete shifts.value[key]
    activePopup.value = null
  } else {
    // Если пусто — открываем попап
    activePopup.value = { empId, day, key }
  }
}

const applyQuickTemplate = (template) => {
  if (!activePopup.value) return
  shifts.value[activePopup.value.key] = {
    name: template.name,
    time: template.time,
    hours: template.hours,
    colorClass: template.colorClass
  }
  activePopup.value = null
}

const isCellSelected = (empId, day) => {
  return selectedCells.value.some(c => c.key === `${empId}-${day}`)
}

const getShift = (empId, day) => {
  return shifts.value[`${empId}-${day}`]
}

const applyTemplateToSelected = (template) => {
  if (selectedCells.value.length === 0) return
  selectedCells.value.forEach(cell => {
    shifts.value[cell.key] = {
      name: template.name,
      time: template.time,
      hours: template.hours,
      colorClass: template.colorClass
    }
  })
  selectedCells.value = []
}

const onDragStart = (template) => {
  draggedTemplate.value = template
}

const onDrop = (empId, day) => {
  if (!draggedTemplate.value) return
  const key = `${empId}-${day}`
  shifts.value[key] = {
    name: draggedTemplate.value.name,
    time: draggedTemplate.value.time,
    hours: draggedTemplate.value.hours,
    colorClass: draggedTemplate.value.colorClass
  }
  draggedTemplate.value = null
  selectedCells.value = selectedCells.value.filter(c => c.key !== key)
}

const applyCustomShift = () => {
  if (selectedCells.value.length === 0) return
  const startParts = customStart.value.split(':')
  const endParts = customEnd.value.split(':')
  let hours = parseInt(endParts[0]) - parseInt(startParts[0])
  if (hours < 0) hours += 24

  selectedCells.value.forEach(cell => {
    shifts.value[cell.key] = {
      name: 'Своя',
      time: `${customStart.value} - ${customEnd.value}`,
      hours: hours,
      colorClass: 'bg-gray-100 text-gray-800'
    }
  })
  selectedCells.value = []
}

const getCoverage = (day) => {
  let count = 0
  employees.value.forEach(emp => {
    if (shifts.value[`${emp.id}-${day}`]) count++
  })
  return count
}

const getInitials = (name) => {
  if (!name) return ''
  return name.split(' ').map(n => n[0]).join('')
}
</script>

<template>
  <div class="flex h-screen bg-[#F8F9FA] text-gray-800 font-sans" @click="activePopup = null">

    <!-- ЛЕВЫЙ САЙДБАР -->
    <aside class="w-64 bg-white border-r border-gray-200 flex flex-col z-50 shrink-0">
      <div class="p-6 font-bold text-xl flex items-center gap-2">
        <div class="w-6 h-6 bg-[#1A1D21] rounded flex items-center justify-center text-white text-xs">C</div>
        CRM
      </div>
      <nav class="flex-1 px-4 space-y-2">
        <a href="#" @click.prevent.stop="alert('Это тестовое задание, тут только график смен :)')" class="flex items-center gap-3 px-4 py-2 text-gray-600 hover:bg-gray-100 rounded-lg transition-colors"><span>🏠</span> Главная</a>
        <a href="#" class="flex items-center gap-3 px-4 py-2 bg-gray-100 text-gray-900 font-medium rounded-lg"><span>📅</span> Смены</a>
      </nav>
    </aside>

    <!-- ПРАВАЯ ЧАСТЬ -->
    <main class="flex-1 flex flex-col overflow-hidden relative z-10">
      <header class="h-16 bg-white border-b border-gray-200 flex items-center px-8 shrink-0">
        <div class="flex items-center gap-4">
          <button class="text-gray-400 hover:text-gray-800">←</button>
          <span class="text-gray-500 text-sm">Графики смен</span>
        </div>
      </header>

      <div class="flex-1 overflow-auto p-8 flex flex-col">
        <div class="flex justify-between items-center mb-6 shrink-0">
          <div class="flex items-center gap-4">
            <h1 class="text-3xl font-semibold">График</h1>
            <span v-if="viewMode === 'month'" class="bg-gray-200 text-gray-700 px-3 py-1 rounded-full text-sm font-medium">{{ monthName }}</span>
          </div>
          <div class="flex gap-3">
            <button @click.stop="publishSchedule" class="px-4 py-2 text-sm font-medium text-blue-600 bg-blue-50 hover:bg-blue-100 rounded-lg transition-colors">Опубликовать</button>
            <button @click.stop="saveSchedule" class="px-4 py-2 text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 rounded-lg transition-colors">Сохранить</button>
          </div>
        </div>

        <div class="flex gap-6 flex-1 min-h-0">

          <!-- ТАБЛИЦА -->
          <div class="flex-1 flex flex-col bg-white border border-gray-200 rounded-xl overflow-hidden shadow-sm">
            <div class="p-4 border-b border-gray-100 flex items-center justify-between">
              <div class="relative w-64">
                <span class="absolute left-3 top-2.5 text-gray-400">🔍</span>
                <input v-model="searchQuery" @click.stop type="text" placeholder="Поиск по ФИО" class="w-full pl-9 pr-4 py-2 bg-gray-50 border border-gray-200 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-100">
              </div>

              <div class="flex bg-gray-100 p-1 rounded-lg">
                <button @click.stop="viewMode = 'month'; selectedCells = []" :class="{'bg-[#1A1D21] text-white shadow-sm': viewMode === 'month', 'text-gray-500 hover:text-gray-800': viewMode !== 'month'}" class="px-4 py-1.5 text-sm font-medium rounded-md transition-colors">Месяц</button>
                <button @click.stop="viewMode = 'week'; selectedCells = []" :class="{'bg-[#1A1D21] text-white shadow-sm': viewMode === 'week', 'text-gray-500 hover:text-gray-800': viewMode !== 'week'}" class="px-4 py-1.5 text-sm font-medium rounded-md transition-colors">Неделя</button>
              </div>
            </div>

            <div class="overflow-auto flex-1 relative">
              <table class="w-full text-left border-collapse">
                <thead class="bg-gray-50 sticky top-0 z-10">
                <tr>
                  <th class="py-3 px-4 font-medium text-gray-500 text-sm border-b border-gray-200 w-64 bg-gray-50 sticky left-0 z-20">Сотрудник</th>
                  <th v-for="day in days" :key="day" class="py-3 px-2 font-medium text-gray-500 text-sm border-b border-gray-200 text-center min-w-[80px] bg-gray-50">
                    {{ day }}
                  </th>
                </tr>
                </thead>
                <tbody>
                <tr v-for="emp in filteredEmployees" :key="emp.id" class="border-b border-gray-100">
                  <td class="py-2 px-4 flex items-center gap-3 border-r border-gray-100 sticky left-0 bg-white z-10 shadow-[2px_0_5px_-2px_rgba(0,0,0,0.1)]">
                    <div :class="`w-8 h-8 rounded-full flex items-center justify-center text-xs font-bold ${emp.color}`">
                      {{ getInitials(emp.name) }}
                    </div>
                    <div>
                      <div class="text-sm font-medium text-gray-900">{{ emp.name }}</div>
                      <div class="text-xs text-gray-500">{{ emp.role }}</div>
                    </div>
                  </td>

                  <td
                      v-for="day in days"
                      :key="day"
                      class="p-1 border-r border-gray-100 relative group"
                      @click.stop="handleCellClick(emp.id, day, $event)"
                      @dragover.prevent
                      @drop="onDrop(emp.id, day)"
                  >
                    <div v-if="isCellSelected(emp.id, day)" class="w-full h-12 rounded-lg border-2 border-blue-400 bg-blue-50 flex items-center justify-center transition-all cursor-pointer">
                      <span class="text-blue-500 font-medium">✓</span>
                    </div>

                    <div v-else-if="getShift(emp.id, day)" :class="`w-full h-12 rounded-lg flex flex-col items-center justify-center text-[11px] leading-tight px-1 cursor-pointer ${getShift(emp.id, day).colorClass}`">
                      <span class="font-bold">{{ getShift(emp.id, day).name }}</span>
                      <span class="opacity-80">{{ getShift(emp.id, day).hours }}ч</span>
                    </div>

                    <div v-else class="w-full h-12 rounded-lg border border-dashed border-transparent group-hover:border-gray-300 flex items-center justify-center transition-all bg-gray-50/50 hover:bg-gray-100 cursor-pointer">
                    </div>

                    <div v-if="activePopup && activePopup.key === `${emp.id}-${day}`" @click.stop class="absolute top-full left-1/2 -translate-x-1/2 mt-1 w-48 bg-white border border-gray-200 rounded-xl shadow-xl z-50 p-2">
                      <div class="text-[10px] text-gray-400 font-medium mb-1 px-2 uppercase tracking-wider">Выберите смену</div>
                      <button @click.stop.prevent="applyQuickTemplate(tpl)" v-for="tpl in templates" :key="tpl.id" class="w-full text-left px-3 py-2 text-sm hover:bg-gray-50 rounded-lg flex justify-between items-center group/menu">
                        <span>{{ tpl.name }}</span>
                        <span class="text-xs text-gray-400">{{ tpl.hours }}ч</span>
                      </button>
                    </div>
                  </td>
                </tr>

                <tr class="bg-red-50/30">
                  <td class="py-2 px-4 text-sm text-gray-500 font-medium border-r border-gray-100 sticky left-0 bg-[#FFF5F5] z-10">Покрытие</td>
                  <td v-for="day in days" :key="day" class="p-1 border-r border-gray-100 text-center">
                    <div :class="`w-full h-8 flex items-center justify-center text-sm rounded-md ${getCoverage(day) > 0 ? 'bg-blue-100 text-blue-700 font-bold' : 'text-gray-400 bg-red-50'}`">
                      {{ getCoverage(day) }}
                    </div>
                  </td>
                </tr>
                </tbody>
              </table>
            </div>
          </div>

          <!-- ПРАВАЯ ПАНЕЛЬ -->
          <div class="w-72 flex flex-col gap-4 shrink-0" @click.stop>
            <div class="bg-white border border-gray-200 rounded-xl p-4 shadow-sm">
              <h3 class="text-sm font-semibold text-gray-700 mb-3">Шаблоны смен</h3>
              <div class="space-y-2">
                <div
                    v-for="tpl in templates"
                    :key="tpl.id"
                    draggable="true"
                    @dragstart="onDragStart(tpl)"
                    @click="applyTemplateToSelected(tpl)"
                    class="p-3 rounded-lg border border-gray-100 cursor-grab active:cursor-grabbing hover:border-blue-400 hover:shadow-sm transition-all flex justify-between items-center group bg-white"
                    :class="{'opacity-50 pointer-events-none': selectedCells.length === 0 && !draggedTemplate}"
                >
                  <div>
                    <div class="text-sm font-medium text-gray-900">{{ tpl.name }}</div>
                    <div class="text-xs text-gray-500 mt-0.5">{{ tpl.time }} · {{ tpl.hours }}ч</div>
                  </div>
                  <div class="w-6 h-6 rounded-full bg-gray-100 text-gray-500 text-xs flex items-center justify-center group-hover:bg-blue-100 group-hover:text-blue-600">
                    {{ tpl.hours }}
                  </div>
                </div>
              </div>
              <div v-if="selectedCells.length > 0" class="mt-4 text-xs text-center text-blue-600 bg-blue-50 py-1.5 rounded">
                Выделено ячеек: {{ selectedCells.length }}
              </div>
              <div v-else class="mt-4 text-xs text-center text-gray-400 bg-gray-50 py-1.5 rounded">
                Клик — меню. Ctrl+Клик — выделить.
              </div>
            </div>

            <div class="bg-white border border-gray-200 rounded-xl p-4 shadow-sm">
              <h3 class="text-sm font-semibold text-gray-700 mb-3">Своя смена</h3>
              <div class="flex gap-2 mb-3">
                <div class="flex-1">
                  <label class="text-[10px] text-gray-500 mb-1 block">Начало</label>
                  <input type="time" v-model="customStart" class="w-full px-2 py-1.5 bg-gray-50 border border-gray-200 rounded text-sm text-center focus:outline-none focus:border-blue-400">
                </div>
                <div class="flex-1">
                  <label class="text-[10px] text-gray-500 mb-1 block">Конец</label>
                  <input type="time" v-model="customEnd" class="w-full px-2 py-1.5 bg-gray-50 border border-gray-200 rounded text-sm text-center focus:outline-none focus:border-blue-400">
                </div>
              </div>
              <button
                  @click="applyCustomShift"
                  class="w-full py-2 bg-blue-50 text-blue-600 text-sm font-medium rounded-lg hover:bg-blue-100 transition-colors flex items-center justify-center gap-2"
                  :class="{'opacity-50 pointer-events-none': selectedCells.length === 0}"
              >
                <span>+</span> Поставить в выделенные
              </button>
            </div>
          </div>

        </div>
      </div>
    </main>
  </div>
</template>