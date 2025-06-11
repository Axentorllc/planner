<template>
  <div class="timeline-roster-view">
    <!-- Header Controls -->
    <div class="timeline-header mb-6 p-6 bg-gradient-to-r from-white to-gray-50 dark:from-gray-800 dark:to-gray-900 rounded-xl shadow-lg border border-gray-200 dark:border-gray-700">
      <div class="flex flex-col lg:flex-row justify-between items-start lg:items-center gap-6">
        <!-- Left Section -->
        <div class="flex items-center gap-4">
          <div class="view-toggle flex bg-gray-100 dark:bg-gray-700 rounded-xl p-1.5">
            <button 
              v-for="mode in viewModes" 
              :key="mode.value"
              @click="currentViewMode = mode.value"
              :class="[
                'px-4 py-2 rounded-lg text-sm font-medium transition-all duration-300 flex items-center gap-2',
                currentViewMode ===   mode.value 
                  ? 'bg-white dark:bg-gray-600 shadow-md text-blue-600 dark:text-blue-400' 
                  : 'text-gray-600 dark:text-gray-300 hover:text-gray-900 dark:hover:text-white'
              ]"
            >
              <FeatherIcon :name="mode.icon" class="w-4 h-4" />
              {{ mode.label }}
            </button>
          </div>

          <div class="date-navigation flex items-center gap-3 bg-white dark:bg-gray-800 rounded-xl p-2 shadow-sm border border-gray-200 dark:border-gray-700">
            <Button 
              variant="ghost" 
              theme="gray" 
              size="sm"
              @click="navigateDate(-1)"
            >
              <FeatherIcon name="chevron-left" class="w-4 h-4" />
            </Button>
            
            <div class="px-4 py-2 bg-gradient-to-r from-blue-50 to-indigo-50 dark:from-blue-900/20 dark:to-indigo-900/20 rounded-lg text-sm font-semibold text-blue-900 dark:text-blue-100 min-w-[200px] text-center">
              {{ formatDateRange() }}
            </div>
            
            <Button 
              variant="ghost" 
              theme="gray" 
              size="sm"
              @click="navigateDate(1)"
            >
              <FeatherIcon name="chevron-right" class="w-4 h-4" />
            </Button>
            
            <Button 
              variant="ghost" 
              theme="blue" 
              size="sm"
              @click="goToToday"
            >
              Today
            </Button>
          </div>
        </div>

        <!-- Right Section -->
        <div class="flex items-center gap-4">
          <div class="flex items-center gap-3 bg-white dark:bg-gray-800 rounded-xl p-3 shadow-sm border border-gray-200 dark:border-gray-700">
            <FeatherIcon name="users" class="w-4 h-4 text-gray-500 dark:text-gray-400" />
            <span class="text-sm font-medium text-gray-700 dark:text-gray-300">{{ filteredAssignees.length }} team members</span>
            <div class="text-xs text-gray-500 dark:text-gray-400">
              ({{ totalScheduledHours }}h scheduled)
            </div>
          </div>

          <div class="flex items-center gap-2">
            <input
              v-model="assigneeSearch"
              type="text"
              placeholder="Search members..."
              class="px-3 py-2 text-sm border border-gray-200 dark:border-gray-600 rounded-lg bg-white dark:bg-gray-800 text-gray-900 dark:text-white focus:outline-none focus:ring-2 focus:ring-blue-500"
            />
            
            <Button 
              variant="ghost" 
              theme="gray" 
              size="sm"
              @click="showBacklog = !showBacklog"
              :class="showBacklog ? 'bg-green-50 dark:bg-green-900/20 text-green-600 dark:text-green-400' : ''"
            >
              <FeatherIcon name="inbox" class="w-4 h-4 mr-2" />
              Backlog ({{ unscheduledTasks.length }})
            </Button>

            <Button 
              variant="ghost" 
              theme="gray" 
              size="sm"
              @click="showFilters = !showFilters"
            >
              <FeatherIcon name="filter" class="w-4 h-4 mr-2" />
              Filters
            </Button>
          </div>
        </div>
      </div>

      <!-- Filters Panel -->
      <div v-if="showFilters" class="mt-4 p-4 bg-gray-50 dark:bg-gray-800 rounded-lg border">
        <div class="flex flex-wrap gap-4">
          <div class="flex items-center gap-2">
            <label class="text-sm font-medium text-gray-700 dark:text-gray-300">Project:</label>
            <select 
              v-model="selectedProject"
              class="px-3 py-1.5 text-sm border border-gray-300 dark:border-gray-600 rounded bg-white dark:bg-gray-700 text-gray-900 dark:text-white"
            >
              <option value="">All Projects</option>
              <option v-for="project in uniqueProjects" :key="project" :value="project">
                {{ project }}
              </option>
            </select>
          </div>
          
          <div class="flex items-center gap-2">
            <label class="text-sm font-medium text-gray-700 dark:text-gray-300">Priority:</label>
            <select 
              v-model="selectedPriority"
              class="px-3 py-1.5 text-sm border border-gray-300 dark:border-gray-600 rounded bg-white dark:bg-gray-700 text-gray-900 dark:text-white"
            >
              <option value="">All Priorities</option>
              <option value="High">High</option>
              <option value="Medium">Medium</option>
              <option value="Low">Low</option>
            </select>
          </div>

          <div class="flex items-center gap-2">
            <label class="text-sm font-medium text-gray-700 dark:text-gray-300">Status:</label>
            <select 
              v-model="selectedStatus"
              class="px-3 py-1.5 text-sm border border-gray-300 dark:border-gray-600 rounded bg-white dark:bg-gray-700 text-gray-900 dark:text-white"
            >
              <option value="">All Statuses</option>
              <option value="To Do">To Do</option>
              <option value="In Progress">In Progress</option>
              <option value="Review">Review</option>
              <option value="Done">Done</option>
            </select>
          </div>
        </div>
      </div>
    </div>

    <!-- Main Timeline Grid -->
    <div class="flex gap-6">
      <!-- Timeline Table -->
      <div class="flex-1">
        <div 
          class="rounded-xl border overflow-hidden shadow-lg bg-white dark:bg-gray-800"
          :class="{ 'animate-pulse pointer-events-none': loading }"
        >
          <!-- Fixed header for scrolling -->
          <div class="overflow-x-auto timeline-scroll-container" style="max-height: 70vh;">
            <table class="border-separate border-spacing-0 w-full min-w-[1200px]">
              <thead class="sticky top-0 z-20">
                <tr class="bg-gradient-to-r from-gray-50 to-gray-100 dark:from-gray-700 dark:to-gray-800">
                  <!-- Assignee Column -->
                  <th class="assignee-column p-4 border-b border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-800 sticky left-0 z-30">
                    <div class="flex items-center justify-between">
                      <div class="flex items-center gap-2">
                        <FeatherIcon name="users" class="w-4 h-4 text-gray-500 dark:text-gray-400" />
                        <span class="font-semibold text-gray-900 dark:text-white">Team Members</span>
                      </div>
                      <div class="flex items-center gap-2">
                        <FeatherIcon name="zap" class="w-4 h-4 text-gray-500 dark:text-gray-400" />
                        <span class="text-xs font-medium text-gray-500 dark:text-gray-400">Capacity</span>
                      </div>
                    </div>
                  </th>

                  <!-- Date Columns -->
                  <th
                    v-for="(date, idx) in dateColumns"
                    :key="date.key"
                    class="date-column font-medium border-b border-gray-200 dark:border-gray-600 p-3 text-center"
                    :class="{ 
                      'border-l border-gray-200 dark:border-gray-600': idx >= 0,
                      'bg-blue-50 dark:bg-blue-900/20': date.isToday,
                      'bg-red-50 dark:bg-red-900/10': date.isWeekend && !date.isToday,
                      'bg-yellow-50 dark:bg-yellow-900/10': date.isHoliday
                    }"
                  >
                    <div class="text-sm font-semibold text-gray-900 dark:text-white">
                      {{ date.label }}
                    </div>
                    <div class="text-xs text-gray-500 dark:text-gray-400 mt-1">
                      {{ date.sublabel }}
                    </div>
                    <div v-if="date.isToday" class="w-2 h-2 bg-blue-500 rounded-full mx-auto mt-1"></div>
                    <div v-if="date.isHoliday" class="text-xs text-yellow-600 dark:text-yellow-400 mt-1">
                      Holiday
                    </div>
                    <!-- Daily capacity indicator -->
                    <div class="text-xs text-gray-400 mt-1">
                      {{ getDayCapacity(date.date) }}h
                    </div>
                  </th>
                </tr>
              </thead>
              <tbody>
                <tr 
                  v-for="(assignee, rowIdx) in filteredAssignees" 
                  :key="assignee.id"
                  class="border-b border-gray-100 dark:border-gray-700 last:border-b-0 hover:bg-gray-50/50 dark:hover:bg-gray-700/30 transition-all duration-200"
                  :class="{ 'bg-red-50/30 dark:bg-red-900/10': isAssigneeOverloaded(assignee.id) }"
                >
                  <!-- Assignee Info Column -->
                  <td class="assignee-column p-4 border-r border-gray-200 dark:border-gray-600 bg-gradient-to-r from-gray-50/50 to-transparent dark:from-gray-800/50 sticky left-0 z-10">
                    <div class="flex items-center justify-between h-full min-h-[80px]">
                      <div class="flex items-center gap-4">
                        <div class="relative">
                          <Avatar 
                            :image="assignee.image" 
                            :label="assignee.name"
                            size="md"
                            class="ring-2 ring-white dark:ring-gray-700 shadow-sm"
                          />
                          <div 
                            class="absolute -bottom-1 -right-1 w-4 h-4 rounded-full border-2 border-white dark:border-gray-800" 
                            :class="getStatusIndicatorClass(assignee.id)"
                          ></div>
                        </div>
                        <div class="min-w-0 flex-1">
                          <div class="font-semibold text-gray-900 dark:text-white text-sm truncate">
                            {{ assignee.name }}
                          </div>
                          <div class="text-xs text-gray-500 dark:text-gray-400 mt-1 truncate">
                            {{ assignee.role || assignee.department || 'Team Member' }}
                          </div>
                          <div class="text-xs text-gray-600 dark:text-gray-400 mt-1">
                            {{ getAssigneeScheduledHours(assignee.id) }}h / {{ getAssigneeCapacity(assignee.id) }}h
                          </div>
                          <div class="text-xs text-gray-500 dark:text-gray-400">
                            {{ getAssigneeTaskCount(assignee.id) }} tasks
                          </div>
                        </div>
                      </div>
                      
                      <div class="text-right flex-shrink-0">
                        <div class="text-sm font-bold mb-2" :class="getUtilizationTextClass(getAssigneeUtilization(assignee.id))">
                          {{ Math.round(getAssigneeUtilization(assignee.id)) }}%
                        </div>
                        <div class="w-20 h-3 bg-gray-200 dark:bg-gray-600 rounded-full overflow-hidden shadow-inner">
                          <div 
                            class="h-full transition-all duration-500 ease-out rounded-full"
                            :class="getCapacityBarClass(getAssigneeUtilization(assignee.id))"
                            :style="{ width: `${Math.min(getAssigneeUtilization(assignee.id), 100)}%` }"
                          ></div>
                        </div>
                        <div class="text-xs mt-1 font-medium" :class="getUtilizationLabelClass(getAssigneeUtilization(assignee.id))">
                          {{ getUtilizationLabel(getAssigneeUtilization(assignee.id)) }}
                        </div>
                      </div>
                    </div>
                  </td>

                  <!-- Date Cells -->
                  <td
                    v-for="(date, colIdx) in dateColumns"
                    :key="`${assignee.id}-${date.key}`"
                    class="date-cell p-2 relative min-h-[80px] align-top"
                    :data-date="date.key"
                    :class="{
                      'border-l border-gray-200 dark:border-gray-600': colIdx >= 0,
                      'bg-blue-50/50 dark:bg-blue-900/10': date.isToday,
                      'bg-red-50/30 dark:bg-red-900/10': date.isWeekend && !date.isToday,
                      'bg-yellow-50/30 dark:bg-yellow-900/10': date.isHoliday,
                      'hover:bg-gray-50 dark:hover:bg-gray-700/20': !date.isToday && !date.isWeekend && !date.isHoliday,
                      'drop-zone-active': isDragOver && dragOverDate === date.key && dragOverAssignee === assignee.id,
                      'resize-highlight': isResizing && resizeDirection?.value === 'end' && date.date <= resizeEndDate?.value && date.date >= resizeStartDate?.value,
                      'overallocated': isDayOverallocated(assignee.id, date.date)
                    }"
                    @drop="handleDrop($event, assignee.id, date.date)"
                    @dragover.prevent="handleDragOver($event, assignee.id, date.key)"
                    @dragenter.prevent="handleDragEnter($event, assignee.id, date.key)"
                    @dragleave="handleDragLeave"
                    @mouseenter="handleCellMouseEnter(assignee.id, date.key)"
                    @mouseleave="handleCellMouseLeave"
                    @click="handleCellClick(assignee.id, date.date)"
                  >
                    <!-- Today/Weekend/Holiday Indicators -->
                    <div v-if="date.isToday" class="absolute left-1/2 top-0 w-0.5 h-full bg-blue-500 transform -translate-x-1/2 opacity-30"></div>
                    <div v-if="date.isWeekend && !date.isToday" class="absolute top-1 right-1 w-2 h-2 bg-gray-400 rounded-full opacity-50"></div>
                    <div v-if="date.isHoliday" class="absolute top-1 left-1 w-2 h-2 bg-yellow-500 rounded-full"></div>
                    
                    <!-- Daily Hours Indicator -->
                    <div class="absolute top-1 left-1/2 transform -translate-x-1/2 text-xs text-gray-400">
                      {{ getDayScheduledHours(assignee.id, date.date) }}h
                    </div>
                    
                    <!-- Task Blocks -->
                    <div class="space-y-1 mt-4">
                      <div
                        v-for="task in getTasksForDateAndAssignee(assignee.id, date.date)"
                        :key="task.id"
                        class="task-block group rounded-lg shadow-sm hover:shadow-md transition-all duration-200 p-2 relative cursor-move"
                        :class="[
                          getTaskBlockClass(task),
                          { 'resizing': isResizing && resizingTask?.id === task.id }
                        ]"
                        @click.stop="$emit('taskClick', task.id)"
                        draggable="true"
                        @dragstart="handleDragStart($event, task)"
                        :title="`${task.title} - ${task.project} (${task.duration}h)`"
                      >
                        <!-- Task content -->
                        <div class="text-xs text-white">
                          <div class="font-semibold truncate mb-1 text-[11px]">{{ task.title }}</div>
                          <div class="text-white/80 truncate text-[10px] mb-1">{{ task.project }}</div>
                          <div class="flex items-center justify-between">
                            <div class="text-white/70 text-[10px]">{{ task.duration }}h</div>
                            <div class="flex items-center gap-1">
                              <div 
                                class="w-1.5 h-1.5 rounded-full" 
                                :class="getPriorityIndicatorClass(task.priority)"
                              ></div>
                              <div 
                                class="w-1.5 h-1.5 rounded-full"
                                :class="getStatusIndicatorClass(task.status)"
                              ></div>
                            </div>
                          </div>
                        </div>

                        <!-- Resize handles -->
                        <div
                          v-if="task.startDate && task.endDate"
                          class="absolute left-0 top-0 bottom-0 w-2 cursor-ew-resize hover:bg-white/20"
                          @mousedown.stop="startResize($event, task, 'start')"
                        ></div>
                        <div
                          v-if="task.startDate && task.endDate"
                          class="absolute right-0 top-0 bottom-0 w-2 cursor-ew-resize hover:bg-white/20"
                          @mousedown.stop="startResize($event, task, 'end')"
                        ></div>
                        
                        <!-- Task progress bar -->
                        <div v-if="task.progress > 0" class="absolute bottom-0 left-0 right-0 h-1 bg-white/20 rounded-b-lg overflow-hidden">
                          <div 
                            class="h-full bg-white/60 transition-all duration-300"
                            :style="{ width: `${task.progress}%` }"
                          ></div>
                        </div>
                      </div>
                    </div>

                    <!-- Add Task Button -->
                    <Button
                      v-if="hoveredCell.assignee === assignee.id && hoveredCell.date === date.key && !date.isWeekend"
                      variant="outline"
                      icon="plus"
                      size="sm"
                      class="border-2 border-dashed border-gray-300 dark:border-gray-600 w-full mt-2 text-gray-500 dark:text-gray-400 hover:border-blue-400 hover:text-blue-600 text-xs py-1"
                      @click.stop="$emit('addTask', { assigneeId: assignee.id, date: date.date })"
                    >
                      Add Task
                    </Button>

                    <!-- Conflict Warning -->
                    <div v-if="isDayOverallocated(assignee.id, date.date)" class="absolute bottom-1 right-1">
                      <FeatherIcon name="alert-triangle" class="w-3 h-3 text-red-500" />
                    </div>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- Summary Bar -->
        <div class="mt-4 p-4 bg-white dark:bg-gray-800 rounded-xl shadow-sm border border-gray-200 dark:border-gray-700">
          <div class="flex items-center justify-between text-sm">
            <div class="flex items-center gap-6">
              <div class="flex items-center gap-2">
                <div class="w-3 h-3 bg-green-500 rounded-full"></div>
                <span class="text-gray-600 dark:text-gray-400">Available: {{ availableMembers }}</span>
              </div>
              <div class="flex items-center gap-2">
                <div class="w-3 h-3 bg-yellow-500 rounded-full"></div>
                <span class="text-gray-600 dark:text-gray-400">High Utilization: {{ highUtilizationMembers }}</span>
              </div>
              <div class="flex items-center gap-2">
                <div class="w-3 h-3 bg-red-500 rounded-full"></div>
                <span class="text-gray-600 dark:text-gray-400">Overloaded: {{ overloadedMembers }}</span>
              </div>
            </div>
            <div class="text-gray-600 dark:text-gray-400">
              Total Capacity: {{ totalCapacity }}h | Scheduled: {{ totalScheduledHours }}h
            </div>
          </div>
        </div>
      </div>

      <!-- Backlog Panel -->
      <BacklogPanel
        v-if="showBacklog"
        :key="showBacklog"
        :tasks="filteredUnscheduledTasks"
        :loading="loading"
        @taskDragStart="handleDragStart"
        @taskClick="$emit('taskClick', $event)"
        @addTask="$emit('addTask', $event)"
        class="w-80 flex-shrink-0"
      />
    </div>

    <!-- Loading Overlay -->
    <div v-if="loading" class="fixed inset-0 bg-black/20 backdrop-blur-sm flex items-center justify-center z-50">
      <div class="bg-white dark:bg-gray-800 rounded-xl p-6 shadow-xl">
        <div class="flex items-center gap-3">
          <div class="w-6 h-6 border-2 border-blue-500 border-t-transparent rounded-full animate-spin"></div>
          <span class="text-gray-900 dark:text-white font-medium">Loading timeline...</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { Avatar, Button, FeatherIcon } from 'frappe-ui'
import BacklogPanel from './BacklogPanel.vue'

const props = defineProps({
  assignees: {
    type: Array,
    required: true,
    default: () => []
  },
  tasks: {
    type: Array,
    required: true,
    default: () => []
  },
  loading: {
    type: Boolean,
    default: false
  },
  holidays: {
    type: Array,
    default: () => []
  },
  workingHours: {
    type: Object,
    default: () => ({
      monday: 8,
      tuesday: 8,
      wednesday: 8,
      thursday: 8,
      friday: 8,
      saturday: 0,
      sunday: 0
    })
  }
})

const emit = defineEmits(['taskClick', 'taskMove', 'addTask', 'cellClick'])

// View state
const currentViewMode = ref('week')
const currentDate = ref(new Date())
const showBacklog = ref(true)
const showFilters = ref(false)
const assigneeSearch = ref('')
const selectedProject = ref('')
const selectedPriority = ref('')
const selectedStatus = ref('')

// Drag and drop state
const isDragOver = ref(false)
const dragOverAssignee = ref(null)
const dragOverDate = ref(null)
const hoveredCell = ref({ assignee: '', date: '' })
const isResizing = ref(false)
const resizingTask = ref(null)
const resizeStartDate = ref(null)
const resizeEndDate = ref(null)
const resizeDirection = ref(null) // 'start' or 'end'

// Constants
const viewModes = [
  { label: 'Week', value: 'week', icon: 'calendar' },
  { label: 'Month', value: 'month', icon: 'calendar' },
  { label: '2 Weeks', value: 'biweek', icon: 'calendar' }
]

// Computed properties
const dateColumns = computed(() => {
  const columns = []
  const today = new Date()
  let startDate, daysToShow
  
  switch (currentViewMode.value) {
    case 'week':
      startDate = getWeekStart(currentDate.value)
      daysToShow = 7
      break
    case 'biweek':
      startDate = getWeekStart(currentDate.value)
      daysToShow = 14
      break
    case 'month':
      startDate = new Date(currentDate.value.getFullYear(), currentDate.value.getMonth(), 1)
      daysToShow = getDaysInMonth(currentDate.value)
      break
    default:
      startDate = getWeekStart(currentDate.value)
      daysToShow = 7
  }
  
  for (let i = 0; i < daysToShow; i++) {
    const date = new Date(startDate)
    date.setDate(startDate.getDate() + i)
    
    const isToday = date.toDateString() === today.toDateString()
    const isWeekend = date.getDay() === 0 || date.getDay() === 6
    const isHoliday = props.holidays.some(holiday => 
      new Date(holiday.date).toDateString() === date.toDateString()
    )
    
    columns.push({
      key: date.toISOString().split('T')[0],
      date: date,
      label: date.getDate().toString(),
      sublabel: date.toLocaleDateString('en-US', { weekday: 'short' }),
      isToday,
      isWeekend,
      isHoliday
    })
  }
  
  return columns
})

const filteredAssignees = computed(() => {
  let filtered = props.assignees
  
  if (assigneeSearch.value) {
    filtered = filtered.filter(assignee => 
      assignee.name.toLowerCase().includes(assigneeSearch.value.toLowerCase()) ||
      (assignee.role || '').toLowerCase().includes(assigneeSearch.value.toLowerCase()) ||
      (assignee.department || '').toLowerCase().includes(assigneeSearch.value.toLowerCase())
    )
  }
  
  return filtered
})

const filteredTasks = computed(() => {
  let tasks = props.tasks
  
  if (selectedProject.value) {
    tasks = tasks.filter(task => task.project === selectedProject.value)
  }
  
  if (selectedPriority.value) {
    tasks = tasks.filter(task => task.priority === selectedPriority.value)
  }
  
  if (selectedStatus.value) {
    tasks = tasks.filter(task => task.status === selectedStatus.value)
  }
  
  return tasks
})

const unscheduledTasks = computed(() => {
  return filteredTasks.value.filter(task => !task.isScheduled || !task.assignee)
})

const filteredUnscheduledTasks = computed(() => {
  return unscheduledTasks.value
})

const uniqueProjects = computed(() => {
  const projects = [...new Set(props.tasks.map(task => task.project).filter(Boolean))]
  return projects.sort()
})

// Summary statistics
const totalCapacity = computed(() => {
  return filteredAssignees.value.reduce((sum, assignee) => {
    return sum + getAssigneeCapacity(assignee.id)
  }, 0)
})

const totalScheduledHours = computed(() => {
  return filteredTasks.value
    .filter(task => task.isScheduled && task.assignee)
    .reduce((sum, task) => sum + (task.duration || 0), 0)
})

const availableMembers = computed(() => {
  return filteredAssignees.value.filter(assignee => 
    getAssigneeUtilization(assignee.id) < 50
  ).length
})

const highUtilizationMembers = computed(() => {
  return filteredAssignees.value.filter(assignee => {
    const util = getAssigneeUtilization(assignee.id)
    return util >= 50 && util < 100
  }).length
})

const overloadedMembers = computed(() => {
  return filteredAssignees.value.filter(assignee => 
    getAssigneeUtilization(assignee.id) >= 100
  ).length
})

// Utility methods
const getWeekStart = (date) => {
  const d = new Date(date)
  const day = d.getDay()
  const diff = d.getDate() - day + (day === 0 ? -6 : 1)
  return new Date(d.setDate(diff))
}

const getDaysInMonth = (date) => {
  return new Date(date.getFullYear(), date.getMonth() + 1, 0).getDate()
}

const navigateDate = (direction) => {
  const newDate = new Date(currentDate.value)
  let daysToMove
  
  switch (currentViewMode.value) {
    case 'week':
      daysToMove = 7
      break
    case 'biweek':
      daysToMove = 14
      break
    case 'month':
      newDate.setMonth(newDate.getMonth() + direction)
      currentDate.value = newDate
      return
    default:
      daysToMove = 7
  }
  
  newDate.setDate(newDate.getDate() + (direction * daysToMove))
  currentDate.value = newDate
}

const goToToday = () => {
  currentDate.value = new Date()
}

const formatDateRange = () => {
  switch (currentViewMode.value) {
    case 'week':
    case 'biweek':
      const start = getWeekStart(currentDate.value)
      const end = new Date(start)
      const days = currentViewMode.value === 'week' ? 6 : 13
      end.setDate(start.getDate() + days)
      return `${start.toLocaleDateString('en-US', { month: 'short', day: 'numeric' })} - ${end.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })}`
    case 'month':
      return currentDate.value.toLocaleDateString('en-US', { year: 'numeric', month: 'long' })
    default:
      return ''
  }
}

const getTasksForDateAndAssignee = (assigneeId, date) => {
  const dateStr = date.toISOString().split('T')[0]
  
  return filteredTasks.value.filter(task => {
    if (!task.isScheduled || task.assignee !== assigneeId) return false
    
    // Handle both single date and date range tasks
    if (task.startDate && task.endDate) {
      const taskStart = new Date(task.startDate).toISOString().split('T')[0]
      const taskEnd = new Date(task.endDate).toISOString().split('T')[0]
      return dateStr >= taskStart && dateStr <= taskEnd
    } else if (task.date) {
      const taskDate = new Date(task.date).toISOString().split('T')[0]
      return dateStr === taskDate
    }
    
    return false
  })
}

const getAssigneeScheduledHours = (assigneeId) => {
  const assigneeTasks = filteredTasks.value.filter(t => t.assignee === assigneeId && t.isScheduled)
  return assigneeTasks.reduce((sum, t) => sum + (t.duration || 0), 0)
}

const getAssigneeCapacity = (assigneeId) => {
  const assignee = props.assignees.find(a => a.id === assigneeId)
  if (!assignee) return 40
  
  // Calculate capacity based on date range and working hours
  const dateRange = dateColumns.value.length
  const weeklyCapacity = Object.values(props.workingHours).reduce((sum, hours) => sum + hours, 0)
  
  if (currentViewMode.value === 'week') {
    return weeklyCapacity
  } else if (currentViewMode.value === 'biweek') {
    return weeklyCapacity * 2
  } else {
    // Monthly view - calculate based on working days
    const workingDays = dateColumns.value.filter(date => !date.isWeekend && !date.isHoliday).length
    return workingDays * 8 // Assuming 8 hours per working day
  }
}

const getAssigneeUtilization = (assigneeId) => {
  const scheduledHours = getAssigneeScheduledHours(assigneeId)
  const capacity = getAssigneeCapacity(assigneeId)
  return capacity > 0 ? (scheduledHours / capacity) * 100 : 0
}

const getAssigneeTaskCount = (assigneeId) => {
  return filteredTasks.value.filter(t => t.assignee === assigneeId && t.isScheduled).length
}

const getDayCapacity = (date) => {
  const dayOfWeek = date.toLocaleDateString('en-US', { weekday: 'long' }).toLowerCase()
  const isHoliday = props.holidays.some(holiday => 
    new Date(holiday.date).toDateString() === date.toDateString()
  )
  
  if (isHoliday) return 0
  return props.workingHours[dayOfWeek] || 0
}

const getDayScheduledHours = (assigneeId, date) => {
  const tasks = getTasksForDateAndAssignee(assigneeId, date)
  return tasks.reduce((sum, task) => {
    // For multi-day tasks, distribute hours across days
    if (task.startDate && task.endDate) {
      const taskStart = new Date(task.startDate)
      const taskEnd = new Date(task.endDate)
      const taskDays = Math.ceil((taskEnd - taskStart) / (1000 * 60 * 60 * 24)) + 1
      return sum + ((task.duration || 0) / taskDays)
    }
    return sum + (task.duration || 0)
  }, 0)
}

const isDayOverallocated = (assigneeId, date) => {
  const scheduledHours = getDayScheduledHours(assigneeId, date)
  const dayCapacity = getDayCapacity(date)
  return scheduledHours > dayCapacity
}

const isAssigneeOverloaded = (assigneeId) => {
  return getAssigneeUtilization(assigneeId) >= 100
}

const getTaskBlockClass = (task) => {
  const baseClass = 'bg-gradient-to-r border-l-4'
  
  // Priority-based coloring
  let colorClass = ''
  switch (task.priority) {
    case 'High':
      colorClass = 'from-red-500 to-red-600 border-red-700'
      break
    case 'Medium':
      colorClass = 'from-blue-500 to-blue-600 border-blue-700'
      break
    case 'Low':
      colorClass = 'from-green-500 to-green-600 border-green-700'
      break
    default:
      colorClass = 'from-gray-500 to-gray-600 border-gray-700'
  }
  
  // Status-based opacity
  let opacityClass = ''
  switch (task.status) {
    case 'Done':
      opacityClass = 'opacity-70'
      break
    case 'In Progress':
      opacityClass = 'opacity-90'
      break
    default:
      opacityClass = 'opacity-100'
  }
  
  return `${baseClass} ${colorClass} ${opacityClass}`
}

const getPriorityIndicatorClass = (priority) => {
  switch (priority) {
    case 'High': return 'bg-red-300'
    case 'Medium': return 'bg-blue-300'
    case 'Low': return 'bg-green-300'
    default: return 'bg-gray-300'
  }
}

const getStatusIndicatorClass = (status) => {
  switch (status) {
    case 'Done': return 'bg-green-300'
    case 'In Progress': return 'bg-yellow-300'
    case 'Review': return 'bg-purple-300'
    case 'To Do': return 'bg-gray-300'
    default: return 'bg-gray-300'
  }
}

const getCapacityBarClass = (utilization) => {
  if (utilization >= 100) return 'bg-red-500'
  if (utilization >= 80) return 'bg-yellow-500'
  if (utilization >= 50) return 'bg-green-500'
  return 'bg-blue-500'
}

const getUtilizationTextClass = (utilization) => {
  if (utilization >= 100) return 'text-red-600 dark:text-red-400'
  if (utilization >= 80) return 'text-yellow-600 dark:text-yellow-400'
  if (utilization >= 50) return 'text-green-600 dark:text-green-400'
  return 'text-blue-600 dark:text-blue-400'
}

const getUtilizationLabelClass = (utilization) => {
  if (utilization >= 100) return 'text-red-600 dark:text-red-400'
  if (utilization >= 80) return 'text-yellow-600 dark:text-yellow-400'
  return 'text-green-600 dark:text-green-400'
}

const getUtilizationLabel = (utilization) => {
  if (utilization >= 100) return 'Overloaded'
  if (utilization >= 80) return 'High'
  if (utilization >= 50) return 'Optimal'
  if (utilization > 0) return 'Low'
  return 'Available'
}

// Drag and drop handlers
const handleDragStart = (event, task) => {
  event.dataTransfer.setData('application/json', JSON.stringify(task))
  event.dataTransfer.effectAllowed = 'move'
  
  // Add visual feedback
  event.target.style.opacity = '0.5'
}

const handleDragOver = (event, assigneeId, dateKey) => {
  event.preventDefault()
  event.dataTransfer.dropEffect = 'move'
  
  isDragOver.value = true
  dragOverAssignee.value = assigneeId
  dragOverDate.value = dateKey
}

const handleDragEnter = (event, assigneeId, dateKey) => {
  event.preventDefault()
  dragOverAssignee.value = assigneeId
  dragOverDate.value = dateKey
}

const handleDragLeave = (event) => {
  // Only reset if we're actually leaving the drop zone
  if (!event.currentTarget.contains(event.relatedTarget)) {
    isDragOver.value = false
    dragOverAssignee.value = null
    dragOverDate.value = null
  }
}

const handleDrop = (event, assigneeId, date) => {
  event.preventDefault()
  
  // Reset drag state
  isDragOver.value = false
  dragOverAssignee.value = null
  dragOverDate.value = null
  
  try {
    const taskData = JSON.parse(event.dataTransfer.getData('application/json'))
    
    // Calculate end date based on task duration
    const startDate = new Date(date)
    const endDate = new Date(startDate)
    
    // If task has duration, calculate end date
    if (taskData.duration) {
      const workingDaysNeeded = Math.ceil(taskData.duration / 8)
      let daysAdded = 0
      let currentDate = new Date(startDate)
      
      while (daysAdded < workingDaysNeeded - 1) {
        currentDate.setDate(currentDate.getDate() + 1)
        // Skip weekends and holidays
        if (currentDate.getDay() !== 0 && currentDate.getDay() !== 6) {
          const isHoliday = props.holidays.some(holiday => 
            new Date(holiday.date).toDateString() === currentDate.toDateString()
          )
          if (!isHoliday) {
            daysAdded++
          }
        }
      }
      endDate.setTime(currentDate.getTime())
    }
    
    // Emit task move event
    emit('taskMove', {
      taskId: taskData.id,
      assigneeId,
      startDate: startDate.toISOString().split('T')[0],
      endDate: endDate.toISOString().split('T')[0]
    })
    
  } catch (error) {
    console.error('Error handling drop:', error)
  }
}

// Mouse events for hover states
const handleCellMouseEnter = (assigneeId, dateKey) => {
  hoveredCell.value = { assignee: assigneeId, date: dateKey }
}

const handleCellMouseLeave = () => {
  hoveredCell.value = { assignee: '', date: '' }
}

const handleCellClick = (assigneeId, date) => {
  emit('cellClick', { assigneeId, date })
}

// Resize handlers
const startResize = (event, task, direction) => {
  event.stopPropagation()
  isResizing.value = true
  resizingTask.value = task
  resizeDirection.value = direction
  resizeStartDate.value = new Date(task.startDate)
  resizeEndDate.value = new Date(task.endDate)
  
  // Add event listeners
  window.addEventListener('mousemove', handleResize)
  window.addEventListener('mouseup', endResize)
}

const handleResize = (event) => {
  if (!isResizing.value || !resizingTask.value) return
  
  // Get the cell element under the mouse
  const elementsUnderMouse = document.elementsFromPoint(event.clientX, event.clientY)
  const cellElement = elementsUnderMouse.find(el => el.classList.contains('date-cell'))
  
  if (!cellElement) return
  
  // Get the date from the cell's data
  const cellDate = dateColumns.value.find(col => 
    cellElement.getAttribute('data-date') === col.key
  )?.date
  
  if (!cellDate) return
  
  // Update the task dates based on resize direction
  if (resizeDirection.value === 'start') {
    if (cellDate <= resizeEndDate.value) {
      resizeStartDate.value = cellDate
    }
  } else {
    if (cellDate >= resizeStartDate.value) {
      resizeEndDate.value = cellDate
    }
  }
  
  // Update the task in the parent component
  emit('taskMove', {
    taskId: resizingTask.value.id,
    assigneeId: resizingTask.value.assignee,
    startDate: resizeStartDate.value.toISOString(),
    endDate: resizeEndDate.value.toISOString(),
    operation: 'resize'
  })
}

const endResize = () => {
  isResizing.value = false
  resizingTask.value = null
  resizeDirection.value = null
  resizeStartDate.value = null
  resizeEndDate.value = null
  
  // Remove event listeners
  window.removeEventListener('mousemove', handleResize)
  window.removeEventListener('mouseup', endResize)
}

// Watch for prop changes
watch(() => props.assignees, () => {
  // Recalculate when assignees change
}, { deep: true })

watch(() => props.tasks, () => {
  // Recalculate when tasks change
}, { deep: true })

watch(currentViewMode, () => {
  // Reset filters when view mode changes
  selectedProject.value = ''
  selectedPriority.value = ''
  selectedStatus.value = ''
})

onMounted(() => {
  // Initialize component
  console.log('Timeline Roster mounted with', props.assignees.length, 'assignees and', props.tasks.length, 'tasks')
})
</script>

<style scoped>
.timeline-roster-view {
  @apply w-full max-w-full;
}

.timeline-scroll-container {
  overflow-x: auto;
  overflow-y: auto;
}

.timeline-scroll-container::-webkit-scrollbar {
  @apply w-2 h-2;
}

.timeline-scroll-container::-webkit-scrollbar-track {
  @apply bg-gray-100 dark:bg-gray-700;
}

.timeline-scroll-container::-webkit-scrollbar-thumb {
  @apply bg-gray-300 dark:bg-gray-600 rounded;
}

.timeline-scroll-container::-webkit-scrollbar-thumb:hover {
  @apply bg-gray-400 dark:bg-gray-500;
}

.timeline-scroll-container::-webkit-scrollbar-corner {
  @apply bg-gray-100 dark:bg-gray-700;
}

.assignee-column {
  min-width: 280px;
  max-width: 280px;
  @apply sticky left-0 bg-white dark:bg-gray-800 border-r border-gray-200 dark:border-gray-600;
  z-index: 10;
}

.date-column {
  min-width: 120px;
  max-width: 120px;
}

.date-cell {
  width: 120px;
  min-width: 120px;
  max-width: 120px;
}

.drop-zone-active {
  @apply bg-blue-100 dark:bg-blue-900/30 border-2 border-dashed border-blue-400;
}

.task-block {
  min-height: 32px;
  transition: all 0.2s ease;
}

.task-block:hover {
  transform: translateY(-1px);
  z-index: 20;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.task-block:active {
  transform: scale(0.95);
}

.overallocated {
  @apply bg-red-50/50 dark:bg-red-900/20;
}

/* Enhanced responsive design */
@media (max-width: 1024px) {
  .assignee-column {
    min-width: 220px;
    max-width: 220px;
  }
  
  .date-column,
  .date-cell {
    min-width: 100px;
    max-width: 100px;
    width: 100px;
  }
}

@media (max-width: 768px) {
  .assignee-column {
    min-width: 180px;
    max-width: 180px;
  }
  
  .date-column,
  .date-cell {
    min-width: 80px;
    max-width: 80px;
    width: 80px;
  }
  
  .task-block {
    padding: 0.375rem;
    min-height: 28px;
  }
}

/* Print styles */
@media print {
  .timeline-roster-view {
    @apply text-black bg-white;
  }
  
  .drop-zone-active,
  .task-block:hover {
    @apply transform-none shadow-none;
  }
  
  .timeline-scroll-container {
    overflow: visible;
    max-height: none;
  }
}

/* Accessibility improvements */
.task-block:focus {
  @apply outline-none ring-2 ring-blue-500 ring-offset-2;
}

.task-block[draggable="true"] {
  cursor: grab;
}

.task-block[draggable="true"]:active {
  cursor: grabbing;
}

/* Animation for loading states */
@keyframes pulse-subtle {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.8;
  }
}

.animate-pulse-subtle {
  animation: pulse-subtle 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

/* Enhanced hover effects */
.date-cell:hover {
  transition: background-color 0.2s ease;
}

.timeline-header {
  backdrop-filter: blur(8px);
}

/* Custom scrollbar for webkit browsers */
* {
  scrollbar-width: thin;
  scrollbar-color: rgb(156 163 175) rgb(243 244 246);
}

/* Dark mode scrollbar */
.dark * {
  scrollbar-color: rgb(75 85 99) rgb(55 65 81);
}

.task-block.resizing {
  opacity: 0.8;
  cursor: ew-resize !important;
  background-color: rgba(255, 255, 255, 0.9);
  border-color: #2563eb;
  box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.4);
}

.resize-highlight {
  background-color: rgba(219, 234, 254, 0.3);
}

.dark .resize-highlight {
  background-color: rgba(30, 58, 138, 0.2);
}

.task-block .resize-handle {
  position: absolute;
  top: 0;
  bottom: 0;
  width: 8px;
  cursor: ew-resize;
  opacity: 0;
  transition: opacity 0.2s;
}

.task-block .resize-handle:hover,
.task-block:hover .resize-handle {
  opacity: 1;
  background-color: rgba(255, 255, 255, 0.2);
}

.task-block .resize-handle-left {
  left: 0;
}

.task-block .resize-handle-right {
  right: 0;
}
</style>