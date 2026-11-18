
<script setup>
import { ref, computed, nextTick } from 'vue'
import { storeToRefs } from 'pinia'
import AppTabbar from '../components/AppTabbar.vue'
import TaskCard from '../components/TaskCard.vue'
import TaskForm from '../components/TaskForm.vue'
import TaskListControls from '../components/TaskListControls.vue'
import { useTasksStore } from '../store/tasks'
import { useProjectsStore } from '../store/projects'
import '../css/task-components.css'
import '../css/task-list.css'

// Отримуємо спільні дані з Pinia
const taskStore = useTasksStore()
const projectStore = useProjectsStore()

const {
  tasks,
  visibleTasks,
  searchText,
  selectedStatus,
  selectedPriority,
  selectedProject,
  sortOrder,
} = storeToRefs(taskStore)

const { projects } = storeToRefs(projectStore)

// Ці змінні потрібні тільки сторінці завдань
const showForm = ref(false)
const editingId = ref(null)
const formAnchor = ref(null)
const feedback = ref('')
const newProjectTitle = ref('')
const newProjectError = ref('')

// Знаходимо завдання, яке користувач редагує
const editingTask = computed(() =>
  tasks.value.find((task) => task.id === editingId.value) ?? null,
)

// Підраховуємо завдання для кожного проєкту
const projectStats = computed(() =>
  projects.value.map((project) => ({
    ...project,
    taskCount: tasks.value.filter(
      (task) => task.projectId === project.id,
    ).length,
  })),
)

// Прокручуємо сторінку до відкритої форми
async function scrollToForm() {
  await nextTick()
  formAnchor.value?.scrollIntoView({
    behavior: 'smooth',
    block: 'start',
  })
}

function startCreate() {
  editingId.value = null
  showForm.value = true
  feedback.value = ''
  scrollToForm()
}

function startEdit(id) {
  editingId.value = Number(id)
  showForm.value = true
  feedback.value = ''
  scrollToForm()
}

function cancelForm() {
  editingId.value = null
  showForm.value = false
}

// Створення та редагування виконують actions зі сховища
function saveTask(values) {
  const isNew = values.id === null

  const saved = isNew
    ? taskStore.addTask(values)
    : taskStore.updateTask(values)

  if (!saved) {
    feedback.value = 'Не вдалося зберегти завдання.'
    return
  }

  cancelForm()
  taskStore.resetFilters()

  feedback.value = isNew
    ? 'Завдання успішно створено.'
    : 'Завдання успішно оновлено.'
}

// Видаляємо завдання тільки після підтвердження
function handleDelete(id) {
  const task = tasks.value.find(
    (item) => item.id === Number(id),
  )

  if (!task) return

  const confirmed = window.confirm(
    `Видалити завдання «${task.title}»?`,
  )

  if (!confirmed) return

  if (taskStore.deleteTask(id)) {
    if (editingId.value === Number(id)) {
      cancelForm()
    }

    feedback.value = `Завдання «${task.title}» видалено.`
  }
}

// Створюємо проєкт через окреме Pinia-сховище
function createProject() {
  const project = projectStore.addProject(
    newProjectTitle.value,
  )

  if (!project) {
    newProjectError.value =
      'Назва: 3–40 символів. Не повторюйте назви проєктів.'
    return
  }

  newProjectTitle.value = ''
  newProjectError.value = ''

  // Новостворений проєкт одразу стає активним фільтром
  selectedProject.value = project.id
  feedback.value = `Проєкт «${project.title}» створено.`
}
</script>

<template>
  <f7-page name="tasks" class="tf3-page">
    <f7-navbar title="Мої завдання" />

    <div class="tf3-container">
      <section class="tf3-hero">
        <div class="tf3-eyebrow">
          TASKFLOW / TASK MANAGER
        </div>

        <h1>Керування завданнями і проєктами</h1>

        <p>
          Pinia забезпечує спільний стан між вкладками
          без перезавантаження.
        </p>

        <button
          type="button"
          class="tf5-btn tf5-btn--primary tf5-new"
          @click="startCreate"
        >
          + Додати завдання
        </button>
      </section>

      <!-- Каталог проєктів -->
      <section
        class="tf5-panel tf6-project-panel"
        aria-label="Проєкти"
      >
        <h2>Мої проєкти</h2>

        <div class="tf6-project-list">
          <span
            v-for="project in projectStats"
            :key="project.id"
            class="tf6-project-pill"
          >
            {{ project.title }}
            <strong>{{ project.taskCount }}</strong>
          </span>
        </div>

        <form
          class="tf6-project-form"
          @submit.prevent="createProject"
        >
          <label>
            Назва нового проєкту
            <input
              v-model="newProjectTitle"
              type="text"
              maxlength="40"
              placeholder="Наприклад: Лабораторні"
              @input="newProjectError = ''"
            />
          </label>

          <button
            type="submit"
            class="tf5-btn tf5-btn--primary"
          >
            Створити проєкт
          </button>
        </form>

        <p
          v-if="newProjectError"
          class="tf5-error"
          role="alert"
        >
          {{ newProjectError }}
        </p>
      </section>

      <!-- Форма створення та редагування -->
      <div ref="formAnchor">
        <TaskForm
          v-if="showForm"
          :key="editingId ?? 'new'"
          :task="editingTask"
          :projects="projects"
          @save="saveTask"
          @cancel="cancelForm"
        />
      </div>

      <p v-if="feedback" class="tf5-feedback" role="status">
        {{ feedback }}
      </p>

      <!-- Пошук, фільтри та сортування -->
      <TaskListControls
        v-model:search="searchText"
        v-model:status="selectedStatus"
        v-model:priority="selectedPriority"
        v-model:project="selectedProject"
        v-model:sort="sortOrder"
        :projects="projects"
        @reset="taskStore.resetFilters"
      />

      <!-- Список завдань -->
      <section class="tf3-section">
        <div class="tf3-section-heading">
          <div>
            <div class="tf3-eyebrow">TASK LIST</div>
            <h2>Список завдань</h2>
          </div>

          <div class="tf3-counter">
            Показано: {{ visibleTasks.length }} із {{ tasks.length }}
          </div>
        </div>

        <div
          v-if="visibleTasks.length"
          class="tf3-task-grid"
        >
          <TaskCard
            v-for="task in visibleTasks"
            :key="task.id"
            :task="task"
            :project-title="
              projectStore.projectTitle(task.projectId)
            "
            @status-change="taskStore.setTaskStatus"
            @priority-change="taskStore.setTaskPriority"
            @edit="startEdit"
            @delete="handleDelete"
          />
        </div>

        <div v-else class="tf5-empty">
          <p>
            За поточними умовами завдань не знайдено.
          </p>

          <button
            type="button"
            class="tf5-btn"
            @click="taskStore.resetFilters"
          >
            Показати всі завдання
          </button>
        </div>
      </section>
    </div>

    <AppTabbar active="tasks" />
  </f7-page>
</template>
