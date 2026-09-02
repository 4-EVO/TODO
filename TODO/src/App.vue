<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'

type Task = {
  id: number
  name: string
  dueDate: string
}

const STORAGE_KEY = 'todo-list.tasks.v1'

const taskName = ref('')
const dueDate = ref('')
const tasks = ref<Task[]>([])
const draggedTaskId = ref<number | null>(null)
const dragOverTaskId = ref<number | null>(null)
const formError = ref('')

const today = new Date()
const minimumDate = [
  today.getFullYear(),
  String(today.getMonth() + 1).padStart(2, '0'),
  String(today.getDate()).padStart(2, '0'),
].join('-')

const taskCountLabel = computed(() => {
  if (tasks.value.length === 0) return 'No tasks yet'
  if (tasks.value.length === 1) return '1 task planned'
  return `${tasks.value.length} tasks planned`
})

function isTask(value: unknown): value is Task {
  if (typeof value !== 'object' || value === null) return false

  const task = value as Partial<Task>
  return (
    typeof task.id === 'number' &&
    typeof task.name === 'string' &&
    typeof task.dueDate === 'string'
  )
}

onMounted(() => {
  try {
    const savedTasks = window.localStorage.getItem(STORAGE_KEY)
    if (!savedTasks) return

    const parsedTasks: unknown = JSON.parse(savedTasks)
    if (Array.isArray(parsedTasks)) {
      tasks.value = parsedTasks.filter(isTask)
    }
  } catch {
    // Keep the app usable if stored data is unavailable or malformed.
  }
})

watch(
  tasks,
  (updatedTasks) => {
    try {
      window.localStorage.setItem(STORAGE_KEY, JSON.stringify(updatedTasks))
    } catch {
      // Changes still work for the current page if storage is unavailable.
    }
  },
  { deep: true },
)

function addTask() {
  const cleanName = taskName.value.trim()

  if (!cleanName || !dueDate.value) {
    formError.value = 'Add both a task name and a due date.'
    return
  }

  tasks.value.push({
    id: Date.now(),
    name: cleanName,
    dueDate: dueDate.value,
  })

  taskName.value = ''
  dueDate.value = ''
  formError.value = ''
}

function deleteTask(taskId: number) {
  tasks.value = tasks.value.filter((task) => task.id !== taskId)
}

function formatDate(date: string) {
  return new Intl.DateTimeFormat(undefined, {
    month: 'short',
    day: 'numeric',
    year: 'numeric',
  }).format(new Date(`${date}T00:00:00`))
}

function startDrag(taskId: number) {
  draggedTaskId.value = taskId
}

function moveTask(targetTaskId: number) {
  if (draggedTaskId.value === null || draggedTaskId.value === targetTaskId) return

  const sourceIndex = tasks.value.findIndex((task) => task.id === draggedTaskId.value)
  const targetIndex = tasks.value.findIndex((task) => task.id === targetTaskId)

  if (sourceIndex === -1 || targetIndex === -1) return

  const reorderedTasks = [...tasks.value]
  const [movedTask] = reorderedTasks.splice(sourceIndex, 1)

  if (movedTask) {
    reorderedTasks.splice(targetIndex, 0, movedTask)
    tasks.value = reorderedTasks
  }
}

function finishDrag() {
  draggedTaskId.value = null
  dragOverTaskId.value = null
}
</script>

<template>
  <main class="app-shell">
    <section class="todo-card" aria-labelledby="page-title">
      <header class="app-header">
        <div>
          <h1 id="page-title">To do list</h1>
        </div>
        <div class="task-count" aria-live="polite">
          <span class="status-dot" aria-hidden="true"></span>
          {{ taskCountLabel }}
        </div>
      </header>

      <form class="task-form" @submit.prevent="addTask">
        <div class="form-field task-name-field">
          <label for="task-name">Task name</label>
          <input
            id="task-name"
            v-model="taskName"
            type="text"
            maxlength="80"
            placeholder="What needs to get done?"
            autocomplete="off"
            @input="formError = ''"
          />
        </div>

        <div class="form-field date-field">
          <label for="due-date">Due date</label>
          <input
            id="due-date"
            v-model="dueDate"
            type="date"
            :min="minimumDate"
            @input="formError = ''"
          />
        </div>

        <button class="add-button" type="submit">
          <span aria-hidden="true">+</span>
          Add task
        </button>
      </form>

      <p v-if="formError" class="form-error" role="alert">{{ formError }}</p>

      <section class="task-section" aria-labelledby="task-list-title">
        <div class="list-heading">
          <div>
            <p class="section-label">Your schedule</p>
            <h2 id="task-list-title">Upcoming tasks</h2>
          </div>
          <p v-if="tasks.length > 1" class="drag-hint">Drag to reorder</p>
        </div>

        <div v-if="tasks.length === 0" class="empty-state">
          <div class="empty-icon" aria-hidden="true">✓</div>
          <h3>Your day is wide open</h3>
          <p>Add a task and due date above to start planning.</p>
        </div>

        <TransitionGroup v-else name="task-list" tag="ul" class="task-list">
          <li
            v-for="task in tasks"
            :key="task.id"
            class="task-item"
            :class="{
              'is-dragging': draggedTaskId === task.id,
              'is-drag-over': dragOverTaskId === task.id,
            }"
            draggable="true"
            @dragstart="startDrag(task.id)"
            @dragenter.prevent="dragOverTaskId = task.id"
            @dragover.prevent
            @drop.prevent="moveTask(task.id); finishDrag()"
            @dragend="finishDrag"
          >
            <div class="drag-handle" title="Drag to reorder" aria-hidden="true">
              <span></span><span></span><span></span>
              <span></span><span></span><span></span>
            </div>

            <div class="task-content">
              <p class="task-name">{{ task.name }}</p>
              <time class="task-date" :datetime="task.dueDate">
                <span aria-hidden="true">◷</span>
                Due {{ formatDate(task.dueDate) }}
              </time>
            </div>

            <button
              class="remove-button"
              type="button"
              :aria-label="`Remove ${task.name}`"
              @click="deleteTask(task.id)"
            >
              <span aria-hidden="true">×</span>
              <span class="remove-label">Remove</span>
            </button>
          </li>
        </TransitionGroup>
      </section>
    </section>
  </main>
</template>
