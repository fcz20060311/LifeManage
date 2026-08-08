<script setup>
import { computed, nextTick, onBeforeUnmount, ref, watch } from 'vue'

const pad = n => String(n).padStart(2, '0')
const toDateKey = d => `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`
const today = toDateKey(new Date())
const storage = {
  get(key, fallback) { try { return JSON.parse(localStorage.getItem(`fuguang_${key}`)) ?? fallback } catch { return fallback } },
  set(key, value) { localStorage.setItem(`fuguang_${key}`, JSON.stringify(value)) }
}
const makeId = () => crypto.randomUUID?.() || `${Date.now()}-${Math.random()}`
const moneyText = n => `¥${Number(n).toLocaleString('zh-CN', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`

const page = ref('home')
const pageMeta = {
  home: ['TODAY', '你好，今天想做些什么？'], focus: ['FOCUS', '留一段完整时间给自己'],
  ledger: ['LEDGER', '每一笔，都心中有数'], calendar: ['CALENDAR', '把事情安放在时间里'],
  roadmap: ['GROWTH MAP', '把远方拆成一步一步'], insights: ['LIFE ARCHIVE', '生活留下痕迹，也长出方向']
}
const nav = [
  ['home', '⌂', '今日总览'], ['focus', '◷', '专注计时'], ['ledger', '¥', '记账本'],
  ['calendar', '□', '日历'], ['roadmap', '≡', '成长计划'], ['insights', '⌁', '生活记录']
]
const quotes = [
  ['生活不是赶路，而是感受路。', '愿你在寻常里，看见细小的光。'], ['去做具体的事，去爱具体的人。', '把注意力交还给真实的生活。'],
  ['缓慢而坚定，也是一种抵达。', '今天只向前一小步就很好。'], ['山有山的沉稳，风有风的自由。', '允许自己拥有不同的节奏。'],
  ['种一棵树最好的时间，是愿意开始的此刻。', '不必等待完美的一天。'], ['认真生活的人，自会与美好相逢。', '你的每一份用心都算数。'],
  ['允许一切发生，也相信一切会过去。', '松弛不是停下，而是更好地前行。'], ['日子常新，未来不远。', '请继续对明天抱有温柔的期待。'],
  ['心若安静，处处都是好风景。', '给自己留一点不被打扰的时间。'], ['今日事，今日慢慢做。', '从容完成，比匆忙完美更重要。'],
  ['不慌不忙，来日方长。', '你不需要和任何人的时钟同步。'], ['微小的日常，构成了具体的幸福。', '记得收藏今天值得喜欢的瞬间。']
]
const dayIndex = Math.floor(new Date(today + 'T00:00:00').getTime() / 86400000)
const dailyQuote = quotes[((dayIndex % quotes.length) + quotes.length) % quotes.length]

const tasks = ref(storage.get('tasks', []).map(t => ({ name: t.name ?? t.title, cat: t.cat ?? t.category, ...t })))
const transactions = ref(storage.get('money', storage.get('transactions', [])).map(t => ({ cat: t.cat ?? t.category, ...t })))
const logs = ref(storage.get('logs', storage.get('timeLogs', [])).map(t => ({ name: t.name ?? t.title, cat: t.cat ?? t.category, ...t })))
const plans = ref(storage.get('plans', []).map(p => ({ ...p, progressText: p.progressText || `${Number(p.progress || 0)}/100` })))
const focusData = ref(storage.get('focus', { date: today, minutes: 0, rounds: 0 }))
if (focusData.value.date !== today) focusData.value = { date: today, minutes: 0, rounds: 0 }
watch(tasks, v => storage.set('tasks', v), { deep: true })
watch(transactions, v => storage.set('money', v), { deep: true })
watch(logs, v => storage.set('logs', v), { deep: true })
watch(plans, v => storage.set('plans', v), { deep: true })
watch(focusData, v => storage.set('focus', v), { deep: true })

const notes = ref(storage.get('notes', storage.get('dailyNotes', {})))
const dailyNote = ref(notes.value[today] || '')
const noteState = ref('已自动保存')
let noteTimer
watch(dailyNote, value => {
  noteState.value = '正在保存…'
  clearTimeout(noteTimer)
  noteTimer = setTimeout(() => { notes.value[today] = value; storage.set('notes', notes.value); noteState.value = '已自动保存' }, 350)
})
function clearNote() { if (!dailyNote.value || confirm('清空今天的速记内容吗？')) dailyNote.value = '' }

const toast = ref('')
let toastTimer
function showToast(text) { toast.value = text; clearTimeout(toastTimer); toastTimer = setTimeout(() => toast.value = '', 1800) }

const taskDialog = ref(null)
const taskNameInput = ref(null)
const taskDraft = ref({ name: '', date: today, time: '', cat: '工作' })
function openTask(date = today) {
  taskDraft.value = { name: '', date, time: '', cat: '工作' }
  taskDialog.value.showModal()
  nextTick(() => taskNameInput.value?.focus())
}
function saveTask() {
  if (!taskDraft.value.name.trim()) return
  tasks.value.push({ id: makeId(), ...taskDraft.value, name: taskDraft.value.name.trim(), done: false })
  taskDialog.value.close(); showToast('任务已放进日历')
}
function toggleTask(id) { const task = tasks.value.find(t => t.id === id); if (task) { task.done = !task.done; showToast(task.done ? '任务已完成' : '任务已恢复') } }
function deleteTask(id) { tasks.value = tasks.value.filter(t => t.id !== id) }

const planDialog = ref(null)
const planNameInput = ref(null)
const planDraft = ref({ phase: '第1阶段', type: '', name: '', status: '待开始', start: today, end: today, progress: 0, progressText: '0/100' })
const planStatuses = ['待开始', '进行中', '已完成', '暂时搁置']
const sortedPlans = computed(() => plans.value)
const completedPlans = computed(() => plans.value.filter(x => x.status === '已完成').length)
const overallPlanProgress = computed(() => plans.value.length ? Math.round(plans.value.reduce((s, x) => s + Number(x.progress || 0), 0) / plans.value.length) : 0)
function openPlan() {
  const nextPhase = plans.value.length + 1
  planDraft.value = { phase: `第${nextPhase}阶段`, type: '', name: '', status: '待开始', start: today, end: today, progress: 0, progressText: '0/100' }
  planDialog.value.showModal(); nextTick(() => planNameInput.value?.focus())
}
function savePlan() {
  if (!planDraft.value.name.trim()) return
  calculatePlanProgress(planDraft.value)
  plans.value.push({ id: makeId(), ...planDraft.value, name: planDraft.value.name.trim(), progress: Number(planDraft.value.progress) })
  planDialog.value.close(); showToast('新的成长阶段已加入')
}
function updatePlanStatus(plan) {
  const match = String(plan.progressText || '').match(/^\s*(\d+(?:\.\d+)?)\s*\/\s*(\d+(?:\.\d+)?)\s*$/)
  const total = match && Number(match[2]) > 0 ? Number(match[2]) : 100
  if (plan.status === '已完成') { plan.progress = 100; plan.progressText = `${total}/${total}` }
  else if (plan.status === '待开始') { plan.progress = 0; plan.progressText = `0/${total}` }
}
function calculatePlanProgress(plan) {
  const match = String(plan.progressText || '').match(/^\s*(\d+(?:\.\d+)?)\s*\/\s*(\d+(?:\.\d+)?)\s*$/)
  if (!match || Number(match[2]) <= 0) return
  const completed = Number(match[1]), total = Number(match[2])
  plan.progress = Math.min(100, Math.max(0, Math.round(completed / total * 100)))
  if (plan.progress === 100) plan.status = '已完成'
  else if (plan.progress > 0 && plan.status === '待开始') plan.status = '进行中'
  else if (plan.progress < 100 && plan.status === '已完成') plan.status = '进行中'
}
function deletePlan(id) { if (confirm('删除这个成长阶段吗？')) plans.value = plans.value.filter(x => x.id !== id) }

const todayTasks = computed(() => tasks.value.filter(t => t.date === today).sort((a, b) => (a.time || '').localeCompare(b.time || '')))
const now = new Date()
const monthTransactions = computed(() => transactions.value.filter(x => { const d = new Date(`${x.date}T00:00:00`); return d.getMonth() === now.getMonth() && d.getFullYear() === now.getFullYear() }))
const monthIncome = computed(() => monthTransactions.value.filter(x => x.type === 'income').reduce((s, x) => s + Number(x.amount), 0))
const monthExpense = computed(() => monthTransactions.value.filter(x => x.type === 'expense').reduce((s, x) => s + Number(x.amount), 0))
const monthBalance = computed(() => monthIncome.value - monthExpense.value)

const secondsLeft = ref(0), timerRunning = ref(false), focusName = ref('')
let interval
let timerStartedAt = 0
let elapsedBeforeStart = 0
const timerDisplay = computed(() => `${pad(Math.floor(secondsLeft.value / 60))}:${pad(secondsLeft.value % 60)}`)
function syncElapsedTime() {
  if (!timerRunning.value) return
  secondsLeft.value = elapsedBeforeStart + Math.floor((Date.now() - timerStartedAt) / 1000)
}
function resetTimer() { clearInterval(interval); timerRunning.value = false; secondsLeft.value = 0; elapsedBeforeStart = 0; timerStartedAt = 0 }
function toggleTimer() {
  if (timerRunning.value) {
    syncElapsedTime(); timerRunning.value = false; elapsedBeforeStart = secondsLeft.value; clearInterval(interval); return
  }
  timerRunning.value = true
  elapsedBeforeStart = secondsLeft.value
  timerStartedAt = Date.now()
  syncElapsedTime()
  interval = setInterval(syncElapsedTime, 500)
}
function finishCountup() {
  syncElapsedTime()
  if (secondsLeft.value < 60) return showToast('至少专注一分钟再记录吧')
  const minutes = Math.max(1, Math.round(secondsLeft.value / 60))
  clearInterval(interval); timerRunning.value = false; focusData.value.minutes += minutes
  logs.value.unshift({ id: makeId(), name: focusName.value.trim() || '自由专注', cat: '工作', date: today, duration: minutes })
  secondsLeft.value = 0; elapsedBeforeStart = 0; timerStartedAt = 0; showToast(`已记录 ${minutes} 分钟专注`)
}
onBeforeUnmount(() => { clearInterval(interval); clearTimeout(noteTimer); clearTimeout(toastTimer) })

const txDraft = ref({ type: 'expense', amount: '', cat: '餐饮', date: today, note: '' })
function addTransaction() {
  transactions.value.unshift({ id: makeId(), ...txDraft.value, amount: Number(txDraft.value.amount) })
  txDraft.value = { type: 'expense', amount: '', cat: '餐饮', date: today, note: '' }; showToast('账目已保存')
}
function deleteTransaction(id) { transactions.value = transactions.value.filter(x => x.id !== id) }

const calendarDate = ref(new Date())
const calendarTitle = computed(() => `${calendarDate.value.getFullYear()} 年 ${calendarDate.value.getMonth() + 1} 月`)
const calendarDays = computed(() => {
  const y = calendarDate.value.getFullYear(), m = calendarDate.value.getMonth(), first = new Date(y, m, 1), start = new Date(first)
  start.setDate(start.getDate() - ((first.getDay() + 6) % 7))
  return Array.from({ length: 42 }, (_, i) => { const date = new Date(start); date.setDate(date.getDate() + i); const key = toDateKey(date); return { key, number: date.getDate(), other: date.getMonth() !== m, today: key === today, tasks: tasks.value.filter(t => t.date === key) } })
})
function moveMonth(step) { calendarDate.value = new Date(calendarDate.value.getFullYear(), calendarDate.value.getMonth() + step, 1) }

function startOfWeek() { const d = new Date(); const day = d.getDay() || 7; d.setDate(d.getDate() - day + 1); d.setHours(0, 0, 0, 0); return d }
function isThisWeek(key) { const d = new Date(`${key}T00:00:00`), start = startOfWeek(), end = new Date(start); end.setDate(end.getDate() + 7); return d >= start && d < end }
const weekLogs = computed(() => logs.value.filter(x => isThisWeek(x.date)))
const trackedMinutes = computed(() => weekLogs.value.reduce((s, x) => s + Number(x.duration), 0))
const categoryTotals = computed(() => weekLogs.value.reduce((all, x) => ({ ...all, [x.cat]: (all[x.cat] || 0) + Number(x.duration) }), {}))
const topCategory = computed(() => Object.entries(categoryTotals.value).sort((a, b) => b[1] - a[1])[0]?.[0] || '暂无')
const weekBars = computed(() => { const start = startOfWeek(); return Array.from({ length: 7 }, (_, i) => { const d = new Date(start); d.setDate(d.getDate() + i); return weekLogs.value.filter(x => x.date === toDateKey(d)).reduce((s, x) => s + Number(x.duration), 0) }) })
const maxBar = computed(() => Math.max(...weekBars.value, 1))
const logDraft = ref({ name: '', cat: '工作', date: today, duration: 60 })
function addLog() { logs.value.unshift({ id: makeId(), ...logDraft.value, name: logDraft.value.name.trim(), duration: Number(logDraft.value.duration) }); logDraft.value = { name: '', cat: '工作', date: today, duration: 60 }; showToast('时间已记录') }
function deleteLog(id) { logs.value = logs.value.filter(x => x.id !== id) }
const recentLogs = computed(() => logs.value.slice(0, 4))
const lifeRange = ref('week')
const archiveMonth = ref(today.slice(0, 7))
const monthLogs = computed(() => logs.value.filter(x => { const d = new Date(`${x.date}T00:00:00`); return d.getMonth() === now.getMonth() && d.getFullYear() === now.getFullYear() }))
const archiveLogs = computed(() => logs.value.filter(x => x.date?.startsWith(archiveMonth.value)))
const archiveTransactions = computed(() => transactions.value.filter(x => x.date?.startsWith(archiveMonth.value)))
const archiveIncome = computed(() => archiveTransactions.value.filter(x => x.type === 'income').reduce((s, x) => s + Number(x.amount), 0))
const archiveExpense = computed(() => archiveTransactions.value.filter(x => x.type === 'expense').reduce((s, x) => s + Number(x.amount), 0))
const activeLifeLogs = computed(() => lifeRange.value === 'week' ? weekLogs.value : archiveLogs.value)
const lifeCategoryTotals = computed(() => activeLifeLogs.value.reduce((all, x) => ({ ...all, [x.cat]: (all[x.cat] || 0) + Number(x.duration) }), {}))
const palette = { 工作: '#557b6d', 学习: '#7f9cad', 生活: '#d6ad67', 运动: '#d98068', 娱乐: '#9c8bac' }
const donutStyle = computed(() => {
  const entries = Object.entries(lifeCategoryTotals.value), total = entries.reduce((s, [, v]) => s + v, 0)
  if (!total) return { background: '#e7e5de' }
  let current = 0; const parts = entries.map(([key, value]) => { const start = current; current += value / total * 100; return `${palette[key] || '#aaa'} ${start}% ${current}%` })
  return { background: `conic-gradient(${parts.join(',')})` }
})
const expenseTotals = computed(() => archiveTransactions.value.filter(x => x.type === 'expense').reduce((all, x) => ({ ...all, [x.cat]: (all[x.cat] || 0) + Number(x.amount) }), {}))
const expensePalette = ['#557b6d','#7f9cad','#d6ad67','#d98068','#9c8bac','#91a985','#c79678']
const expenseDonutStyle = computed(() => {
  const entries = Object.entries(expenseTotals.value), total = entries.reduce((s, [, v]) => s + v, 0)
  if (!total) return { background: '#e7e5de' }
  let current = 0; return { background: `conic-gradient(${entries.map(([, value], i) => { const start = current; current += value / total * 100; return `${expensePalette[i % expensePalette.length]} ${start}% ${current}%` }).join(',')})` }
})
function exportMemories() {
  const payload = { exportedAt: new Date().toISOString(), month: archiveMonth.value, summary: { income: archiveIncome.value, expense: archiveExpense.value, balance: archiveIncome.value - archiveExpense.value }, transactions: archiveTransactions.value, lifeRecords: archiveLogs.value, dailyNotes: Object.fromEntries(Object.entries(notes.value).filter(([date]) => date.startsWith(archiveMonth.value))) }
  const blob = new Blob([JSON.stringify(payload, null, 2)], { type: 'application/json;charset=utf-8' })
  const url = URL.createObjectURL(blob), link = document.createElement('a')
  link.href = url; link.download = `浮光生活纪念册-${archiveMonth.value}.json`; link.click(); URL.revokeObjectURL(url)
  showToast('这个月的记忆已导出')
}
</script>

<template>
  <div class="shell">
    <aside>
      <div class="brand"><span>光</span><div><strong>浮光</strong><small>个人管理台</small></div></div>
      <nav><button v-for="item in nav" :key="item[0]" :class="{ active: page === item[0] }" @click="page = item[0]"><i>{{ item[1] }}</i>{{ item[2] }}</button></nav>
      <div class="side-quote"><small>今日一句 · 每日更新</small><p>{{ dailyQuote[0] }}</p></div>
    </aside>
    <main>
      <header class="topbar"><div><p class="eyebrow">{{ pageMeta[page][0] }}</p><h1>{{ pageMeta[page][1] }}</h1></div><button v-if="page !== 'focus' && page !== 'ledger' && page !== 'insights'" class="primary" @click="page === 'roadmap' ? openPlan() : openTask()">{{ page === 'roadmap' ? '＋ 添加阶段' : '＋ 新建任务' }}</button></header>

      <section v-if="page === 'home'">
        <div class="grid hero-grid"><article class="card focus-hero"><p class="eyebrow">今日专注</p><h2>把时间留给重要的事</h2><strong class="hero-time">{{ timerDisplay }}</strong><button class="round-play" @click="page = 'focus'">▶</button></article><article class="card quote-card"><small>DAILY WORDS · 每日更新</small><p>“{{ dailyQuote[0] }}”</p><span>{{ dailyQuote[1] }}</span></article></div>
        <div class="grid stats"><article class="card stat"><span>今日专注</span><strong>{{ focusData.minutes }}<small> 分钟</small></strong></article><article class="card stat"><span>待办任务</span><strong>{{ todayTasks.filter(t => !t.done).length }}<small> 项</small></strong></article><article class="card stat"><span>本月结余</span><strong>{{ moneyText(monthBalance) }}</strong></article></div>
        <div class="grid content-grid"><article class="card"><div class="card-head"><h2>今天的正式任务</h2><button class="text-btn" @click="page = 'calendar'">查看日历 →</button></div><div v-if="todayTasks.length" class="item-list"><div v-for="task in todayTasks" :key="task.id" class="item"><button class="check" :class="{ checked: task.done }" @click="toggleTask(task.id)">✓</button><div :class="['item-main', { done: task.done }]"><strong>{{ task.name }}</strong><small>{{ task.time || '全天' }}</small></div><span class="tag">{{ task.cat }}</span><button class="delete" @click="deleteTask(task.id)">×</button></div></div><div v-else class="empty">今天没有正式任务，随手想法可以写在速记里。</div></article><article class="card"><h2>最近的时间记录</h2><div v-if="recentLogs.length" class="item-list"><div v-for="log in recentLogs" :key="log.id" class="item"><span class="dot"></span><div class="item-main"><strong>{{ log.name }}</strong><small>{{ log.cat }} · {{ log.date }}</small></div><b>{{ log.duration }} 分钟</b></div></div><div v-else class="empty">记录后会显示在这里</div></article></div>
        <article class="quick-note"><div class="note-accent"></div><div class="note-head"><div><p class="eyebrow">QUICK NOTE</p><h2>今日速记</h2><span>这里不需要格式，想到什么就写什么。</span></div><button class="quiet-btn" @click="clearNote">清空</button></div><textarea v-model="dailyNote" maxlength="2000" placeholder="例如：&#10;· 下午联系小王&#10;· 买牛奶和咖啡豆&#10;· 晚上整理一下本周思路"></textarea><footer><span>{{ new Date().getMonth() + 1 }}月{{ new Date().getDate() }}日</span><span>{{ noteState }} · {{ dailyNote.length }}/2000</span></footer></article>
      </section>

      <section v-else-if="page === 'focus'" class="grid focus-layout">
        <article class="card timer-card">
          <p class="eyebrow">FLOW TIMER</p><h2 class="focus-heading">正向专注</h2><p class="countup-hint">从零开始，不设终点。结束后，这段光阴会被收进生活记录。</p>
          <div class="timer-ring countup"><div><small>此刻专注</small><strong>{{ timerDisplay }}</strong><span>今日已专注 {{ focusData.minutes }} 分钟</span></div></div>
          <input v-model="focusName" class="focus-input" placeholder="这次想专注做什么？"><div class="timer-actions"><button class="secondary" @click="resetTimer">↺ 重新开始</button><button class="primary" @click="toggleTimer">{{ timerRunning ? 'Ⅱ 暂停一下' : secondsLeft ? '▶ 继续计时' : '▶ 开始计时' }}</button><button v-if="secondsLeft" class="secondary finish-btn" @click="finishCountup">✓ 结束并记录</button></div>
        </article>
        <article class="card today-panel"><div class="card-head"><div><p class="eyebrow">TODAY'S TASKS</p><h2>今日应做</h2></div><span class="task-count">{{ todayTasks.filter(t => !t.done).length }} 项待办</span></div><div v-if="todayTasks.length" class="item-list"><div v-for="task in todayTasks" :key="task.id" class="item"><button class="check" :class="{ checked: task.done }" @click="toggleTask(task.id)">✓</button><div :class="['item-main', { done: task.done }]"><strong>{{ task.name }}</strong><small>{{ task.time || '全天' }} · {{ task.cat }}</small></div></div></div><div v-else class="empty">今天还没有任务<br><button class="text-btn" @click="openTask()">添加第一项安排 →</button></div><button v-if="todayTasks.length" class="add-task-inline" @click="openTask()">＋ 添加今日任务</button></article>
      </section>

      <section v-else-if="page === 'ledger'"><div class="grid stats"><article class="balance-card"><span>本月结余</span><strong>{{ moneyText(monthBalance) }}</strong></article><article class="card stat"><span>本月收入</span><strong class="green">{{ moneyText(monthIncome) }}</strong></article><article class="card stat"><span>本月支出</span><strong class="coral">{{ moneyText(monthExpense) }}</strong></article></div><div class="grid form-layout"><article class="card"><h2>记一笔</h2><form @submit.prevent="addTransaction"><div class="type-switch"><button type="button" :class="{ active: txDraft.type === 'expense' }" @click="txDraft.type = 'expense'">支出</button><button type="button" :class="{ active: txDraft.type === 'income' }" @click="txDraft.type = 'income'">收入</button></div><label>金额<input v-model="txDraft.amount" type="number" min="0.01" step="0.01" required placeholder="0.00"></label><div class="form-row"><label>分类<select v-model="txDraft.cat"><option v-for="x in ['餐饮','交通','购物','居住','学习','工资','其他']">{{ x }}</option></select></label><label>日期<input v-model="txDraft.date" type="date" required></label></div><label>备注<input v-model="txDraft.note" placeholder="可选"></label><button class="primary full">保存记录</button></form></article><article class="card"><h2>最近明细</h2><div v-if="transactions.length" class="item-list"><div v-for="tx in transactions.slice(0,20)" :key="tx.id" class="item"><span class="money-icon">{{ tx.cat[0] }}</span><div class="item-main"><strong>{{ tx.note || tx.cat }}</strong><small>{{ tx.date }} · {{ tx.cat }}</small></div><b :class="tx.type === 'income' ? 'green' : 'coral'">{{ tx.type === 'income' ? '+' : '−' }}{{ moneyText(tx.amount) }}</b><button class="delete" @click="deleteTransaction(tx.id)">×</button></div></div><div v-else class="empty">还没有账目记录</div></article></div></section>

      <section v-else-if="page === 'calendar'"><div class="calendar-toolbar"><button class="secondary" @click="moveMonth(-1)">←</button><h2>{{ calendarTitle }}</h2><button class="secondary" @click="moveMonth(1)">→</button></div><article class="card calendar-card"><div class="week-head"><span v-for="d in ['周一','周二','周三','周四','周五','周六','周日']">{{ d }}</span></div><div class="calendar-grid"><div v-for="day in calendarDays" :key="day.key" :class="['day', { other: day.other, today: day.today }]" @click="openTask(day.key)"><b>{{ day.number }}</b><button v-for="task in day.tasks.slice(0,3)" :key="task.id" :class="['event', { done: task.done }]" :title="task.done ? '点击恢复任务' : '点击完成任务'" @click.stop="toggleTask(task.id)">{{ task.name }}</button></div></div></article><p class="calendar-tip">提示：点击任务可完成或恢复；点击日期空白处可添加任务。</p></section>

      <section v-else-if="page === 'roadmap'">
        <div class="grid plan-stats"><article class="plan-overview"><div><p class="eyebrow">YOUR JOURNEY</p><h2>成长不是赶路，是沿途留下坐标。</h2><span>把想抵达的地方，拆成今天可以开始的一小步。</span></div><div class="overall-ring" :style="{ '--value': `${overallPlanProgress * 3.6}deg` }"><strong>{{ overallPlanProgress }}%</strong></div></article><article class="card stat"><span>全部阶段</span><strong>{{ plans.length }}<small> 个</small></strong></article><article class="card stat"><span>已经走过</span><strong>{{ completedPlans }}<small> 个</small></strong></article></div>
        <article class="card roadmap-card">
          <div class="card-head roadmap-head"><div><p class="eyebrow">ROADMAP</p><h2>我的成长路线</h2><span>状态和进度可在表格中直接修改，所有变化都会自动保存。</span></div><button class="secondary" @click="openPlan">＋ 新阶段</button></div>
          <div class="roadmap-scroll">
            <table class="roadmap-table">
              <thead><tr><th>阶段</th><th>类型</th><th>学习 / 成长内容</th><th>状态</th><th>时间安排</th><th>完成量 / 总量</th><th>进度</th><th></th></tr></thead>
              <tbody v-if="sortedPlans.length"><tr v-for="plan in sortedPlans" :key="plan.id" :class="{ completed: plan.status === '已完成' }"><td><span class="phase-number">{{ plan.phase }}</span></td><td><input v-model="plan.type" class="type-input" aria-label="自定义类型" placeholder="填写类型"></td><td><input v-model="plan.name" class="plan-name-input" aria-label="成长内容" placeholder="输入学习或成长内容"></td><td><select v-model="plan.status" :class="['status-select', `status-${plan.status}`]" @change="updatePlanStatus(plan)"><option v-for="status in planStatuses">{{ status }}</option></select></td><td><span class="date-range">{{ plan.start }}<i>→</i>{{ plan.end }}</span></td><td><input v-model="plan.progressText" class="fraction-input" inputmode="decimal" placeholder="例如 173/200" @input="calculatePlanProgress(plan)" @change="calculatePlanProgress(plan)"></td><td><div class="auto-progress"><div><i :style="{ width: `${plan.progress}%` }"></i></div><b>{{ plan.progress }}%</b></div></td><td><button class="delete" @click="deletePlan(plan.id)">×</button></td></tr></tbody>
            </table>
            <div v-if="!sortedPlans.length" class="roadmap-empty"><span>⌁</span><h3>还没有成长路线</h3><p>先写下第一个想学习、完成或长期坚持的方向。</p><button class="primary" @click="openPlan">添加第一个阶段</button></div>
          </div>
        </article>
      </section>

      <section v-else>
        <div class="archive-toolbar"><div><p class="eyebrow">MONTHLY MEMORY</p><h2>月度留存</h2><span>每个月的收支、时间与速记都会留在这里。</span></div><div class="archive-actions"><label>回看月份<input v-model="archiveMonth" type="month"></label><button class="primary" @click="exportMemories">⇩ 导出这个月</button></div></div>
        <div class="grid stats life-stats"><article class="balance-card"><span>{{ archiveMonth }} 月度结余</span><strong>{{ moneyText(archiveIncome - archiveExpense) }}</strong><small>收入 {{ moneyText(archiveIncome) }} · 支出 {{ moneyText(archiveExpense) }}</small></article><article class="card stat"><span>留下的生活片段</span><strong>{{ archiveLogs.length + archiveTransactions.length }}<small> 条</small></strong></article><article class="card stat"><span>本周拾取的光阴</span><strong>{{ (trackedMinutes / 60).toFixed(1) }}<small> 小时</small></strong></article></div>
        <div class="grid life-grid">
          <article class="card chart-card"><div class="card-head"><div><p class="eyebrow">MONEY FLOW</p><h2>人间烟火账</h2><span class="chart-subtitle">一蔬一饭，也是一月生活的注脚。</span></div><span class="soft-pill">{{ moneyText(archiveExpense) }}</span></div><div class="donut-layout"><div class="donut" :style="expenseDonutStyle"><div><strong>{{ archiveTransactions.filter(x => x.type === 'expense').length }}</strong><span>笔支出</span></div></div><div class="chart-legend"><div v-for="([cat,value],i) in Object.entries(expenseTotals).sort((a,b)=>b[1]-a[1])" :key="cat"><i :style="{ background: expensePalette[i % expensePalette.length] }"></i><span>{{ cat }}</span><b>{{ moneyText(value) }}</b></div><p v-if="!archiveExpense">这个月还没有支出记录</p></div></div></article>
          <article class="card chart-card"><div class="card-head"><div><p class="eyebrow">TIME FLOW</p><h2>光阴落处</h2><span class="chart-subtitle">看见时间停驻过的地方。</span></div><div class="range-switch"><button :class="{ active: lifeRange === 'week' }" @click="lifeRange = 'week'">本周</button><button :class="{ active: lifeRange === 'month' }" @click="lifeRange = 'month'">所选月</button></div></div><div class="donut-layout"><div class="donut" :style="donutStyle"><div><strong>{{ (activeLifeLogs.reduce((s,x)=>s + Number(x.duration),0) / 60).toFixed(1) }}</strong><span>小时</span></div></div><div class="chart-legend"><div v-for="([cat,value]) in Object.entries(lifeCategoryTotals).sort((a,b)=>b[1]-a[1])" :key="cat"><i :style="{ background: palette[cat] }"></i><span>{{ cat }}</span><b>{{ (value / 60).toFixed(1) }}h</b></div><p v-if="!activeLifeLogs.length">还没有时间记录</p></div></div></article>
        </div>
        <div class="grid form-layout section-gap"><article class="card"><div class="card-head"><div><p class="eyebrow">ADD A MOMENT</p><h2>拾取一段生活</h2></div></div><form @submit.prevent="addLog"><label>这段时间做了什么<input v-model="logDraft.name" required placeholder="例如：画稿、阅读、散步"></label><div class="form-row"><label>分类<select v-model="logDraft.cat"><option v-for="x in ['工作','学习','生活','运动','娱乐']">{{ x }}</option></select></label><label>日期<input v-model="logDraft.date" type="date" required></label></div><label>用时（分钟）<input v-model="logDraft.duration" type="number" min="5" step="5" required></label><button class="primary full">收入生活册</button></form></article><article class="card"><div class="card-head"><div><p class="eyebrow">RECENT MOMENTS</p><h2>近日拾光</h2></div></div><div v-if="logs.length" class="item-list"><div v-for="log in logs.slice(0,8)" :key="log.id" class="item"><span class="dot" :style="{ background: palette[log.cat] }"></span><div class="item-main"><strong>{{ log.name }}</strong><small>{{ log.date }} · {{ log.cat }}</small></div><b>{{ log.duration }} 分钟</b><button class="delete" @click="deleteLog(log.id)">×</button></div></div><div v-else class="empty">从记录今天的一小段生活开始吧。</div></article></div>
      </section>
    </main>
  </div>

  <dialog ref="taskDialog"><form class="dialog-form" @submit.prevent="saveTask"><div class="dialog-head"><div><p class="eyebrow">NEW TASK</p><h2>添加正式任务</h2></div><button type="button" class="close" @click="taskDialog.close()">×</button></div><label>任务名称<input ref="taskNameInput" v-model="taskDraft.name" required placeholder="想做什么？"></label><div class="form-row"><label>日期<input v-model="taskDraft.date" type="date" required></label><label>时间（可不填）<input v-model="taskDraft.time" type="time"></label></div><label>分类<select v-model="taskDraft.cat"><option v-for="x in ['工作','学习','生活','运动']">{{ x }}</option></select></label><div class="dialog-actions"><button type="button" class="secondary" @click="taskDialog.close()">取消</button><button class="primary">保存任务</button></div></form></dialog>
  <dialog ref="planDialog"><form class="dialog-form" @submit.prevent="savePlan"><div class="dialog-head"><div><p class="eyebrow">NEW MILESTONE</p><h2>添加成长阶段</h2></div><button type="button" class="close" @click="planDialog.close()">×</button></div><div class="form-row"><label>阶段名称 / 序号<input v-model="planDraft.phase" type="text" required placeholder="例如：第一阶段、基础入门"></label><label>自定义类型<input v-model="planDraft.type" type="text" required placeholder="例如：技术线、作品集、长期习惯"></label></div><label>学习 / 成长内容<input ref="planNameInput" v-model="planDraft.name" type="text" required placeholder="例如：完成 Vue 3 系统学习"></label><div class="form-row"><label>开始日期<input v-model="planDraft.start" type="date" required></label><label>结束日期<input v-model="planDraft.end" type="date" :min="planDraft.start" required></label></div><div class="form-row"><label>初始状态<select v-model="planDraft.status"><option v-for="status in planStatuses">{{ status }}</option></select></label><label>完成量 / 总量<input v-model="planDraft.progressText" type="text" inputmode="decimal" required placeholder="例如：173/200" @input="calculatePlanProgress(planDraft)"></label></div><div class="progress-preview"><span>自动计算进度</span><strong>{{ planDraft.progress }}%</strong></div><div class="dialog-actions"><button type="button" class="secondary" @click="planDialog.close()">取消</button><button class="primary">加入成长路线</button></div></form></dialog>
  <Transition name="toast"><div v-if="toast" class="toast">{{ toast }}</div></Transition>
</template>
