<script setup>
import { ref, computed, reactive } from 'vue'
import { useParkStore, newRequestId } from '@/store/park'

const store = useParkStore()

const tabs = [
  { k: 'board', label: '排班看板' },
  { k: 'attendance', label: '考勤工时' },
  { k: 'requests', label: '调班加班' },
  { k: 'coverage', label: '岗位覆盖' }
]
const tab = ref('board')

const today = computed(() => store.clock.day)
const days = computed(() => {
  const d = today.value
  return [d, d + 1, d + 2, d + 3]
})
const dayNames = d => d === today.value ? '今天' : d === today.value + 1 ? '明天' : `第${d}天`
const dayNames2 = d => dayNames(d)

// 按 日 × 员工 组织排班
const schedMap = computed(() => {
  const m = new Map()
  for (const s of store.schedules) {
    if (!m.has(s.day)) m.set(s.day, new Map())
    m.get(s.day).set(s.staff_id, s)
    if (s.board_day && s.board_day !== s.day && !m.get(s.board_day)?.has(s.staff_id)) {
      if (!m.has(s.board_day)) m.set(s.board_day, new Map())
      m.get(s.board_day).set(s.staff_id, { ...s, _cross_day_board: true })
    }
  }
  return m
})
const schedOf = (staffId, day) => schedMap.value.get(day)?.get(staffId)

const activeStaff = computed(() => store.staff.filter(s => s.active))
const nonSupervisor = computed(() => activeStaff.value.filter(s => s.role !== '运营主管'))

// 运营主管手动排班
const form = reactive({ staff_id: null, shift_id: null, day: null })
const formMsg = ref('')
function resetForm() { form.staff_id = null; form.shift_id = null; form.day = today.value }
resetForm()
async function submitSchedule() {
  if (!form.staff_id || !form.shift_id || !form.day) { formMsg.value = '请选择员工、班次与日期'; return }
  const r = await store.scheduleShift({
    staff_id: form.staff_id, shift_id: form.shift_id, day: form.day, request_id: newRequestId()
  })
  if (r?.ok) {
    formMsg.value = `排班成功 ${r.code || ''}${r.warnings?.length ? `，但有 ${r.warnings.length} 条岗位覆盖预警，请查看「岗位覆盖」` : ''}`
  } else formMsg.value = r?.msg ? `${r.msg}${r.code ? `〔${r.code}〕` : ''}` : '排班失败'
}
async function cancelS(s) {
  if (!confirm(`取消 ${s.staff_name} 第${s.day}天的「${s.shift_name}」排班？`)) return
  await store.cancelSchedule(s.id, newRequestId())
}

// 员工协作：调班 / 加班（行内表单）
const swaps = reactive({})   // scheduleId -> { target_staff_id, target_shift_id, target_day, reason }
const otForm = reactive({})   // scheduleId -> { ticks, reason }
function swapDraft(s) {
  if (!swaps[s.id]) swaps[s.id] = { target_staff_id: null, target_shift_id: s.shift_id, target_day: s.day, reason: '', open: false }
  return swaps[s.id]
}
function otDraft(s) {
  if (!otForm[s.id]) otForm[s.id] = { ticks: 1, reason: '', open: false }
  return otForm[s.id]
}
async function submitSwap(s) {
  const f = swaps[s.id]
  if (!f?.target_staff_id) return
  const r = await store.requestSwap(s.id, {
    staff_id: s.staff_id,
    target_staff_id: f.target_staff_id, target_shift_id: f.target_shift_id, target_day: f.target_day,
    reason: f.reason, request_id: newRequestId()
  })
  if (r?.ok) { f.open = false } else alert(r?.msg ? `${r.msg}〔${r.code || ''}〕` : '调班申请失败')
}
async function submitOt(s) {
  const f = otForm[s.id]
  const r = await store.requestOvertime(s.id, {
    staff_id: s.staff_id, ticks: f.ticks, reason: f.reason, request_id: newRequestId()
  })
  if (r?.ok) { f.open = false } else alert(r?.msg ? `${r.msg}〔${r.code || ''}〕` : '加班申请失败')
}
async function checkin(s) {
  const r = await store.checkinSchedule(s.id, newRequestId())
  if (!r?.ok) alert(r?.msg || '打卡失败')
}
async function leave(s) {
  const reason = prompt('登记离岗原因（当前考勤将按实际工时立即结算）：', '个人原因')
  if (reason === null) return
  const r = await store.leaveAttendance(s.attendance_id, reason, newRequestId())
  if (!r?.ok) alert(r?.msg || '离岗失败')
}

// ---- 考勤工时 ----
const attTab = ref('today')
const attendanceList = computed(() => {
  if (attTab.value === 'today') return store.attendance
  if (attTab.value === 'onduty') return store.attendance.filter(a => a.status === 'checked_in')
  if (attTab.value === 'absent') return store.attendance.filter(a => a.status === 'absent' || a.status === 'leave')
  return store.attendance
})
const statusMeta = st => ({
  checked_in: { label: '在岗', cls: 'st-on' },
  checked_out: { label: '已结算', cls: 'st-out' },
  absent: { label: '旷工', cls: 'st-absent' },
  leave: { label: '离岗', cls: 'st-leave' }
}[st] || { label: st, cls: '' })

// ---- 调班 / 加班申请 ----
const reqTab = ref('pending')
const requestList = computed(() => {
  if (reqTab.value === 'all') return store.shiftRequests
  return store.shiftRequests.filter(r => r.status === reqTab.value)
})
const reqStatusMeta = st => ({
  pending: { label: '待审批', cls: 'st-on' },
  approved: { label: '已批准', cls: 'st-out' },
  rejected: { label: '已驳回', cls: 'st-absent' },
  cancelled: { label: '已取消', cls: 'st-leave' }
}[st] || { label: st, cls: '' })

const approverId = computed(() => store.supervisors[0]?.id || null)
async function approve(r) {
  const out = await store.approveShiftRequest(r.id, approverId.value)
  if (out?.ok && out.warnings?.length) {
    alert(`调班已批准，但产生 ${out.warnings.length} 条覆盖预警，请查看「岗位覆盖」`)
  }
}
async function reject(r) {
  const note = prompt('驳回原因（可选）：', '')
  if (note === null) return
  await store.rejectShiftRequest(r.id, approverId.value, note)
}
async function withdraw(r) { await store.cancelShiftRequest(r.id, r.staff_id) }

// ---- 岗位覆盖 ----
const coverage = computed(() => store.coverageToday)
const coverageDay = ref(0)
const coverageData = computed(() => store.coverageByDay[coverageDay.value] || coverage.value)
const forecastTotal = computed(() => coverageData.value.signals?.dayTotal || 0)
const activeCoverage = computed(() => coverageData.value.warnings.filter(w => w.status === 'active'))
const upcomingCoverage = computed(() => coverageData.value.warnings.filter(w => w.status !== 'active'))
function selectCoverageDay(d) { coverageDay.value = d }
const warnIcon = lv => lv === 'block' ? '⛔' : lv === 'info' ? '🕘' : '⚠️'
const demandMeta = {
  baseline: { label: '基础覆盖', cls: 'baseline' },
  traffic: { label: '预约客流', cls: 'traffic' },
  maintenance: { label: '检修工单', cls: 'maintenance' },
  complaint: { label: '投诉岗位', cls: 'complaint' },
  night: { label: '跨日夜班', cls: 'night' }
}
function demandTag(s) {
  return demandMeta[s.demand_type]?.label || (s.source === 'swap' ? '调班接替' : s.source === 'auto' ? '动态补位' : '人工排班')
}

// 排班状态徽标
function schedState(s) {
  if (!s) return { text: '休息', cls: 'off' }
  if (s.status === 'cancelled') return { text: '已取消', cls: 'cancelled' }
  if (s.status === 'swap') return { text: '调班中', cls: 'swap' }
  if (s.att_status === 'checked_in') return { text: s.on_duty ? '在岗' : '考勤中', cls: 'on' }
  if (s.att_status === 'checked_out') return { text: '已下班', cls: 'done' }
  if (s.att_status === 'absent') return { text: '旷工', cls: 'absent' }
  if (s.att_status === 'leave') return { text: '离岗', cls: 'leave' }
  return { text: '已排班', cls: 'plan' }
}

// 时间线
const detailLogs = ref([])
const detailTitle = ref('')
async function openLogs(s) {
  let logs = []
  if (s.attendance_id) ({ logs } = await store.scheduleLogs({ attendanceId: s.attendance_id }))
  else ({ logs } = await store.scheduleLogs({ scheduleId: s.id }))
  detailLogs.value = logs || []
  detailTitle.value = `${s.staff_name} · 第${s.day}天 · ${s.shift_name}`
}
async function openReqLogs(r) {
  const { logs } = await store.scheduleLogs({ requestId: r.id })
  detailLogs.value = logs || []
  detailTitle.value = `${r.code} · ${r.staff_name}`
}
function closeLogs() { detailLogs.value = [] }
const ACTION_LABEL = {
  schedule: '排班', autofill: '自动排班', cancel: '取消排班',
  checkin: '打卡上班', late: '迟到', leave: '离岗', checkout: '下班结算', absent: '旷工',
  workdone: '完工回写',
  swap_request: '申请调班', swap_approve: '批准调班', swap_reject: '驳回调班',
  ot_request: '申请加班', ot_approve: '批准加班', ot_reject: '驳回加班'
}

const stats = computed(() => store.schedulingStats)
// 自动排班开关来自后端配置（scheduleAutoFill 未单独下发时用 /state 的排班数据反推不准，
// 故用本地态初始化并在切换时落库；刷新页面后由引擎行为兜底）
const autoFill = ref(true)
async function toggleAutoFill(v) {
  autoFill.value = v
  await store.saveScheduleConfig({ auto_fill: v ? 1 : 0 })
}
</script>

<template>
  <div class="sch">
    <div class="stat-grid">
      <div class="card stat"><span>🗓️</span><b>{{ stats.todayScheduled }}</b><em>今日排班</em></div>
      <div class="card stat"><span>👷</span><b class="on">{{ stats.onDuty }}</b><em>当前在岗</em></div>
      <div class="card stat" :class="{ alert: stats.absentToday }"><span>❌</span><b :class="stats.absentToday ? 'neg' : ''">{{ stats.absentToday }}</b><em>今日旷工</em></div>
      <div class="card stat"><span>⏰</span><b>{{ stats.lateToday }}</b><em>今日迟到</em></div>
      <div class="card stat"><span>📈</span><b>{{ stats.reservationForecast || 0 }}</b><em>预约客流预测</em></div>
      <div class="card stat"><span>🕑</span><b>{{ stats.overtimeHoursToday }}h</b><em>今日加班</em></div>
      <div class="card stat"><span>💰</span><b class="money neg">¥{{ (stats.payToday || 0).toLocaleString() }}</b><em>今日工时工资</em></div>
      <div class="card stat" :class="{ alert: stats.pendingRequests }"><span>📝</span><b>{{ stats.pendingRequests }}</b><em>待审批申请</em></div>
      <div class="card stat" :class="{ alert: stats.coverageWarnings || stats.upcomingWarnings }"><span>⚠️</span><b>{{ stats.coverageWarnings }}/{{ stats.upcomingWarnings || 0 }}</b><em>当前/即将缺岗</em></div>
    </div>

    <div class="tabs card-tabs">
      <button v-for="t in tabs" :key="t.k" :class="{ on: tab === t.k }" @click="tab = t.k">
        {{ t.label }}
        <i v-if="t.k === 'requests' && stats.pendingRequests" class="badge-dot">{{ stats.pendingRequests }}</i>
      </button>
      <label class="autofill" v-if="tab === 'board'">
        <input type="checkbox" v-model="autoFill" @change="toggleAutoFill($event.target.checked)" />
        动态自动补位（预约/检修/投诉驱动 · 未来 3 天）
      </label>
    </div>

    <!-- ============ 排班看板 ============ -->
    <template v-if="tab === 'board'">
      <div class="board-wrap card">
        <div class="board">
          <div class="bcol corner">
            <span class="muted">员工 ＼ 日期</span>
            <div class="shift-legend">
              <i v-for="sh in store.shifts" :key="sh.id" :style="{ background: sh.color }">{{ sh.name }}<em>{{ sh.time_text }}</em></i>
            </div>
          </div>
          <div class="bcol" v-for="d in days" :key="d">
            <div class="day-head" :class="{ today: d === today }">{{ dayNames(d) }}<em>第 {{ d }} 天</em></div>
          </div>

          <template v-for="st in nonSupervisor" :key="st.id">
            <div class="bcol corner staff-cell">
              <b>{{ st.name }}</b><em class="muted">{{ st.role }} · {{ store.zones.find(z => z.id === st.zone_id)?.name || '—' }}</em>
            </div>
            <div class="bcol" v-for="d in days" :key="st.id + '-' + d">
              <div v-if="schedOf(st.id, d)" class="shift-card"
                   :class="schedState(schedOf(st.id, d)).cls"
                   :style="{ borderColor: schedOf(st.id, d).shift_color + '88' }">
                <div class="sc-top">
                  <span class="sc-name" :style="{ color: schedOf(st.id, d).shift_color }">{{ schedOf(st.id, d).shift_name }}</span>
                  <span class="sc-state">{{ schedState(schedOf(st.id, d)).text }}</span>
                </div>
                <div class="sc-time muted">{{ schedOf(st.id, d).time_text }}</div>
                <div class="sc-tags">
                  <em class="demand" :class="schedOf(st.id, d).demand_type || schedOf(st.id, d).source">{{ demandTag(schedOf(st.id, d)) }}</em>
                  <em v-if="schedOf(st.id, d).cross_day">跨日</em>
                  <em v-if="schedOf(st.id, d).late">迟到</em>
                  <em v-if="schedOf(st.id, d).ot_approved">加班{{ schedOf(st.id, d).overtime_ticks }}h</em>
                  <em v-if="schedOf(st.id, d).work_ticks">出勤{{ schedOf(st.id, d).work_ticks }}h</em>
                  <em v-if="schedOf(st.id, d).pay" class="money neg">¥{{ schedOf(st.id, d).pay }}</em>
                </div>
                <div class="sc-actions">
                  <button class="ghost sm" @click="openLogs(schedOf(st.id, d))">时间线</button>
                  <button v-if="!schedOf(st.id, d).att_status && schedOf(st.id, d).status !== 'cancelled'" class="ghost sm"
                          @click="checkin(schedOf(st.id, d))">打卡</button>
                  <button v-if="schedOf(st.id, d).on_duty" class="ghost sm danger" @click="leave(schedOf(st.id, d))">离岗</button>
                  <button v-if="!schedOf(st.id, d).att_status && schedOf(st.id, d).status !== 'cancelled'" class="ghost sm"
                          @click="swapDraft(schedOf(st.id, d)).open = !swapDraft(schedOf(st.id, d)).open">调班</button>
                  <button v-if="schedOf(st.id, d).on_duty" class="ghost sm"
                          @click="otDraft(schedOf(st.id, d)).open = !otDraft(schedOf(st.id, d)).open">加班</button>
                  <button v-if="!schedOf(st.id, d).att_status && schedOf(st.id, d).status !== 'cancelled'" class="ghost sm danger"
                          @click="cancelS(schedOf(st.id, d))">取消</button>
                </div>
                <!-- 调班申请 -->
                <div class="inline-form" v-if="swaps[schedOf(st.id, d).id]?.open">
                  <select v-model.number="swaps[schedOf(st.id, d).id].target_staff_id">
                    <option :value="null" disabled>代班同事…</option>
                    <option v-for="t in nonSupervisor.filter(x => x.id !== st.id)" :key="t.id" :value="t.id">{{ t.name }} · {{ t.role }}</option>
                  </select>
                  <select v-model.number="swaps[schedOf(st.id, d).id].target_shift_id">
                    <option v-for="sh in store.shifts" :key="sh.id" :value="sh.id">{{ sh.name }} {{ sh.time_text }}</option>
                  </select>
                  <select v-model.number="swaps[schedOf(st.id, d).id].target_day">
                    <option v-for="dd in days" :key="dd" :value="dd">{{ dayNames(dd) }}</option>
                  </select>
                  <input v-model="swaps[schedOf(st.id, d).id].reason" placeholder="调班原因（可选）" maxlength="120" />
                  <button class="succ sm" @click="submitSwap(schedOf(st.id, d))">提交主管审批</button>
                </div>
                <!-- 加班申请 -->
                <div class="inline-form" v-if="otForm[schedOf(st.id, d).id]?.open">
                  <span class="muted sm-text">延后下班</span>
                  <select v-model.number="otForm[schedOf(st.id, d).id].ticks">
                    <option :value="1">1h</option><option :value="2">2h</option><option :value="3">3h</option><option :value="4">4h</option>
                  </select>
                  <input v-model="otForm[schedOf(st.id, d).id].reason" placeholder="加班事由（可选）" maxlength="120" />
                  <button class="succ sm" @click="submitOt(schedOf(st.id, d))">提交加班申请</button>
                </div>
              </div>
              <span v-else class="rest muted">休</span>
            </div>
          </template>
        </div>
      </div>

      <!-- 主管排班 -->
      <div class="card assign-card">
        <h3>🧑‍💼 运营主管排班</h3>
        <div class="assign-row">
          <label>员工
            <select v-model.number="form.staff_id">
              <option :value="null" disabled>选择员工…</option>
              <option v-for="s in nonSupervisor" :key="s.id" :value="s.id">{{ s.name }} · {{ s.role }} · {{ store.zones.find(z => z.id === s.zone_id)?.name }}</option>
            </select>
          </label>
          <label>班次
            <select v-model.number="form.shift_id">
              <option :value="null" disabled>选择班次…</option>
              <option v-for="sh in store.shifts" :key="sh.id" :value="sh.id">{{ sh.name }} · {{ sh.time_text }} · 基准 {{ sh.standard_hours }}h</option>
            </select>
          </label>
          <label>日期
            <select v-model.number="form.day">
              <option v-for="d in days" :key="d" :value="d">{{ dayNames(d) }}（第 {{ d }} 天）</option>
            </select>
          </label>
          <button class="primary" @click="submitSchedule">排入班次</button>
          <em class="mmsg" :class="{ bad: formMsg.includes('失败') || formMsg.includes('冲突') || formMsg.includes('不能') }">{{ formMsg }}</em>
        </div>
        <p class="muted tips">说明：同一员工同日仅允许一个有效排班（冲突拦截）；自动补位只在出现预约客流、检修工单、投诉或夜间值守需求时生成排班，并在排班单上标注需求来源。跨日夜班当日 17:00 上班、次日 09:00 下班并结算，看板次日仍会展示在岗状态。</p>
      </div>
    </template>

    <!-- ============ 考勤工时 ============ -->
    <template v-else-if="tab === 'attendance'">
      <div class="card">
        <div class="tabs">
          <button :class="{ on: attTab === 'today' }" @click="attTab = 'today'">今日结算</button>
          <button :class="{ on: attTab === 'onduty' }" @click="attTab = 'onduty'">在岗中</button>
          <button :class="{ on: attTab === 'absent' }" @click="attTab === 'absent' ? attTab = 'all' : attTab = 'absent'">旷工/离岗</button>
        </div>
        <div class="atable">
          <div class="ahead"><span>考勤号</span><span>员工</span><span>班次</span><span>状态</span><span>出勤</span><span>加班</span><span>满意度</span><span>结算工资</span><span>结算日</span><span></span></div>
          <div class="arow" v-for="a in attendanceList" :key="a.id">
            <span class="mono">{{ a.code }}</span>
            <span><b>{{ a.staff_name }}</b><em class="muted">{{ a.staff_role }}</em></span>
            <span><i class="dot-sh" :style="{ background: a.shift_color }"></i>{{ a.shift_name }}<em class="muted">{{ a.time_text }}</em></span>
            <span><span class="abadge" :class="statusMeta(a.status).cls">{{ statusMeta(a.status).label }}</span><em v-if="a.late" class="late-tag">迟到</em></span>
            <span>{{ a.status === 'checked_in' ? '进行中' : a.work_ticks + 'h' }}</span>
            <span>{{ a.ot_approved ? a.overtime_ticks + 'h（1.5×）' : '—' }}</span>
            <span :class="a.satisfaction_delta > 0 ? 'pos' : a.satisfaction_delta < 0 ? 'neg' : ''">
              {{ a.satisfaction_delta ? (a.satisfaction_delta > 0 ? '+' : '') + a.satisfaction_delta : '—' }}
            </span>
            <span class="money neg" v-if="a.pay">¥{{ a.pay.toLocaleString() }}</span>
            <span class="muted" v-else>—</span>
            <span>第{{ a.settle_day || a.day }}天</span>
            <span><button class="ghost sm" @click="openLogs({ id: a.schedule_id, attendance_id: a.id, staff_name: a.staff_name, day: a.day, shift_name: a.shift_name })">时间线</button></span>
          </div>
          <div class="muted empty" v-if="!attendanceList.length">暂无考勤记录，引擎会在班次开始时自动打卡。</div>
        </div>
        <p class="muted tips">工资口径：时薪 = 日薪 ÷ 5；下班按班次基准工时结算，加班按 {{ store.data ? '1.5' : '1.5' }} 倍时薪另计，旷工无薪，中途离岗按实际出勤比例折算。每张考勤单下班时逐条写入「工资」财务流水，跨日夜班计入次日。</p>
      </div>
    </template>

    <!-- ============ 调班 / 加班 ============ -->
    <template v-else-if="tab === 'requests'">
      <div class="card">
        <div class="tabs">
          <button :class="{ on: reqTab === 'pending' }" @click="reqTab = 'pending'">待审批（{{ stats.pendingRequests }}）</button>
          <button :class="{ on: reqTab === 'approved' }" @click="reqTab = 'approved'">已批准</button>
          <button :class="{ on: reqTab === 'rejected' }" @click="reqTab = 'rejected'">已驳回</button>
          <button :class="{ on: reqTab === 'all' }" @click="reqTab = 'all'">全部</button>
        </div>
        <div class="reqlist">
          <div class="req card2" v-for="r in requestList" :key="r.id">
            <div class="rq-head">
              <span class="rq-kind" :class="r.kind">{{ r.kind === 'swap' ? '🔄 调班申请' : '🕑 加班申请' }}</span>
              <span class="mono muted">{{ r.code }}</span>
              <span class="abadge" :class="reqStatusMeta(r.status).cls">{{ reqStatusMeta(r.status).label }}</span>
            </div>
            <div class="rq-body">
              <template v-if="r.kind === 'swap'">
                <b>{{ r.staff_name }}</b><em class="muted">（{{ r.staff_role }} · 第{{ r.day }}天 {{ r.shift_name }}）</em>
                <span class="arrow">→</span>
                <b>{{ r.target_name }}</b><em class="muted">（{{ r.target_role }}）</em>
                <span class="tag">{{ dayNames2(r.target_day) }} · {{ r.target_shift_name }}{{ r.target_cross_day ? '（跨日）' : '' }}</span>
              </template>
              <template v-else>
                <b>{{ r.staff_name }}</b><em class="muted">（{{ r.staff_role }} · 当值 {{ r.shift_name }}）</em>
                <span class="tag ot">申请延后下班 +{{ r.ot_ticks }}h（1.5 倍时薪）</span>
              </template>
              <p class="muted reason" v-if="r.reason">事由：{{ r.reason }}</p>
              <p class="muted reason" v-if="r.handle_note">处理：{{ r.handle_note }}</p>
            </div>
            <div class="rq-foot">
              <button class="ghost sm" @click="openReqLogs(r)">时间线</button>
              <template v-if="r.status === 'pending'">
                <button class="succ sm" @click="approve(r)">主管批准</button>
                <button class="danger sm" @click="reject(r)">驳回</button>
                <button class="ghost sm" @click="withdraw(r)">员工撤回</button>
              </template>
            </div>
          </div>
          <div class="muted empty" v-if="!requestList.length">暂无调班 / 加班申请。</div>
        </div>
      </div>
    </template>

    <!-- ============ 岗位覆盖 ============ -->
    <template v-else>
      <div class="card coverage">
        <div class="coverage-head">
          <h3>🛡️ 动态调度覆盖（第 {{ coverageDay || today }} 天 · 有效花名册 {{ coverageData.rosterCount }} 人）</h3>
          <div class="day-pills">
            <button v-for="d in days" :key="d" :class="{ on: (coverageDay || today) === d }" @click="selectCoverageDay(d)">{{ dayNames(d) }}</button>
          </div>
        </div>
        <div class="signal-bar">
          <span>📈 预约入园预测 <b>{{ forecastTotal }}</b> 人</span>
          <span>⛔ 当前阻断 <b>{{ activeCoverage.filter(w => w.level === 'block').length }}</b></span>
          <span>⚠️ 当前缺口 <b>{{ activeCoverage.length }}</b></span>
          <span>🕘 即将缺口 <b>{{ upcomingCoverage.length }}</b></span>
        </div>
        <div v-if="!coverageData.warnings.length" class="ok-box">✅ 预约客流、在途检修工单、待处置投诉与跨日夜班的岗位覆盖齐全。</div>
        <div v-if="activeCoverage.length" class="warn-section">
          <h4>当前小时缺岗</h4>
          <div v-for="w in activeCoverage" :key="w.demand_id" class="warn-item" :class="w.level">
            <b>{{ warnIcon(w.level) }} {{ w.msg }}</b>
            <span class="tag" :class="w.type">{{ demandMeta[w.type]?.label || '调度' }}</span>
          </div>
        </div>
        <div v-if="upcomingCoverage.length" class="warn-section">
          <h4>即将到来的调度需求</h4>
          <div v-for="w in upcomingCoverage" :key="w.demand_id" class="warn-item info">
            <b>{{ warnIcon(w.level) }} {{ w.msg }}</b>
            <span class="tag" :class="w.type">{{ demandMeta[w.type]?.label || '调度' }}</span>
          </div>
        </div>
        <p class="muted tips">调度规则：每小时读取未来三天入园/设施预约量，按区域客流补派保安、保洁与会员专员；在途检修优先补维修工；未结投诉按类别和 SLA 补匹配岗位；17 点后安全/晚间客流触发跨日夜班。当前缺口要求已打卡在岗，未来缺口只要求有效排班；调班审批会联动复核原班与目标班。</p>
      </div>
    </template>

    <!-- 时间线弹窗 -->
    <div class="modal-mask" v-if="detailLogs.length" @click.self="closeLogs">
      <div class="modal card">
        <h3>🕓 排班时间线 · {{ detailTitle }}</h3>
        <div class="tl" v-for="l in detailLogs" :key="l.id">
          <span class="tl-time muted">第{{ l.day }}天 {{ String(l.hour).padStart(2, '0') }}:00</span>
          <b>{{ ACTION_LABEL[l.action] || l.action }}</b>
          <em class="muted">{{ l.note }}</em>
          <span class="muted" v-if="l.staff_name">👤 {{ l.staff_name }}</span>
          <span class="muted" v-if="l.approver_name">🧑‍💼 {{ l.approver_name }}</span>
        </div>
        <div class="muted empty" v-if="!detailLogs.length">暂无记录</div>
        <button class="ghost" @click="closeLogs">关闭</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.sch { display: flex; flex-direction: column; gap: 14px; }
.stat-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(130px, 1fr)); gap: 12px; }
.stat { display: flex; flex-direction: column; gap: 2px; padding: 14px; }
.stat span { font-size: 20px; }
.stat b { font-size: 24px; }
.stat b.on { color: var(--green); }
.stat em { font-style: normal; color: var(--muted); font-size: 12px; }
.stat.alert { border-color: rgba(255,107,107,.5); }
.tabs { display: flex; gap: 6px; flex-wrap: wrap; align-items: center; margin-bottom: 12px; }
.card-tabs { padding: 10px 14px; background: none; border: none; }
.tabs button { position: relative; }
.tabs button.on { background: rgba(255,107,107,.18); border-color: rgba(255,107,107,.5); color: var(--accent); }
.badge-dot { font-style: normal; background: var(--accent); color: #fff; border-radius: 20px; font-size: 10px; padding: 0 6px; margin-left: 4px; }
.autofill { margin-left: auto; font-size: 12px; color: var(--muted); display: flex; align-items: center; gap: 6px; }

/* 排班看板 */
.board-wrap { overflow-x: auto; padding: 14px; }
.board { display: grid; grid-template-columns: 170px repeat(4, minmax(190px, 1fr)); gap: 8px; min-width: 960px; }
.bcol { min-height: 64px; }
.corner { display: flex; flex-direction: column; justify-content: center; gap: 6px; font-size: 12px; padding: 4px; }
.day-head { text-align: center; font-weight: 700; padding: 8px; border-radius: 10px; background: var(--panel2); border: 1px solid var(--border); }
.day-head.today { color: var(--accent); border-color: rgba(255,107,107,.5); }
.day-head em { display: block; font-style: normal; font-weight: 400; font-size: 11px; color: var(--muted); }
.staff-cell { border-bottom: 1px solid var(--border); }
.staff-cell em { font-style: normal; font-size: 11px; }
.shift-legend { display: flex; flex-wrap: wrap; gap: 6px; }
.shift-legend i { font-style: normal; font-size: 10px; color: #fff; padding: 1px 7px; border-radius: 10px; display: inline-flex; gap: 4px; align-items: center; }
.shift-legend i em { font-style: normal; opacity: .85; font-size: 9px; }
.shift-card { background: var(--panel2); border: 1px solid var(--border); border-left-width: 3px; border-radius: 10px; padding: 8px 10px; display: flex; flex-direction: column; gap: 4px; }
.shift-card.absent { border-color: var(--red) !important; background: rgba(255,107,107,.1); }
.shift-card.leave { opacity: .75; }
.sc-top { display: flex; justify-content: space-between; align-items: center; }
.sc-name { font-weight: 700; font-size: 13px; }
.sc-state { font-size: 11px; color: var(--muted); }
.sc-time { font-size: 11px; }
.sc-tags { display: flex; gap: 4px; flex-wrap: wrap; }
.sc-tags em { font-style: normal; font-size: 10px; background: rgba(102,166,255,.15); color: var(--blue); border-radius: 8px; padding: 0 6px; }
.sc-tags em.demand { background: rgba(167,139,250,.16); color: var(--purple); }
.sc-tags em.demand.traffic { background: rgba(102,166,255,.18); color: var(--blue); }
.sc-tags em.demand.maintenance { background: rgba(255,158,100,.16); color: var(--accent2); }
.sc-tags em.demand.complaint { background: rgba(255,107,107,.16); color: var(--red); }
.sc-tags em.demand.night { background: rgba(40,45,80,.25); color: var(--purple); }
.sc-tags em.money { background: rgba(109,213,160,.15); color: var(--green); }
.sc-actions { display: flex; gap: 4px; flex-wrap: wrap; margin-top: 2px; }
.sm { padding: 3px 8px; font-size: 11px; }
.inline-form { display: flex; flex-direction: column; gap: 5px; margin-top: 6px; padding-top: 6px; border-top: 1px dashed var(--border); }
.inline-form input, .inline-form select { font-size: 12px; padding: 5px 8px; }
.rest { font-size: 12px; opacity: .5; }

.assign-card { margin-top: 0; }
.assign-row { display: flex; gap: 10px; align-items: flex-end; flex-wrap: wrap; }
.assign-row label { display: flex; flex-direction: column; gap: 4px; font-size: 12px; color: var(--muted); }
.mmsg { font-size: 12px; color: var(--green); }
.mmsg.bad { color: var(--red); }
.tips { margin-top: 10px; font-size: 12px; line-height: 1.6; }

/* 考勤表 */
.atable { display: flex; flex-direction: column; }
.ahead, .arow { display: grid; grid-template-columns: .8fr 1fr 1.4fr .9fr .7fr 1fr .7fr .9fr .7fr .7fr; gap: 8px; padding: 10px 8px; font-size: 13px; align-items: center; }
.ahead { color: var(--muted); font-size: 12px; border-bottom: 1px solid var(--border); }
.arow { border-bottom: 1px solid var(--border); }
.arow em { display: block; font-style: normal; font-size: 11px; }
.mono { font-family: ui-monospace, monospace; font-size: 12px; color: var(--muted); }
.abadge { font-size: 11px; padding: 2px 8px; border-radius: 12px; border: 1px solid var(--border); }
.abadge.st-on { color: var(--green); border-color: rgba(109,213,160,.5); background: rgba(109,213,160,.1); }
.abadge.st-out { color: var(--muted); }
.abadge.st-absent { color: var(--red); border-color: rgba(255,107,107,.5); background: rgba(255,107,107,.1); }
.abadge.st-leave { color: var(--accent2); border-color: rgba(255,209,102,.4); }
.late-tag { font-style: normal; font-size: 10px; color: var(--accent2); margin-left: 4px; }
.dot-sh { display: inline-block; width: 8px; height: 8px; border-radius: 50%; margin-right: 5px; }
.pos { color: var(--green); } .neg { color: var(--red); }
.empty { padding: 18px; text-align: center; }

/* 申请 */
.reqlist { display: flex; flex-direction: column; gap: 10px; }
.card2 { background: var(--panel2); border: 1px solid var(--border); border-radius: 12px; padding: 12px 14px; }
.rq-head { display: flex; gap: 10px; align-items: center; margin-bottom: 8px; }
.rq-kind { font-weight: 700; font-size: 13px; }
.rq-kind.swap { color: var(--blue); } .rq-kind.overtime, .rq-kind { color: var(--purple); }
.rq-body { display: flex; gap: 8px; align-items: center; flex-wrap: wrap; font-size: 13px; }
.rq-body em { font-style: normal; font-size: 12px; }
.arrow { color: var(--muted); }
.rq-body .tag.ot { color: var(--accent2); border-color: rgba(255,209,102,.4); }
.reason { width: 100%; font-size: 12px; }
.rq-foot { display: flex; gap: 8px; margin-top: 8px; justify-content: flex-end; }

/* 覆盖 */
.coverage-head { display: flex; justify-content: space-between; gap: 12px; align-items: center; flex-wrap: wrap; margin-bottom: 12px; }
.coverage-head h3 { margin: 0; }
.day-pills { display: flex; gap: 6px; flex-wrap: wrap; }
.day-pills button { padding: 5px 10px; font-size: 12px; }
.day-pills button.on { background: rgba(255,107,107,.18); border-color: rgba(255,107,107,.5); color: var(--accent); }
.signal-bar { display: flex; gap: 10px; flex-wrap: wrap; margin-bottom: 12px; }
.signal-bar span { font-size: 12px; color: var(--muted); background: var(--panel2); border: 1px solid var(--border); border-radius: 16px; padding: 4px 10px; }
.signal-bar b { color: var(--text); }
.warn-section { margin-bottom: 12px; }
.warn-section h4 { margin: 0 0 8px; font-size: 13px; color: var(--muted); }
.coverage .warn-item { display: flex; justify-content: space-between; align-items: center; padding: 10px 12px; border-radius: 10px; margin-bottom: 8px; background: rgba(255,209,102,.08); border: 1px solid rgba(255,209,102,.3); gap: 12px; }
.coverage .warn-item.block { background: rgba(255,107,107,.1); border-color: rgba(255,107,107,.45); }
.coverage .warn-item.info { background: rgba(102,166,255,.08); border-color: rgba(102,166,255,.28); }
.coverage .tag.traffic { color: var(--blue); border-color: rgba(102,166,255,.35); }
.coverage .tag.maintenance { color: var(--accent2); border-color: rgba(255,209,102,.4); }
.coverage .tag.complaint { color: var(--red); border-color: rgba(255,107,107,.4); }
.coverage .tag.night { color: var(--purple); border-color: rgba(167,139,250,.4); }
.ok-box { padding: 16px; text-align: center; color: var(--green); background: rgba(109,213,160,.08); border: 1px solid rgba(109,213,160,.3); border-radius: 10px; }

/* 弹窗 */
.modal-mask { position: fixed; inset: 0; background: rgba(5,8,18,.7); display: flex; align-items: center; justify-content: center; z-index: 100; }
.modal { width: 560px; max-width: 92vw; max-height: 80vh; overflow-y: auto; }
.tl { display: flex; gap: 10px; align-items: baseline; flex-wrap: wrap; padding: 8px 4px; border-bottom: 1px dashed var(--border); font-size: 13px; }
.tl .tl-time { font-size: 11px; min-width: 74px; }
</style>
