<template>
  <div class="app">
    <header class="topbar">
      <div class="brand">
        <span class="brand-mark">A</span>
        <span>方舟服务台</span>
      </div>
      <nav class="nav">
        <button :class="{ active: page === 'list' }" @click="goList">
          工单中心
        </button>
        <button :class="{ active: page === 'create' }" @click="goCreate">
          创建工单
        </button>
      </nav>
      <div class="current-user">
        <span class="avatar">林</span>
        <span>林安 · 应用支持组</span>
      </div>
    </header>

    <main class="workspace">
      <!-- 工单列表 -->
      <section v-if="page === 'list'">
        <div class="page-heading">
          <div>
            <p class="eyebrow">工单中心</p>
            <h1>服务请求</h1>
            <p>查看和跟踪当前支持队列中的工单。</p>
          </div>
          <button class="primary" @click="goCreate">创建工单</button>
        </div>

        <div class="toolbar">
          <input
            v-model="filters.keyword"
            class="search"
            type="search"
            placeholder="搜索工单编号或标题"
        />
          <select v-model="filters.status">
            <option
              v-for="item in statusOptions"
              :key="item.value"
              :value="item.value"
            >
              {{ item.label }}
            </option>
          </select>
          <span class="result-count">共 {{ list.total }} 条</span>
        </div>

        <div v-if="list.error" class="alert error">
          <span>{{ list.error }}</span>
          <button class="text-button" @click="loadList">重新加载</button>
        </div>

        <div v-else-if="list.loading" class="panel loading-panel">
          正在加载工单...
        </div>

        <div v-else-if="!list.items.length" class="panel empty-state">
          <h2>没有符合条件的工单</h2>
          <p>调整搜索条件，或创建一张新工单。</p>
          <button class="primary" @click="goCreate">创建工单</button>
        </div>

        <div v-else class="ticket-table">
          <div class="ticket-row table-header">
            <span>工单编号</span>
            <span>标题</span>
            <span>状态</span>
            <span>优先级</span>
            <span>处理人</span>
            <span>更新时间</span>
          </div>
          <button
            v-for="ticket in list.items"
            :key="ticket.id"
            class="ticket-row ticket-item"
            @click="openDetail(ticket.id)"
          >
            <span class="ticket-number">{{ ticket.ticketNo }}</span>
            <span class="ticket-title">
              <strong>{{ ticket.title }}</strong>
              <small>{{ ticket.category }}</small>
            </span>
            <span>
              <span class="badge" :class="`status-${ticket.status}`">
                {{ statusMap[ticket.status] }}
              </span>
            </span>
            <span>
              <span class="badge" :class="`priority-${ticket.priority}`">
                {{ priorityMap[ticket.priority] }}
              </span>
            </span>
            <span class="ticket-assignee">{{ ticket.assignee || '待分配' }}</span>
            <span class="ticket-updated">{{ formatDate(ticket.updatedAt) }}</span>
          </button>
        </div>
      </section>

      <!-- 创建工单 -->
      <section v-else-if="page === 'create'">
        <button class="back-button" @click="goList">返回工单列表</button>
        <div class="page-heading compact">
          <div>
            <p class="eyebrow">新建请求</p>
            <h1>创建工单</h1>
            <p>填写问题信息，提交后将进入待处理队列。</p>
          </div>
        </div>

        <div class="create-layout">
          <form class="panel form-panel" novalidate @submit.prevent="submitTicket">
            <div v-if="submitError" class="alert error">{{ submitError }}</div>

            <div class="field" :class="{ 'has-error': errors.title }">
              <label for="title">标题 <span>必填</span></label>
              <input
                id="title"
                v-model="form.title"
                maxlength="80"
                placeholder="简要描述遇到的问题"
                @input="errors.title = ''"
              />
              <div class="field-meta">
                <span class="field-error">{{ errors.title }}</span>
                <span>{{ form.title.length }}/80</span>
              </div>
            </div>

            <div class="form-grid">
              <div class="field">
                <label for="category">问题分类</label>
                <select id="category" v-model="form.category">
                  <option v-for="item in categories" :key="item" :value="item">
                    {{ item }}
                  </option>
                </select>
              </div>
              <div class="field">
                <label for="priority">优先级</label>
                <select id="priority" v-model="form.priority">
                  <option
                    v-for="item in priorities"
                    :key="item.value"
                    :value="item.value"
                  >
                    {{ item.label }}
                  </option>
                </select>
              </div>
            </div>

            <div class="field" :class="{ 'has-error': errors.description }">
              <label for="description">问题描述 <span>必填</span></label>
              <textarea
                id="description"
                v-model="form.description"
                maxlength="2000"
                rows="9"
                placeholder="请描述问题现象、发生时间、影响范围和已尝试的操作"
                @input="errors.description = ''"
              ></textarea>
              <div class="field-meta">
                <span class="field-error">{{ errors.description }}</span>
                <span>{{ form.description.length }}/2000</span>
              </div>
            </div>

            <div class="form-actions">
              <button type="button" class="secondary" @click="goList">
                取消
              </button>
              <button class="primary" type="submit" :disabled="saving">
                {{ saving ? '正在提交...' : '提交工单' }}
              </button>
            </div>
          </form>

          <aside class="panel summary-panel">
            <p class="eyebrow">登记摘要</p>
            <h2>{{ form.title || '尚未填写标题' }}</h2>
            <dl>
              <div>
                <dt>分类</dt>
                <dd>{{ form.category }}</dd>
              </div>
              <div>
                <dt>优先级</dt>
                <dd>{{ priorityMap[form.priority] }}</dd>
              </div>
              <div>
                <dt>当前处理人</dt>
                <dd>待分配</dd>
              </div>
              <div>
                <dt>提交后状态</dt>
                <dd><span class="badge status-pending">待处理</span></dd>
              </div>
            </dl>
          </aside>
        </div>
      </section>

      <!-- 工单详情 -->
      <section v-else>
        <button class="back-button" @click="goList">返回工单列表</button>

        <div v-if="detail.loading" class="panel loading-panel">
          正在加载工单详情...
        </div>
        <div v-else-if="detail.error" class="panel empty-state">
          <h2>无法加载工单</h2>
          <p>{{ detail.error }}</p>
          <button class="primary" @click="loadDetail">重新加载</button>
        </div>
        <template v-else-if="detail.ticket">
          <div v-if="createdHint" class="alert success">
            工单已创建并保存，当前状态为“待处理”。
          </div>

          <div class="detail-layout">
            <article class="panel detail-main">
              <div class="detail-title">
                <div>
                  <p class="eyebrow">{{ detail.ticket.ticketNo }}</p>
                  <h1>{{ detail.ticket.title }}</h1>
                </div>
                <span class="badge" :class="`status-${detail.ticket.status}`">
                  {{ statusMap[detail.ticket.status] }}
                </span>
              </div>

              <div class="description">
                <h2>问题描述</h2>
                <p>{{ detail.ticket.description }}</p>
              </div>

              <div class="activity">
                <h2>操作记录</h2>
                <ol v-if="detail.activities.length">
                  <li
                    v-for="record in detail.activities"
                    :key="record.id"
                  >
                    <span class="timeline-dot"></span>
                    <div>
                      <strong>{{ record.action }}</strong>
                      <p>{{ record.detail }}</p>
                      <small>
                        {{ record.operatorName }} ·
                        {{ formatDateTime(record.operatedAt) }}
                      </small>
                    </div>
                  </li>
                </ol>
                <p v-else class="muted">暂无操作记录</p>
              </div>
            </article>

            <aside class="panel detail-side">
              <h2>工单信息</h2>
              <dl>
                <div>
                  <dt>工单编号</dt>
                  <dd>{{ detail.ticket.ticketNo }}</dd>
                </div>
                <div>
                  <dt>分类</dt>
                  <dd>{{ detail.ticket.category }}</dd>
                </div>
                <div>
                  <dt>优先级</dt>
                  <dd>
                    <span
                      class="badge"
                      :class="`priority-${detail.ticket.priority}`"
                    >
                      {{ priorityMap[detail.ticket.priority] }}
                    </span>
                  </dd>
                </div>
                <div>
                  <dt>创建人</dt>
                  <dd>{{ detail.ticket.reporter || '未知' }}</dd>
                </div>
                <div>
                  <dt>处理人</dt>
                  <dd>{{ detail.ticket.assignee || '待分配' }}</dd>
                </div>
                <div>
                  <dt>创建时间</dt>
                  <dd>{{ formatDateTime(detail.ticket.createdAt) }}</dd>
                </div>
                <div>
                  <dt>更新时间</dt>
                  <dd>{{ formatDateTime(detail.ticket.updatedAt) }}</dd>
                </div>
              </dl>
            </aside>
          </div>
        </template>
      </section>
    </main>

    <div v-if="toast" class="toast" role="status">{{ toast }}</div>
  </div>
</template>

<script setup>
import {
  computed,
  nextTick,
  onBeforeUnmount,
  onMounted,
  reactive,
  ref,
  watch,
} from 'vue'

const USE_MOCK =
  String(import.meta.env.VITE_USE_MOCK ?? 'true').toLowerCase() !== 'false'
const API_BASE = (import.meta.env.VITE_API_BASE_URL || '/api').replace(/\/$/, '')
const CURRENT_USER = '林安'
const STORAGE_KEY = 'ark-ticket-console-v1'

const page = ref('list')
const detailId = ref('')
const createdHint = ref(false)
const toast = ref('')
let toastTimer

const statusMap = {
  pending: '待处理',
  processing: '处理中',
  escalated: '已升级',
  resolved: '已解决',
  closed: '已关闭',
}

const priorityMap = {
  low: '低',
  normal: '普通',
  high: '高',
  urgent: '紧急',
}

const statusOptions = [
  { value: 'all', label: '全部状态' },
  { value: 'pending', label: '待处理' },
  { value: 'processing', label: '处理中' },
  { value: 'escalated', label: '已升级' },
  { value: 'resolved', label: '已解决' },
]

const categories = ['系统故障', '账号与权限', '数据异常', '产品咨询', '其他']
const priorities = [
  { value: 'low', label: '低' },
  { value: 'normal', label: '普通' },
  { value: 'high', label: '高' },
  { value: 'urgent', label: '紧急' },
]

const filters = reactive({
  keyword: '',
  status: 'all',
})

const list = reactive({
  items: [],
  total: 0,
  loading: true,
  error: '',
})

const detail = reactive({
  ticket: null,
  activities: [],
  loading: false,
  error: '',
})

const form = reactive({
  title: '',
  category: categories[0],
  priority: 'normal',
  description: '',
})

const errors = reactive({
  title: '',
  description: '',
})

const saving = ref(false)
const submitError = ref('')

const seedTickets = [
  {
    id: 'ticket-1001',
    ticketNo: 'GD-20261103-001',
    title: '财务共享中心无法上传报销附件',
    description:
      '从今天上午开始，报销单上传 PDF 附件时提示“文件上传失败”。更换浏览器后仍然出现，当前影响财务月结相关工作。',
    category: '系统故障',
    priority: 'high',
    status: 'processing',
    reporter: '陈薇',
    assignee: '周宁',
    createdAt: '2026-11-03T08:42:00+08:00',
    updatedAt: '2026-11-03T09:25:00+08:00',
    activities: [
      {
        id: 'record-1',
        operatorName: '陈薇',
        action: '创建工单',
        detail: '提交问题描述并进入待处理队列',
        operatedAt: '2026-11-03T08:42:00+08:00',
      },
      {
        id: 'record-2',
        operatorName: '周宁',
        action: '受理工单',
        detail: '工单已分配给应用支持组处理',
        operatedAt: '2026-11-03T09:25:00+08:00',
      },
    ],
  },
  {
    id: 'ticket-1002',
    ticketNo: 'GD-20261102-014',
    title: '新员工无法登录项目管理平台',
    description:
      '新入职员工完成账号开通后，首次登录提示账号不存在。需要核对账号同步状态和权限配置。',
    category: '账号与权限',
    priority: 'normal',
    status: 'pending',
    reporter: '许晨',
    assignee: null,
    createdAt: '2026-11-02T16:10:00+08:00',
    updatedAt: '2026-11-02T16:10:00+08:00',
    activities: [
      {
        id: 'record-3',
        operatorName: '许晨',
        action: '创建工单',
        detail: '提交问题描述并进入待处理队列',
        operatedAt: '2026-11-02T16:10:00+08:00',
      },
    ],
  },
  {
    id: 'ticket-1003',
    ticketNo: 'GD-20261101-008',
    title: '客户导入任务持续显示处理中',
    description:
      '批量导入客户资料后，任务状态超过两小时未更新。导入文件约 8000 条记录，需要检查任务执行日志。',
    category: '数据异常',
    priority: 'urgent',
    status: 'escalated',
    reporter: '罗嘉',
    assignee: '许晨',
    createdAt: '2026-11-01T11:32:00+08:00',
    updatedAt: '2026-11-01T14:08:00+08:00',
    activities: [
      {
        id: 'record-4',
        operatorName: '罗嘉',
        action: '创建工单',
        detail: '提交批量导入异常信息',
        operatedAt: '2026-11-01T11:32:00+08:00',
      },
      {
        id: 'record-5',
        operatorName: '许晨',
        action: '升级工单',
        detail: '已升级至数据平台组，等待排查任务队列',
        operatedAt: '2026-11-01T14:08:00+08:00',
      },
    ],
  },
]

function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms))
}

function readMockTickets() {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) return JSON.parse(saved)

  const initial = JSON.parse(JSON.stringify(seedTickets))
  localStorage.setItem(STORAGE_KEY, JSON.stringify(initial))
  return initial
}

function writeMockTickets(tickets) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(tickets))
}

const mockApi = {
  async list(query) {
    await sleep(240)

    const keyword = (query.keyword || '').trim().toLowerCase()
    const items = readMockTickets()
      .filter((ticket) => {
        const matchesKeyword =
          !keyword ||
          ticket.title.toLowerCase().includes(keyword) ||
          ticket.ticketNo.toLowerCase().includes(keyword)
        const matchesStatus =
          !query.status ||
          query.status === 'all' ||
          ticket.status === query.status

        return matchesKeyword && matchesStatus
      })
      .sort((a, b) => new Date(b.updatedAt) - new Date(a.updatedAt))

    return { items, total: items.length }
  },

  async create(payload) {
    await sleep(360)

    const tickets = readMockTickets()
    const now = new Date()
    const datePart = now.toISOString().slice(0, 10).replaceAll('-', '')
    const sequence = String(tickets.length + 1).padStart(3, '0')
    const timestamp = now.toISOString()

    const ticket = {
      id: `ticket-${now.getTime()}`,
      ticketNo: `GD-${datePart}-${sequence}`,
      ...payload,
      status: 'pending',
      reporter: CURRENT_USER,
      assignee: null,
      createdAt: timestamp,
      updatedAt: timestamp,
      activities: [
        {
          id: `record-${now.getTime()}`,
          operatorName: CURRENT_USER,
          action: '创建工单',
          detail: '创建工单并进入待处理队列',
          operatedAt: timestamp,
        },
      ],
    }

    writeMockTickets([ticket, ...tickets])
    return ticket
  },

  async get(id) {
    await sleep(220)
    const ticket = readMockTickets().find((item) => item.id === id)
    if (!ticket) throw new Error('工单不存在或已被删除')

    return {
      ticket,
      activities: ticket.activities || [],
    }
  },
}

async function httpRequest(path, options = {}) {
  const controller = new AbortController()
  const timeout = setTimeout(() => controller.abort(), 12000)

  try {
    const response = await fetch(`${API_BASE}${path}`, {
      ...options,
      signal: controller.signal,
      headers: {
        Accept: 'application/json',
        ...(options.body ? { 'Content-Type': 'application/json' } : {}),
        ...options.headers,
      },
    })

    const payload = await response.json().catch(() => null)
    if (!response.ok) {
      throw new Error(
        payload?.message ||
          payload?.error?.message ||
          `请求失败（HTTP ${response.status}）`,
      )
    }

    return payload?.data ?? payload
  } catch (error) {
    if (error.name === 'AbortError') throw new Error('请求超时，请稍后重试')
    throw error
  } finally {
    clearTimeout(timeout)
  }
}

const ticketApi = {
  async list(query) {
    if (USE_MOCK) return mockApi.list(query)

    const params = new URLSearchParams({
      keyword: query.keyword || '',
      status: query.status || 'all',
    })
    const data = await httpRequest(`/tickets?${params}`)
    const items = data.items || data.records || []

    return { items, total: data.total ?? items.length }
  },

  async create(payload) {
    if (USE_MOCK) return mockApi.create(payload)

    const data = await httpRequest('/tickets', {
      method: 'POST',
      body: JSON.stringify(payload),
    })
    return data.ticket || data
  },

  async get(id) {
    if (USE_MOCK) return mockApi.get(id)

    const data = await httpRequest(`/tickets/${encodeURIComponent(id)}`)
    const ticket = data.ticket || data
    const activities =
      data.activities || data.operationRecords || ticket.activities || []

    return { ticket, activities }
  },
}

let listRequestId = 0

async function loadList() {
  const requestId = ++listRequestId
  list.loading = true
  list.error = ''

  try {
    const result = await ticketApi.list(filters)
    if (requestId !== listRequestId) return
    list.items = result.items
    list.total = result.total
  } catch (error) {
    if (requestId !== listRequestId) return
    list.error = error.message || '工单列表加载失败'
  } finally {
    if (requestId === listRequestId) list.loading = false
  }
}

let detailRequestId = 0

async function loadDetail() {
  if (!detailId.value) return

  const requestId = ++detailRequestId
  detail.loading = true
  detail.error = ''
  detail.ticket = null
  detail.activities = []

  try {
    const result = await ticketApi.get(detailId.value)
    if (requestId !== detailRequestId) return
    detail.ticket = result.ticket
    detail.activities = result.activities
  } catch (error) {
    if (requestId !== detailRequestId) return
    detail.error = error.message || '工单详情加载失败'
  } finally {
    if (requestId === detailRequestId) detail.loading = false
  }
}

function validateForm() {
  form.title = form.title.trim()
  form.description = form.description.trim()

  if (form.title.length < 4) {
    errors.title = '标题至少需要 4 个字符'
  } else if (form.title.length > 80) {
    errors.title = '标题不能超过 80 个字符'
  }

  if (form.description.length < 10) {
    errors.description = '请至少填写 10 个字符，以便准确处理问题'
  } else if (form.description.length > 2000) {
    errors.description = '问题描述不能超过 2000 个字符'
  }

  if (!categories.includes(form.category)) {
    submitError.value = '请选择有效的问题分类'
    return false
  }

  if (!priorityMap[form.priority]) {
    submitError.value = '请选择有效的优先级'
    return false
  }

  const firstInvalid = Object.keys(errors).find((key) => errors[key])
  if (firstInvalid) {
    nextTick(() => document.getElementById(firstInvalid)?.focus())
    return false
  }

  return true
}

async function submitTicket() {
  submitError.value = ''
  errors.title = ''
  errors.description = ''

  if (!validateForm()) return

  saving.value = true

  try {
    const ticket = await ticketApi.create({
      title: form.title,
      category: form.category,
      priority: form.priority,
      description: form.description,
    })

    form.title = ''
    form.description = ''
    form.category = categories[0]
    form.priority = 'normal'

    createdHint.value = true
    showToast('工单创建成功')
    openDetail(ticket.id)
  } catch (error) {
    submitError.value = error.message || '工单提交失败，请稍后重试'
  } finally {
    saving.value = false
  }
}

function showToast(message) {
  clearTimeout(toastTimer)
  toast.value = message
  toastTimer = setTimeout(() => {
    toast.value = ''
  }, 2400)
}

function setHash(hash) {
  if (window.location.hash === hash) {
    syncRoute()
  } else {
    window.location.hash = hash
  }
}

function goList() {
  createdHint.value = false
  setHash('#/tickets')
}

function goCreate() {
  createdHint.value = false
  setHash('#/create')
}

function openDetail(id) {
  detailId.value = id
  setHash(`#/tickets/${encodeURIComponent(id)}`)
}

function syncRoute() {
  const rawPath = window.location.hash.replace(/^#\/?/, '').split('?')[0]
  const parts = rawPath.split('/').filter(Boolean)

  if (parts[0] === 'create') {
    page.value = 'create'
    return
  }

  if (parts[0] === 'tickets' && parts[1]) {
    page.value = 'detail'
    detailId.value = decodeURIComponent(parts[1])
    loadDetail()
    return
  }

  page.value = 'list'
  loadList()
}

function formatDate(value) {
  if (!value) return '-'
  return new Intl.DateTimeFormat('zh-CN', {
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  }).format(new Date(value))
}

function formatDateTime(value) {
  if (!value) return '-'
  return new Intl.DateTimeFormat('zh-CN', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  }).format(new Date(value))
}

watch(
  [() => filters.keyword, () => filters.status],
  () => {
    if (page.value === 'list') loadList()
  },
)

onMounted(() => {
  window.addEventListener('hashchange', syncRoute)
  syncRoute()
})

onBeforeUnmount(() => {
  window.removeEventListener('hashchange', syncRoute)
  clearTimeout(toastTimer)
})
</script>

<style scoped>
:global(*) {
  box-sizing: border-box;
}

:global(body) {
  margin: 0;
  background: #f3f5f1;
  color: #1b2521;
  font-family: "Microsoft YaHei", "PingFang SC", "Noto Sans SC", sans-serif;
}

button,
input,
select,
textarea {
  font: inherit;
}

button {
  cursor: pointer;
}

.app {
  min-height: 100vh;
}

.topbar {
  position: sticky;
  top: 0;
  z-index: 10;
  display: flex;
  align-items: center;
  gap: 36px;
  height: 68px;
  padding: 0 32px;
  background: #19211e;
  color: #fff;
}

.brand {
  display: flex;
  align-items: center;
  gap: 11px;
  min-width: 180px;
  font-weight: 700;
  letter-spacing: 0.01em;
}

.brand-mark {
  display: grid;
  width: 30px;
  height: 30px;
  place-items: center;
  border-radius: 6px;
  background: #d8f061;
  color: #19211e;
}

.nav {
  display: flex;
  align-self: stretch;
  gap: 6px;
}

.nav button {
  border: 0;
  border-bottom: 2px solid transparent;
  background: transparent;
  color: #aeb9b3;
  padding: 0 14px;
}

.nav button.active {
  border-bottom-color: #d8f061;
  color: #fff;
}

.current-user {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-left: auto;
  color: #c9d1cc;
  font-size: 13px;
}

.avatar {
  display: grid;
  width: 30px;
  height: 30px;
  place-items: center;
  border-radius: 50%;
  background: #2f3b36;
}

.workspace {
  width: min(1240px, calc(100% - 48px));
  margin: 0 auto;
  padding: 42px 0 72px;
}

.page-heading {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 24px;
  margin-bottom: 28px;
}

.page-heading.compact {
  margin-top: 20px;
}

.page-heading h1,
.detail-title h1 {
  margin: 4px 0 8px;
  font-size: 30px;
  line-height: 1.25;
  letter-spacing: 0;
}

.page-heading p,
.detail-title p {
  margin: 0;
  color: #69736e;
}

.eyebrow {
  margin: 0;
  color: #237552 !important;
  font-size: 12px;
  font-weight: 700;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.primary,
.secondary,
.text-button,
.back-button {
  border-radius: 6px;
  padding: 10px 16px;
  font-weight: 700;
}

.primary {
  border: 1px solid #175f43;
  background: #176b4d;
  color: #fff;
}

.primary:hover {
  background: #12573e;
}

.primary:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

.secondary {
  border: 1px solid #cfd6d1;
  background: #fff;
  color: #35413b;
}

.text-button,
.back-button {
  border: 0;
  background: transparent;
  color: #176b4d;
  padding-left: 0;
}

.back-button {
  margin-bottom: 8px;
}

.toolbar {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 16px;
}

.search {
  width: min(440px, 100%);
}

input,
select,
textarea {
  width: 100%;
  border: 1px solid #cfd6d1;
  border-radius: 6px;
  background: #fff;
  color: #1b2521;
  padding: 11px 13px;
  outline: none;
}

input:focus,
select:focus,
textarea:focus {
  border-color: #237552;
  box-shadow: 0 0 0 3px rgb(35 117 82 / 12%);
}

textarea {
  resize: vertical;
}

.toolbar select {
  width: 150px;
}

.result-count {
  margin-left: auto;
  color: #69736e;
  font-size: 13px;
}

.panel,
.ticket-table {
  border: 1px solid #dfe4df;
  border-radius: 8px;
  background: #fff;
}

.ticket-table {
  overflow: hidden;
}

.ticket-row {
  display: grid;
  grid-template-columns: 155px minmax(250px, 1fr) 100px 90px 110px 110px;
  align-items: center;
  gap: 16px;
  width: 100%;
  min-height: 68px;
  padding: 0 20px;
  border: 0;
  border-bottom: 1px solid #edf0ed;
  background: #fff;
  color: #34413b;
  text-align: left;
}

.ticket-row:last-child {
  border-bottom: 0;
}

.table-header {
  min-height: 44px;
  background: #f7f8f5;
  color: #6c7671;
  font-size: 12px;
  font-weight: 700;
}

.ticket-item:hover {
  background: #f8faf7;
}

.ticket-number {
  color: #5d6862;
  font-family: Consolas, monospace;
  font-size: 12px;
}

.ticket-title {
  display: flex;
  min-width: 0;
  flex-direction: column;
  gap: 4px;
}

.ticket-title strong {
  overflow: hidden;
  color: #1b2521;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.ticket-title small,
.ticket-updated,
.muted {
  color: #79837d;
  font-size: 12px;
}

.badge {
  display: inline-flex;
  align-items: center;
  border-radius: 4px;
  padding: 4px 8px;
  font-size: 12px;
  font-weight: 700;
  white-space: nowrap;
}

.status-pending {
  background: #edf0ed;
  color: #53605a;
}

.status-processing {
  background: #e4f2ec;
  color: #176b4d;
}

.status-escalated {
  background: #fff0e4;
  color: #a84a18;
}

.status-resolved {
  background: #e7f3d8;
  color: #476a1d;
}

.status-closed {
  background: #eceff1;
  color: #5d6870;
}

.priority-low {
  background: #eef1ee;
  color: #667169;
}

.priority-normal {
  background: #e9eff5;
  color: #3c5e7f;
}

.priority-high {
  background: #fff3db;
  color: #946015;
}

.priority-urgent {
  background: #fce8e6;
  color: #a63c33;
}

.loading-panel,
.empty-state {
  padding: 52px 24px;
  text-align: center;
  color: #6e7872;
}

.empty-state h2 {
  margin: 0 0 8px;
  color: #26322c;
  font-size: 18px;
}

.empty-state p {
  margin: 0 0 20px;
}

.alert {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 16px;
  border-radius: 6px;
  padding: 12px 14px;
  font-size: 14px;
}

.alert.error {
  border: 1px solid #efcbc7;
  background: #fff4f2;
  color: #9c3830;
}

.alert.success {
  border: 1px solid #c7dfcf;
  background: #edf8f1;
  color: #176b4d;
}

.create-layout,
.detail-layout {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 310px;
  align-items: start;
  gap: 22px;
}

.form-panel,
.detail-main {
  padding: 26px;
}

.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 18px;
}

.field {
  margin-bottom: 20px;
}

.field label {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
  color: #303b35;
  font-size: 13px;
  font-weight: 700;
}

.field label span {
  color: #9a4a2f;
  font-weight: 400;
}

.field-meta {
  display: flex;
  min-height: 18px;
  justify-content: space-between;
  gap: 12px;
  padding-top: 5px;
  color: #8a938e;
  font-size: 11px;
}

.field-error {
  color: #b44236;
}

.has-error input,
.has-error textarea {
  border-color: #c75648;
}

.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  border-top: 1px solid #edf0ed;
  padding-top: 20px;
}

.summary-panel,
.detail-side {
  position: sticky;
  top: 90px;
  padding: 22px;
}

.summary-panel h2,
.detail-side h2,
.description h2,
.activity h2 {
  margin: 8px 0 20px;
  font-size: 18px;
  letter-spacing: 0;
}

dl {
  margin: 0;
}

dl div {
  display: flex;
  justify-content: space-between;
  gap: 18px;
  border-bottom: 1px solid #edf0ed;
  padding: 12px 0;
}

dl div:last-child {
  border-bottom: 0;
}

dt {
  color: #758079;
  font-size: 13px;
}

dd {
  margin: 0;
  color: #26322c;
  text-align: right;
  font-size: 13px;
  font-weight: 600;
  overflow-wrap: anywhere;
}

.detail-main {
  padding: 30px;
}

.detail-title {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 24px;
  border-bottom: 1px solid #edf0ed;
  padding-bottom: 24px;
}

.detail-title h1 {
  font-size: 25px;
}

.description {
  padding: 24px 0;
}

.description h2,
.activity h2 {
  margin-bottom: 12px;
}

.description p {
  margin: 0;
  color: #46524c;
  line-height: 1.8;
  white-space: pre-wrap;
}

.activity {
  border-top: 1px solid #edf0ed;
  padding-top: 24px;
}

.activity ol {
  margin: 0;
  padding: 0;
  list-style: none;
}

.activity li {
  position: relative;
  display: flex;
  gap: 14px;
  padding-bottom: 24px;
}

.activity li:not(:last-child)::before {
  position: absolute;
  top: 12px;
  bottom: 0;
  left: 5px;
  width: 1px;
  background: #dfe4df;
  content: "";
}

.timeline-dot {
  position: relative;
  z-index: 1;
  flex: 0 0 auto;
  width: 11px;
  height: 11px;
  margin-top: 4px;
  border: 2px solid #237552;
  border-radius: 50%;
  background: #fff;
}

.activity strong {
  color: #26322c;
  font-size: 14px;
}

.activity p {
  margin: 5px 0;
  color: #59655f;
  font-size: 13px;
  line-height: 1.6;
}

.activity small {
  color: #8a938e;
  font-size: 12px;
}

.toast {
  position: fixed;
  right: 24px;
  bottom: 24px;
  z-index: 20;
  border-radius: 6px;
  background: #1e2a24;
  color: #fff;
  padding: 12px 18px;
  box-shadow: 0 10px 30px rgb(20 30 25 / 20%);
}

@media (max-width: 900px) {
  .topbar {
    gap: 18px;
    padding: 0 18px;
  }

  .current-user span:last-child {
    display: none;
  }

  .workspace {
    width: min(100% - 28px, 760px);
    padding-top: 28px;
  }

  .create-layout,
  .detail-layout {
    grid-template-columns: 1fr;
  }

  .summary-panel,
  .detail-side {
    position: static;
  }

  .ticket-row {
    grid-template-columns: minmax(200px, 1fr) 100px 80px;
  }

  .table-header span:nth-child(1),
  .table-header span:nth-child(5),
  .table-header span:nth-child(6),
  .ticket-number,
  .ticket-assignee,
  .ticket-updated {
    display: none;
  }
}

@media (max-width: 600px) {
  .topbar {
    flex-wrap: wrap;
    height: auto;
    padding-top: 12px;
  }

  .brand {
    min-width: 0;
  }

  .nav {
    order: 3;
    width: 100%;
    height: 42px;
  }

  .nav button {
    flex: 1;
  }

  .page-heading {
    align-items: flex-start;
    flex-direction: column;
  }

  .page-heading .primary {
    width: 100%;
  }

  .toolbar {
    align-items: stretch;
    flex-direction: column;
  }

  .toolbar select {
    width: 100%;
  }

  .result-count {
    margin-left: 0;
  }

  .ticket-row {
    grid-template-columns: minmax(0, 1fr) auto;
    gap: 10px;
    padding: 13px 16px;
  }

  .table-header {
    display: none;
  }

  .ticket-item > span:nth-child(3),
  .ticket-item > span:nth-child(4) {
    display: none;
  }

  .ticket-title {
    grid-column: 1;
  }

  .ticket-item > span:nth-child(2) {
    grid-column: 2;
    grid-row: 1;
  }

  .form-grid {
    grid-template-columns: 1fr;
  }

  .form-panel,
  .detail-main {
    padding: 20px;
  }

  .detail-title {
    flex-direction: column;
  }

  .toast {
    right: 14px;
    bottom: 14px;
    left: 14px;
    text-align: center;
  }
}
</style>