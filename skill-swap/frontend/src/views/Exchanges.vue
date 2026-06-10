<template>
  <div class="exchanges">
    <div class="card">
      <h1 class="page-title">交换记录</h1>

      <div class="exchange-tabs">
        <div class="tab" :class="{ active: activeTab === 'pending' }" @click="activeTab = 'pending'">
          待确认 ({{ pendingExchanges.length }})
        </div>
        <div class="tab" :class="{ active: activeTab === 'negotiating' }" @click="activeTab = 'negotiating'">
          待协商 ({{ negotiatingExchanges.length }})
        </div>
        <div class="tab" :class="{ active: activeTab === 'completed' }" @click="activeTab = 'completed'">
          已完成 ({{ completedExchanges.length }})
        </div>
      </div>

      <div v-if="activeTab === 'pending'" class="exchange-list">
        <div v-for="exchange in pendingExchanges" :key="exchange.id" class="exchange-card">
          <div class="exchange-header">
            <div class="exchange-users">
              <el-avatar :src="getUserAvatar(exchange.initiatorId)" :size="40" />
              <el-icon class="exchange-icon"><Switch /></el-icon>
              <el-avatar :src="getUserAvatar(exchange.partnerId)" :size="40" />
            </div>
            <div class="exchange-status pending">
              待确认 {{ exchange.confirmedBy.length }}/2
            </div>
          </div>

          <div class="exchange-skills">
            <div class="skill-col">
              <span class="label">你将教授:</span>
              <span v-for="skill in getMyTeachSkills(exchange)" :key="skill" class="skill-tag skill-teach">{{ skill }}</span>
            </div>
            <div class="skill-col">
              <span class="label">你将学习:</span>
              <span v-for="skill in getMyLearnSkills(exchange)" :key="skill" class="skill-tag skill-learn">{{ skill }}</span>
            </div>
          </div>

          <div v-if="exchange.schedule" class="exchange-schedule">
            <el-icon><Clock /></el-icon>
            <span class="schedule-label">
              {{ getScheduleStatusText(exchange.schedule) }}
            </span>
            <span v-if="exchange.schedule.startTime && exchange.schedule.endTime" class="schedule-time">
              {{ formatDateTime(exchange.schedule.startTime) }} - {{ formatDateTime(exchange.schedule.endTime) }}
            </span>
            <el-button v-if="exchange.schedule.status === 'negotiating' && exchange.schedule.proposedBy !== myId"
              type="primary" link size="small" @click="confirmSchedule(exchange)">
              确认时间
            </el-button>
          </div>

          <div class="exchange-footer">
            <span class="exchange-time">{{ formatTime(exchange.createdAt) }}</span>
            <div class="exchange-actions">
              <el-button type="primary" @click="confirmExchange(exchange)" :disabled="exchange.confirmedBy.includes(myId)" :loading="confirmingId === exchange.id">
                <el-icon><Check /></el-icon>
                {{ exchange.confirmedBy.includes(myId) ? '已确认' : '确认完成' }}
              </el-button>
              <el-button @click="goToChat(exchange)">
                <el-icon><ChatDotRound /></el-icon>联系对方
              </el-button>
            </div>
          </div>
        </div>
        <el-empty v-if="pendingExchanges.length === 0" description="暂无待确认的交换" />
      </div>

      <div v-if="activeTab === 'negotiating'" class="exchange-list">
        <div v-for="exchange in negotiatingExchanges" :key="exchange.id" class="exchange-card">
          <div class="exchange-header">
            <div class="exchange-users">
              <el-avatar :src="getUserAvatar(exchange.initiatorId)" :size="40" />
              <el-icon class="exchange-icon"><Switch /></el-icon>
              <el-avatar :src="getUserAvatar(exchange.partnerId)" :size="40" />
            </div>
            <div class="exchange-status negotiating">
              ⏳ 待协商
            </div>
          </div>

          <div class="exchange-skills">
            <div class="skill-col">
              <span class="label">你将教授:</span>
              <span v-for="skill in getMyTeachSkills(exchange)" :key="skill" class="skill-tag skill-teach">{{ skill }}</span>
            </div>
            <div class="skill-col">
              <span class="label">你将学习:</span>
              <span v-for="skill in getMyLearnSkills(exchange)" :key="skill" class="skill-tag skill-learn">{{ skill }}</span>
            </div>
          </div>

          <div class="exchange-schedule negotiating-schedule">
            <el-icon><Clock /></el-icon>
            <span class="schedule-label">
              {{ getScheduleStatusText(exchange.schedule) }}
            </span>
            <span v-if="exchange.schedule?.startTime && exchange.schedule?.endTime" class="schedule-time">
              {{ formatDateTime(exchange.schedule.startTime) }} - {{ formatDateTime(exchange.schedule.endTime) }}
            </span>
            <span v-else class="schedule-time schedule-time-empty">
              暂未约定时间
            </span>
          </div>

          <div class="exchange-footer">
            <span class="exchange-time">{{ formatTime(exchange.createdAt) }}</span>
            <div class="exchange-actions">
              <el-button type="primary" @click="showScheduleDialog(exchange)">
                <el-icon><Edit /></el-icon>约定时间
              </el-button>
              <el-button @click="goToChat(exchange)">
                <el-icon><ChatDotRound /></el-icon>联系对方
              </el-button>
            </div>
          </div>
        </div>
        <el-empty v-if="negotiatingExchanges.length === 0" description="暂无待协商的交换" />
      </div>

      <div v-if="activeTab === 'completed'" class="exchange-list">
        <div v-for="exchange in completedExchanges" :key="exchange.id" class="exchange-card">
          <div class="exchange-header">
            <div class="exchange-users">
              <el-avatar :src="getUserAvatar(exchange.initiatorId)" :size="40" />
              <el-icon class="exchange-icon"><Switch /></el-icon>
              <el-avatar :src="getUserAvatar(exchange.partnerId)" :size="40" />
            </div>
            <div class="exchange-status completed">
              ✅ 已完成
            </div>
          </div>

          <div class="exchange-skills">
            <div class="skill-col">
              <span class="label">教授:</span>
              <span v-for="skill in getMyTeachSkills(exchange)" :key="skill" class="skill-tag skill-teach">{{ skill }}</span>
            </div>
            <div class="skill-col">
              <span class="label">学习:</span>
              <span v-for="skill in getMyLearnSkills(exchange)" :key="skill" class="skill-tag skill-learn">{{ skill }}</span>
            </div>
          </div>

          <div class="exchange-footer">
            <span class="exchange-time">完成于 {{ formatTime(exchange.completedAt) }}</span>
            <div class="exchange-actions">
              <el-button type="success" @click="showReviewDialog(exchange)" :disabled="hasReviewed(exchange)">
                <el-icon><Star /></el-icon>
                {{ hasReviewed(exchange) ? '已评价' : '去评价' }}
              </el-button>
            </div>
          </div>
        </div>
        <el-empty v-if="completedExchanges.length === 0" description="暂无已完成的交换" />
      </div>
    </div>

    <el-dialog v-model="showScheduleDialog" title="约定交换时间" width="500px">
      <div v-if="currentExchange" class="schedule-dialog">
        <el-form label-position="top">
          <el-form-item label="交换时间">
            <div class="time-selector">
              <el-date-picker
                v-model="scheduleForm.startTime"
                type="datetime"
                placeholder="选择开始时间"
                style="width: 100%"
                @change="onScheduleTimeChange"
              />
              <span class="time-to">至</span>
              <el-date-picker
                v-model="scheduleForm.endTime"
                type="datetime"
                placeholder="选择结束时间"
                style="width: 100%"
                @change="onScheduleTimeChange"
              />
            </div>
          </el-form-item>
        </el-form>

        <div v-if="scheduleConflicts.length > 0" class="conflict-section">
          <div class="conflict-title">
            <el-icon color="#f56c6c"><Warning /></el-icon>
            <span>时间冲突</span>
          </div>
          <div v-for="(conflict, index) in scheduleConflicts" :key="index" class="conflict-item">
            <el-avatar :src="conflict.avatar" :size="36" />
            <div class="conflict-info">
              <div class="conflict-name">
                {{ conflict.username }}
                <span class="conflict-side">{{ conflict.side === 'me' || conflict.side === 'initiator' ? '(你的日程)' : '(对方日程)' }}</span>
              </div>
              <div class="conflict-time">
                {{ formatDateTime(conflict.startTime) }} - {{ formatDateTime(conflict.endTime) }}
              </div>
            </div>
          </div>
        </div>
      </div>
      <template #footer>
        <el-button @click="showScheduleDialog = false">取消</el-button>
        <el-button type="primary" @click="submitSchedule" :loading="submittingSchedule" :disabled="scheduleConflicts.length > 0">
          确认时间
        </el-button>
      </template>
    </el-dialog>

    <el-dialog v-model="showReview" title="评价这次交换" width="500px">
      <div v-if="currentExchange" class="review-form">
        <div class="review-user">
          <el-avatar :src="getUserAvatar(getOtherUserId(currentExchange))" :size="56" />
          <span class="username">{{ getUserName(getOtherUserId(currentExchange)) }}</span>
        </div>
        <el-form :model="reviewForm" label-position="top">
          <el-form-item label="评分">
            <el-rate v-model="reviewForm.rating" size="large" />
          </el-form-item>
          <el-form-item label="评价内容">
            <el-input v-model="reviewForm.comment" type="textarea" :rows="4" placeholder="分享一下这次交换的体验..." maxlength="500" show-word-limit />
          </el-form-item>
        </el-form>
      </div>
      <template #footer>
        <el-button @click="showReview = false">取消</el-button>
        <el-button type="primary" @click="submitReview" :loading="submitting">提交评价</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import { exchangeAPI, reviewAPI, authAPI } from '../api'
import { useUserStore } from '../stores/user'
import { ElMessage } from 'element-plus'
import dayjs from 'dayjs'
import { Switch, Check, ChatDotRound, Star, Clock, Edit, Warning } from '@element-plus/icons-vue'

const router = useRouter()
const userStore = useUserStore()
const myId = userStore.user?.id

const exchanges = ref([])
const users = ref({})
const activeTab = ref('pending')
const confirmingId = ref(null)
const showReview = ref(false)
const currentExchange = ref(null)
const submitting = ref(false)
const reviewForm = ref({ rating: 5, comment: '' })
const showScheduleDialog = ref(false)
const submittingSchedule = ref(false)
const scheduleConflicts = ref([])
const scheduleForm = ref({
  startTime: null,
  endTime: null
})

const pendingExchanges = computed(() =>
  exchanges.value.filter(e => e.status === 'pending' && (!e.schedule || e.schedule.status !== 'negotiating'))
)

const negotiatingExchanges = computed(() =>
  exchanges.value.filter(e =>
    e.status === 'pending' && e.schedule?.status === 'negotiating'
  )
)

const completedExchanges = computed(() =>
  exchanges.value.filter(e => e.status === 'completed')
)

onMounted(async () => {
  await loadExchanges()
})

async function loadExchanges() {
  const res = await exchangeAPI.getExchanges()
  exchanges.value = res.data

  const userIds = new Set()
  exchanges.value.forEach(e => {
    userIds.add(e.initiatorId)
    userIds.add(e.partnerId)
  })

  for (const id of userIds) {
    try {
      const userRes = await authAPI.getUser(id)
      users.value[id] = userRes.data
    } catch (e) {}
  }
}

function getUserAvatar(id) {
  return users.value[id]?.avatar || ''
}

function getUserName(id) {
  return users.value[id]?.username || '未知用户'
}

function getOtherUserId(exchange) {
  return exchange.initiatorId === myId ? exchange.partnerId : exchange.initiatorId
}

function getMyTeachSkills(exchange) {
  if (exchange.initiatorId === myId) {
    return exchange.skills?.teach || []
  } else {
    return exchange.skills?.learn || []
  }
}

function getMyLearnSkills(exchange) {
  if (exchange.initiatorId === myId) {
    return exchange.skills?.learn || []
  } else {
    return exchange.skills?.teach || []
  }
}

function formatTime(time) {
  return dayjs(time).format('YYYY-MM-DD HH:mm')
}

function formatDateTime(time) {
  return dayjs(time).format('YYYY-MM-DD HH:mm')
}

function getScheduleStatusText(schedule) {
  if (!schedule) return '待约定时间'
  if (schedule.status === 'confirmed') return '已约定时间'
  if (schedule.status === 'negotiating') {
    if (schedule.proposedBy === myId) {
      return '等待对方确认时间'
    }
    return '对方提议时间，待你确认'
  }
  return '待约定时间'
}

async function confirmExchange(exchange) {
  try {
    confirmingId.value = exchange.id
    await exchangeAPI.confirmExchange(exchange.id)
    ElMessage.success('确认成功')
    await loadExchanges()
  } catch (e) {
    ElMessage.error('确认失败')
  } finally {
    confirmingId.value = null
  }
}

function goToChat(exchange) {
  const otherId = getOtherUserId(exchange)
  router.push(`/chat/${otherId}`)
}

function hasReviewed(exchange) {
  return exchange.reviewedBy?.includes(myId)
}

function showReviewDialog(exchange) {
  currentExchange.value = exchange
  reviewForm.value = { rating: 5, comment: '' }
  showReview.value = true
}

async function submitReview() {
  if (!currentExchange.value) return
  if (!reviewForm.value.rating) {
    ElMessage.warning('请选择评分')
    return
  }

  try {
    submitting.value = true
    await reviewAPI.createReview({
      exchangeId: currentExchange.value.id,
      targetUserId: getOtherUserId(currentExchange.value),
      rating: reviewForm.value.rating,
      comment: reviewForm.value.comment
    })

    if (!currentExchange.value.reviewedBy) {
      currentExchange.value.reviewedBy = []
    }
    currentExchange.value.reviewedBy.push(myId)

    ElMessage.success('评价成功')
    showReview.value = false
  } catch (e) {
    ElMessage.error(e.message || '评价失败')
  } finally {
    submitting.value = false
  }
}

function showScheduleDialog(exchange) {
  currentExchange.value = exchange
  scheduleForm.value = {
    startTime: exchange.schedule?.startTime || null,
    endTime: exchange.schedule?.endTime || null
  }
  scheduleConflicts.value = []
  showScheduleDialog.value = true
}

async function onScheduleTimeChange() {
  if (!scheduleForm.value.startTime || !scheduleForm.value.endTime || !currentExchange.value) {
    scheduleConflicts.value = []
    return
  }

  const start = new Date(scheduleForm.value.startTime).getTime()
  const end = new Date(scheduleForm.value.endTime).getTime()

  if (start >= end) {
    ElMessage.warning('结束时间必须晚于开始时间')
    return
  }

  const partnerId = getOtherUserId(currentExchange.value)
  try {
    const res = await exchangeAPI.checkConflict({
      partnerId,
      startTime: scheduleForm.value.startTime,
      endTime: scheduleForm.value.endTime
    })
    scheduleConflicts.value = res.data.conflicts || []
  } catch (e) {
    scheduleConflicts.value = []
  }
}

async function submitSchedule() {
  if (!currentExchange.value || !scheduleForm.value.startTime || !scheduleForm.value.endTime) {
    ElMessage.warning('请选择开始和结束时间')
    return
  }

  if (scheduleConflicts.value.length > 0) {
    ElMessage.warning('时间存在冲突，请调整时间')
    return
  }

  try {
    submittingSchedule.value = true
    await exchangeAPI.updateSchedule(currentExchange.value.id, {
      startTime: scheduleForm.value.startTime,
      endTime: scheduleForm.value.endTime,
      status: 'negotiating'
    })
    ElMessage.success('时间已提议，等待对方确认')
    showScheduleDialog.value = false
    await loadExchanges()
  } catch (e) {
    if (e.response?.status === 409) {
      scheduleConflicts.value = e.response.data.conflicts || []
      ElMessage.warning('时间存在冲突')
    } else {
      ElMessage.error(e.message || '提交失败')
    }
  } finally {
    submittingSchedule.value = false
  }
}

async function confirmSchedule(exchange) {
  try {
    await exchangeAPI.confirmSchedule(exchange.id)
    ElMessage.success('已确认时间')
    await loadExchanges()
  } catch (e) {
    ElMessage.error(e.message || '确认失败')
  }
}
</script>

<style scoped>
.exchanges {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.exchange-tabs {
  display: flex;
  gap: 16px;
  margin-bottom: 24px;
  border-bottom: 2px solid #eee;
}

.tab {
  padding: 12px 24px;
  cursor: pointer;
  font-weight: 500;
  color: #999;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  transition: all 0.2s;
}

.tab.active {
  color: #667eea;
  border-bottom-color: #667eea;
}

.exchange-list {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.exchange-card {
  background: #fafafa;
  border-radius: 16px;
  padding: 24px;
  transition: all 0.3s;
  border: 2px solid transparent;
}

.exchange-card:hover {
  border-color: #667eea;
}

.exchange-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.exchange-users {
  display: flex;
  align-items: center;
  gap: 12px;
}

.exchange-icon {
  font-size: 24px;
  color: #667eea;
}

.exchange-status {
  padding: 6px 16px;
  border-radius: 20px;
  font-weight: 600;
  font-size: 14px;
}

.exchange-status.pending {
  background: #fff7e6;
  color: #fa8c16;
}

.exchange-status.completed {
  background: #f6ffed;
  color: #52c41a;
}

.exchange-skills {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
  margin-bottom: 20px;
  padding: 20px;
  background: white;
  border-radius: 12px;
}

.skill-col {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.skill-col .label {
  font-size: 13px;
  color: #666;
  font-weight: 500;
}

.exchange-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.exchange-time {
  color: #999;
  font-size: 13px;
}

.exchange-actions {
  display: flex;
  gap: 12px;
}

.review-form {
  text-align: center;
}

.review-user {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  margin-bottom: 24px;
}

.review-user .username {
  font-weight: 600;
  font-size: 18px;
  color: #333;
}

.exchange-status.negotiating {
  background: #e6f7ff;
  color: #1890ff;
}

.exchange-schedule {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 16px;
  background: #f0f5ff;
  border-radius: 8px;
  margin-bottom: 16px;
  font-size: 13px;
}

.exchange-schedule .schedule-label {
  font-weight: 500;
  color: #667eea;
}

.exchange-schedule .schedule-time {
  color: #666;
  margin-left: 4px;
}

.exchange-schedule .schedule-time-empty {
  color: #999;
  font-style: italic;
}

.negotiating-schedule {
  background: #e6f7ff;
}

.negotiating-schedule .schedule-label {
  color: #1890ff;
}

.time-selector {
  display: flex;
  align-items: center;
  gap: 12px;
}

.time-to {
  color: #999;
  font-size: 14px;
  white-space: nowrap;
}

.conflict-section {
  background: #fff1f0;
  border: 1px solid #ffa39e;
  border-radius: 8px;
  padding: 16px;
  margin-top: 8px;
}

.conflict-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 600;
  color: #f5222d;
  margin-bottom: 12px;
  font-size: 14px;
}

.conflict-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px;
  background: white;
  border-radius: 8px;
  margin-bottom: 8px;
}

.conflict-item:last-child {
  margin-bottom: 0;
}

.conflict-info {
  flex: 1;
}

.conflict-name {
  font-weight: 500;
  color: #333;
  font-size: 14px;
  margin-bottom: 4px;
}

.conflict-side {
  color: #999;
  font-size: 12px;
  margin-left: 4px;
}

.conflict-time {
  font-size: 13px;
  color: #666;
}

.schedule-dialog {
  padding: 0 10px;
}
</style>
