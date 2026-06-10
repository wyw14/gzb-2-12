<template>
  <div class="matches">
    <div class="card">
      <h1 class="page-title">智能匹配</h1>

      <div class="filter-bar">
        <el-input v-model="filters.keyword" placeholder="搜索技能" clearable style="width: 200px">
          <template #prefix><el-icon><Search /></el-icon></template>
        </el-input>
        <el-select v-model="filters.category" placeholder="技能类别" clearable style="width: 160px">
          <el-option v-for="cat in categories" :key="cat.id" :label="cat.name" :value="cat.id" />
        </el-select>
        <el-slider v-model="filters.minScore" :min="0" :max="100" :step="10" style="width: 200px" />
        <span class="slider-label">最低契合度: {{ filters.minScore }}%</span>
        <el-button type="primary" @click="loadMatches">
          <el-icon><Refresh /></el-icon>刷新匹配
        </el-button>
      </div>

      <div v-if="matches.length" class="matches-list">
        <div v-for="match in filteredMatches" :key="match.userId" class="match-card">
          <div class="match-header">
            <el-avatar :src="match.user.avatar" :size="64" />
            <div class="match-user-info">
              <div class="match-name-row">
                <span class="match-name">{{ match.user.username }}</span>
                <span class="score-badge" :class="getScoreClass(match.score)">
                  {{ match.score }}% 契合
                </span>
              </div>
              <div class="match-rating">
                <el-rate :model-value="match.user.rating" disabled />
                <span class="rating-text">{{ match.user.rating }}</span>
                <span class="exchange-count">交换 {{ match.user.exchangeCount || 0 }} 次</span>
              </div>
              <div class="match-bio">{{ match.user.bio || '这个人很懒，什么都没写' }}</div>
            </div>
            <div class="match-score-circle" :class="getScoreClass(match.score)">
              <span class="score-number">{{ match.score }}</span>
              <span class="score-label">契合度</span>
            </div>
          </div>

          <div class="match-skills-section">
            <div class="skill-column">
              <h4 class="column-title">📖 你可以学到</h4>
              <div class="skills-tags">
                <span v-for="skill in match.matchedSkills.iCanLearn" :key="skill" class="skill-tag skill-learn">
                  {{ skill }}
                </span>
              </div>
            </div>
            <div class="skill-divider">
              <el-icon><Switch /></el-icon>
            </div>
            <div class="skill-column">
              <h4 class="column-title">🎓 你可以教授</h4>
              <div class="skills-tags">
                <span v-for="skill in match.matchedSkills.iCanTeach" :key="skill" class="skill-tag skill-teach">
                  {{ skill }}
                </span>
              </div>
            </div>
          </div>

          <div class="match-actions">
            <el-button @click="goToProfile(match.userId)">
              <el-icon><User /></el-icon>查看主页
            </el-button>
            <el-button type="primary" @click="goToChat(match.userId)">
              <el-icon><ChatDotRound /></el-icon>开始聊天
            </el-button>
            <el-button type="success" @click="createExchange(match)">
              <el-icon><Handshake /></el-icon>发起交换
            </el-button>
          </div>
        </div>
      </div>

      <el-empty v-else description="暂无匹配，先去发布你的技能吧">
        <template #action>
          <el-button type="primary" @click="$router.push('/publish')">发布技能</el-button>
        </template>
      </el-empty>
    </div>

    <el-dialog v-model="showExchangeDialog" title="发起技能交换" width="560px">
      <div v-if="currentMatch" class="exchange-dialog-content">
        <div class="exchange-user-info">
          <el-avatar :src="currentMatch.user.avatar" :size="56" />
          <div class="exchange-user-detail">
            <div class="exchange-username">{{ currentMatch.user.username }}</div>
            <div class="exchange-score">契合度 {{ currentMatch.score }}%</div>
          </div>
        </div>

        <el-divider />

        <div class="exchange-skills-preview">
          <div class="skill-preview-col">
            <div class="skill-preview-label">你将教授</div>
            <div class="skill-preview-tags">
              <span v-for="s in currentMatch.matchedSkills.iCanTeach" :key="s" class="skill-tag skill-teach">{{ s }}</span>
            </div>
          </div>
          <div class="skill-preview-col">
            <div class="skill-preview-label">你将学习</div>
            <div class="skill-preview-tags">
              <span v-for="s in currentMatch.matchedSkills.iCanLearn" :key="s" class="skill-tag skill-learn">{{ s }}</span>
            </div>
          </div>
        </div>

        <el-form label-position="top" style="margin-top: 20px">
          <el-form-item label="约定交换时间">
            <div class="time-selector">
              <el-date-picker
                v-model="exchangeForm.startTime"
                type="datetime"
                placeholder="选择开始时间"
                style="width: 100%"
                @change="onTimeChange"
              />
              <span class="time-to">至</span>
              <el-date-picker
                v-model="exchangeForm.endTime"
                type="datetime"
                placeholder="选择结束时间"
                style="width: 100%"
                @change="onTimeChange"
              />
            </div>
            <div class="time-hint">可选：不填时间则保存为待协商状态</div>
          </el-form-item>
        </el-form>

        <div v-if="conflicts.length > 0" class="conflict-section">
          <div class="conflict-title">
            <el-icon color="#f56c6c"><Warning /></el-icon>
            <span>时间冲突</span>
          </div>
          <div v-for="(conflict, index) in conflicts" :key="index" class="conflict-item">
            <el-avatar :src="conflict.avatar" :size="36" />
            <div class="conflict-info">
              <div class="conflict-name">
                {{ conflict.username }}
                <span class="conflict-side">{{ conflict.side === 'me' ? '(你的日程)' : '(对方日程)' }}</span>
              </div>
              <div class="conflict-time">
                {{ formatDateTime(conflict.startTime) }} - {{ formatDateTime(conflict.endTime) }}
              </div>
            </div>
          </div>
        </div>
      </div>
      <template #footer>
        <el-button @click="showExchangeDialog = false">取消</el-button>
        <el-button @click="saveAsNegotiating">保存为待协商</el-button>
        <el-button type="primary" @click="confirmCreateExchange" :loading="submitting" :disabled="conflicts.length > 0">
          确认发起
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import { matchAPI, skillAPI, exchangeAPI } from '../api'
import { ElMessage } from 'element-plus'
import dayjs from 'dayjs'
import { Search, Refresh, User, ChatDotRound, Switch, Handshake, Warning } from '@element-plus/icons-vue'

const router = useRouter()
const matches = ref([])
const categories = ref([])
const filters = ref({
  keyword: '',
  category: '',
  minScore: 30
})
const showExchangeDialog = ref(false)
const currentMatch = ref(null)
const submitting = ref(false)
const conflicts = ref([])
const exchangeForm = ref({
  startTime: null,
  endTime: null
})

const filteredMatches = computed(() => {
  let result = matches.value
  if (filters.value.keyword) {
    const keyword = filters.value.keyword.toLowerCase()
    result = result.filter(m =>
      m.matchedSkills.iCanLearn.some(s => s.toLowerCase().includes(keyword)) ||
      m.matchedSkills.iCanTeach.some(s => s.toLowerCase().includes(keyword)) ||
      m.user.username.toLowerCase().includes(keyword)
    )
  }
  return result
})

onMounted(async () => {
  await loadCategories()
  await loadMatches()
})

async function loadCategories() {
  const res = await skillAPI.getCategories()
  categories.value = res.data
}

async function loadMatches() {
  try {
    const params = { minScore: filters.value.minScore }
    if (filters.value.category) params.category = filters.value.category
    const res = await matchAPI.getMatches(params)
    matches.value = res.data
  } catch (e) {
    ElMessage.error('加载匹配失败')
  }
}

function getScoreClass(score) {
  if (score >= 70) return 'score-high'
  if (score >= 40) return 'score-medium'
  return 'score-low'
}

function goToProfile(userId) {
  router.push(`/profile/${userId}`)
}

function goToChat(userId) {
  router.push(`/chat/${userId}`)
}

function createExchange(match) {
  currentMatch.value = match
  exchangeForm.value = {
    startTime: null,
    endTime: null
  }
  conflicts.value = []
  showExchangeDialog.value = true
}

function formatDateTime(time) {
  return dayjs(time).format('YYYY-MM-DD HH:mm')
}

async function onTimeChange() {
  if (!exchangeForm.value.startTime || !exchangeForm.value.endTime) {
    conflicts.value = []
    return
  }

  const start = new Date(exchangeForm.value.startTime).getTime()
  const end = new Date(exchangeForm.value.endTime).getTime()

  if (start >= end) {
    ElMessage.warning('结束时间必须晚于开始时间')
    return
  }

  try {
    const res = await exchangeAPI.checkConflict({
      partnerId: currentMatch.value.userId,
      startTime: exchangeForm.value.startTime,
      endTime: exchangeForm.value.endTime
    })
    conflicts.value = res.data.conflicts || []
  } catch (e) {
    conflicts.value = []
  }
}

async function confirmCreateExchange() {
  if (!currentMatch.value) return

  const hasTime = exchangeForm.value.startTime && exchangeForm.value.endTime
  if (hasTime && conflicts.value.length > 0) {
    ElMessage.warning('时间存在冲突，请调整时间或保存为待协商')
    return
  }

  try {
    submitting.value = true
    const data = {
      partnerId: currentMatch.value.userId,
      skills: {
        teach: currentMatch.value.matchedSkills.iCanTeach,
        learn: currentMatch.value.matchedSkills.iCanLearn
      }
    }

    if (hasTime) {
      data.schedule = {
        startTime: exchangeForm.value.startTime,
        endTime: exchangeForm.value.endTime,
        status: 'negotiating'
      }
    }

    await exchangeAPI.createExchange(data)
    ElMessage.success('交换请求已发送')
    showExchangeDialog.value = false
  } catch (e) {
    if (e.response?.status === 409) {
      conflicts.value = e.response.data.conflicts || []
      ElMessage.warning('时间存在冲突')
    } else {
      ElMessage.error(e.message || '发起交换失败')
    }
  } finally {
    submitting.value = false
  }
}

async function saveAsNegotiating() {
  if (!currentMatch.value) return

  try {
    submitting.value = true
    const data = {
      partnerId: currentMatch.value.userId,
      skills: {
        teach: currentMatch.value.matchedSkills.iCanTeach,
        learn: currentMatch.value.matchedSkills.iCanLearn
      },
      schedule: {
        startTime: exchangeForm.value.startTime || null,
        endTime: exchangeForm.value.endTime || null,
        status: 'negotiating'
      }
    }

    await exchangeAPI.createExchange(data)
    ElMessage.success('已保存为待协商状态')
    showExchangeDialog.value = false
  } catch (e) {
    ElMessage.error(e.message || '保存失败')
  } finally {
    submitting.value = false
  }
}
</script>

<style scoped>
.matches {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.filter-bar {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 32px;
  flex-wrap: wrap;
}

.slider-label {
  font-size: 14px;
  color: #666;
}

.matches-list {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.match-card {
  background: #fafafa;
  border-radius: 16px;
  padding: 24px;
  transition: all 0.3s;
  border: 2px solid transparent;
}

.match-card:hover {
  border-color: #667eea;
  box-shadow: 0 8px 32px rgba(102, 126, 234, 0.15);
}

.match-header {
  display: flex;
  align-items: center;
  gap: 20px;
  margin-bottom: 24px;
}

.match-user-info {
  flex: 1;
}

.match-name-row {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
}

.match-name {
  font-size: 20px;
  font-weight: 700;
  color: #333;
}

.match-rating {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.rating-text {
  font-weight: 600;
  color: #ff9800;
}

.exchange-count {
  color: #999;
  font-size: 13px;
}

.match-bio {
  color: #666;
  font-size: 14px;
}

.match-score-circle {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: white;
}

.score-number {
  font-size: 32px;
  font-weight: 700;
}

.score-label {
  font-size: 12px;
}

.match-skills-section {
  display: flex;
  align-items: center;
  gap: 20px;
  padding: 20px;
  background: white;
  border-radius: 12px;
  margin-bottom: 20px;
}

.skill-column {
  flex: 1;
}

.column-title {
  font-size: 14px;
  font-weight: 600;
  color: #333;
  margin-bottom: 12px;
}

.skills-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.skill-divider {
  font-size: 32px;
  color: #667eea;
}

.match-actions {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
}

.exchange-dialog-content {
  padding: 0 10px;
}

.exchange-user-info {
  display: flex;
  align-items: center;
  gap: 16px;
}

.exchange-user-detail {
  flex: 1;
}

.exchange-username {
  font-size: 18px;
  font-weight: 600;
  color: #333;
  margin-bottom: 4px;
}

.exchange-score {
  font-size: 14px;
  color: #667eea;
  font-weight: 500;
}

.exchange-skills-preview {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
}

.skill-preview-col {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.skill-preview-label {
  font-size: 13px;
  font-weight: 500;
  color: #666;
}

.skill-preview-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
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

.time-hint {
  margin-top: 8px;
  font-size: 12px;
  color: #999;
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
</style>
