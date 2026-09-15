<template>
  <div class="app" :class="{ 'sidebar-collapsed': sidebarCollapsed }">
    <aside class="sidebar">
      <div class="sidebar-logo">
        <div class="sidebar-logo-text">
          <h1>{{ t('nav.companyName') }}</h1>
          <span class="subtitle">{{ t('nav.subtitle') }}</span>
        </div>
        <button
          class="sidebar-toggle"
          type="button"
          @click="sidebarCollapsed = !sidebarCollapsed"
          :title="sidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'"
          :aria-label="sidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'"
        >
          <span class="toggle-icon" :class="{ flipped: sidebarCollapsed }" v-html="ICONS.toggle"></span>
        </button>
      </div>
      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.path"
          :to="item.path"
          :class="{ active: $route.path === item.path }"
          :title="item.label"
        >
          <span class="nav-icon" v-html="item.icon"></span>
          <span class="nav-label">{{ item.label }}</span>
        </router-link>
      </nav>
    </aside>

    <div class="content-area">
      <header class="top-bar">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </header>
      <FilterBar />
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

const ICON_PROPS = 'viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"'

const ICONS = {
  overview: `<svg ${ICON_PROPS}><rect x="3" y="3" width="7" height="7" rx="1.5"/><rect x="14" y="3" width="7" height="7" rx="1.5"/><rect x="3" y="14" width="7" height="7" rx="1.5"/><rect x="14" y="14" width="7" height="7" rx="1.5"/></svg>`,
  inventory: `<svg ${ICON_PROPS}><path d="M21 7.5l-9-4.5-9 4.5 9 4.5 9-4.5z"/><path d="M3 7.5v9l9 4.5 9-4.5v-9"/><path d="M12 12v9"/></svg>`,
  orders: `<svg ${ICON_PROPS}><rect x="5" y="4" width="14" height="17" rx="2"/><path d="M9 4V2.5h6V4"/><path d="M8 10h8M8 14h8M8 18h5"/></svg>`,
  finance: `<svg ${ICON_PROPS}><circle cx="12" cy="12" r="9"/><path d="M12 7v10M9.5 9.5c0-1.1 1.1-2 2.5-2s2.5.7 2.5 1.8c0 2.4-5 1.1-5 3.5 0 1.1 1.1 1.9 2.5 1.9s2.5-.9 2.5-2"/></svg>`,
  demand: `<svg ${ICON_PROPS}><path d="M3 17l6-6 4 4 8-8"/><path d="M15 6h6v6"/></svg>`,
  reports: `<svg ${ICON_PROPS}><path d="M4 20V10M11 20V4M18 20v-7"/></svg>`,
  toggle: `<svg ${ICON_PROPS}><path d="M15 18l-6-6 6-6"/></svg>`
}

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])
    const sidebarCollapsed = ref(false)

    const navItems = computed(() => [
      { path: '/', label: t('nav.overview'), icon: ICONS.overview },
      { path: '/inventory', label: t('nav.inventory'), icon: ICONS.inventory },
      { path: '/orders', label: t('nav.orders'), icon: ICONS.orders },
      { path: '/spending', label: t('nav.finance'), icon: ICONS.finance },
      { path: '/demand', label: t('nav.demandForecast'), icon: ICONS.demand },
      { path: '/reports', label: 'Reports', icon: ICONS.reports }
    ])

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      navItems,
      showProfileDetails,
      showTasks,
      sidebarCollapsed,
      ICONS,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: #f8fafc;
  color: #1e293b;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.app {
  display: flex;
  min-height: 100vh;
}

.sidebar {
  position: fixed;
  top: 0;
  left: 0;
  width: 240px;
  height: 100vh;
  background: #0f172a;
  display: flex;
  flex-direction: column;
  z-index: 100;
  overflow-y: auto;
  transition: width 0.2s ease;
}

.sidebar-logo {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  padding: 1.5rem 1.25rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

.sidebar-logo-text {
  overflow: hidden;
}

.sidebar-toggle {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border: none;
  border-radius: 6px;
  background: rgba(255, 255, 255, 0.06);
  color: #94a3b8;
  cursor: pointer;
  transition: all 0.2s ease;
}

.sidebar-toggle:hover {
  background: rgba(255, 255, 255, 0.12);
  color: #f1f5f9;
}

.toggle-icon {
  display: flex;
  width: 16px;
  height: 16px;
  transition: transform 0.2s ease;
}

.toggle-icon svg {
  width: 100%;
  height: 100%;
}

.toggle-icon.flipped {
  transform: rotate(180deg);
}

.sidebar-logo h1 {
  font-size: 1.125rem;
  font-weight: 700;
  color: #f8fafc;
  letter-spacing: -0.025em;
  margin: 0 0 0.25rem;
}

.sidebar-logo .subtitle {
  font-size: 0.75rem;
  color: #64748b;
  font-weight: 400;
}

.sidebar-nav {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  padding: 1rem 0.75rem;
}

.sidebar-nav a {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.625rem 0.875rem;
  color: #94a3b8;
  text-decoration: none;
  font-weight: 500;
  font-size: 0.875rem;
  border-radius: 8px;
  transition: all 0.2s ease;
}

.sidebar-nav a:hover {
  color: #f1f5f9;
  background: rgba(255, 255, 255, 0.06);
}

.sidebar-nav a.active {
  color: #ffffff;
  background: #2563eb;
}

.nav-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  flex-shrink: 0;
}

.nav-icon svg {
  width: 100%;
  height: 100%;
}

.nav-label {
  white-space: nowrap;
  transition: opacity 0.15s ease;
}

.content-area {
  flex: 1;
  margin-left: 240px;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  transition: margin-left 0.2s ease;
}

/* Collapsible sidebar: manual toggle via .sidebar-collapsed on .app */
.app.sidebar-collapsed .sidebar {
  width: 72px;
}

.app.sidebar-collapsed .content-area {
  margin-left: 72px;
}

.app.sidebar-collapsed .sidebar-logo {
  justify-content: center;
}

.app.sidebar-collapsed .sidebar-logo-text,
.app.sidebar-collapsed .nav-label {
  display: none;
}

.app.sidebar-collapsed .sidebar-nav a {
  justify-content: center;
}

/* Icons-only mode on smaller screens, independent of the manual toggle above */
@media (max-width: 1024px) {
  .sidebar {
    width: 72px;
  }

  .content-area {
    margin-left: 72px;
  }

  .sidebar-logo-text,
  .sidebar-toggle,
  .nav-label {
    display: none;
  }

  .sidebar-logo {
    justify-content: center;
  }

  .sidebar-nav a {
    justify-content: center;
  }
}

.top-bar {
  background: #ffffff;
  border-bottom: 1px solid #e2e8f0;
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.05);
  position: sticky;
  top: 0;
  z-index: 90;
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 1rem;
  padding: 0 2rem;
}

.main-content {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
}

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.06);
}

.stat-label {
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value {
  color: #ea580c;
}

.stat-card.success .stat-value {
  color: #059669;
}

.stat-card.danger .stat-value {
  color: #dc2626;
}

.stat-card.info .stat-value {
  color: #2563eb;
}

.card {
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.badge.decreasing {
  background: #fecaca;
  color: #991b1b;
}

.badge.stable {
  background: #e0e7ff;
  color: #3730a3;
}

.badge.high {
  background: #fecaca;
  color: #991b1b;
}

.badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
