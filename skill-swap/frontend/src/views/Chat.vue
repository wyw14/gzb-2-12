<template>
  <div class="chat-page">
    <div class="chat-layout">
      <div class="sidebar">
        <div class="sidebar-header">
          <h3>消息列表</h3>
        </div>
        <div class="conversation-list">
          <div
            v-for="conv in conversations"
            :key="conv.userId"
            class="conversation-item"
            :class="{ active: currentUserId === conv.userId }"
            @click="selectConversation(conv.userId)"
          >
            <el-avatar :src="conv.avatar" :size="48" />
            <div class="conv-info">
              <div class="conv-header">
                <span class="conv-name">{{ conv.username }}</span>
                <span class="conv-time">{{ formatTime(conv.lastMessageTime) }}</span>
              </div>
              <div class="conv-message">
                <span class="last-msg">{{ conv.lastMessage }}</span>
                <el-badge v-if="conv.unreadCount > 0" :value="conv.unreadCount" class="unread-badge" />
              </div>
            </div>
          </div>
          <el-empty v-if="conversations.length === 0" description="暂无消息" />
        </div>
      </div>

      <div class="chat-main">
        <div v-if="currentUser" class="chat-header">
          <el-avatar :src="currentUser.avatar" :size="40" />
          <span class="chat-username">{{ currentUser.username }}</span>
          <div class="header-actions">
            <el-button type="primary" size="small" @click="showExchangeDialog = true">
              <el-icon><Handshake /></el-icon>发起交换
            </el-button>
          </div>
        </div>

        <div v-if="currentUserId" class="messages-container" ref="messagesContainer">
          <div v-for="msg in messages" :key="msg.id" class="message-row" :class="msg.senderId === myId ? 'sent' : 'received'">
            <el-avatar v-if="msg.senderId !== myId" :src="currentUser?.avatar" :size="32" />
            <div class="chat-bubble" :class="msg.senderId === myId ? 'sent' : 'received'">
              {{ msg.content }}
            </div>
          </div>
          <el-empty v-if="messages.length === 0" description="开始你们的对话吧" />
        </div>

        <div v-else class="chat-empty">
          <el-icon size="64"><ChatDotRound /></el-icon>
          <p>选择一个会话开始聊天</p>
        </div>

        <div v-if="currentUserId" class="chat-input-area">
          <el-input
            v-model="messageInput"
            placeholder="输入消息..."
            size="large"
            @keyup.enter="sendMessage"
          >
            <template #append>
              <el-button type="primary" @click="sendMessage" :loading="sending">
                <el-icon><Promotion /></el-icon>发送
              </el-button>
            </template>
          </el-input>
        </div>
      </div>
    </div>

    <el-dialog v-model="showExchangeDialog" title="发起技能交换" width="560px">
      <el-form :model="exchangeForm" label-position="top">
        <el-form-item label="你想教授的技能">
          <el-select v-model="exchangeForm.teachSkill" placeholder="选择你可以教的技能" style="width: 100%">
            <el-option v-for="s in myTeachSkills" :key="s.id" :label="s.name" :value="s.name" />
          </el-select>
        </el-form-item>
        <el-form-item label="你想学习的技能">
          <el-select v-model="exchangeForm.learnSkill" placeholder="选择你想要学习的技能" style="width: 100%">
            <el-option v-for="s in otherTeachSkills" :key="s.id" :label="s.name" :value="s.name" />
          </el-select>
        </el-form-item>
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
                <span :class="['conflict-status-tag', conflict.scheduleStatus === 'confirmed' ? 'status-confirmed' : 'status-negotiating']">
                  {{ conflict.scheduleStatus === 'confirmed' ? '已确认' : '协商中' }}
                </span>
              </div>
              <div class="conflict-time">
                {{ formatDateTime(conflict.startTime) }} - {{ formatDateTime(conflict.endTime) }}
              </div>
            </div>
          </div>
        </div>
      </el-form>
      <template #footer>
        <el-button @click="showExchangeDialog = false">取消</el-button>
        <el-button @click="saveAsNegotiating">保存为待协商</el-button>
        <el-button type="primary" @click="createExchange" :loading="submitting" :disabled="conflicts.length > 0">
          确认发起
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { messageAPI, skillAPI, exchangeAPI, authAPI } from '../api'
import { useUserStore } from '../stores/user'
import { ElMessage } from 'element-plus'
import dayjs from 'dayjs'
import { ChatDotRound, Promotion, Handshake, Warning } from '@element-plus/icons-vue'

const route = useRoute()
const router = useRouter()
const userStore = useUserStore()
const myId = userStore.user?.id

const conversations = ref([])
const messages = ref([])
const currentUserId = ref(null)
const currentUser = ref(null)
const messageInput = ref('')
const sending = ref(false)
const messagesContainer = ref(null)
const showExchangeDialog = ref(false)
const myTeachSkills = ref([])
const otherTeachSkills = ref([])
const exchangeForm = ref({
  teachSkill: '',
  learnSkill: '',
  startTime: null,
  endTime: null
})
const conflicts = ref([])
const checkingConflict = ref(false)

onMounted(async () => {
  await loadConversations()
  if (route.params.userId) {
    await selectConversation(route.params.userId)
  }
})

watch(() => route.params.userId, async (newId) => {
  if (newId) {
    await selectConversation(newId)
  }
})

async function loadConversations() {
  try {
    const res = await messageAPI.getConversations()
    conversations.value = res.data
  } catch (e) {}
}

async function selectConversation(userId) {
  currentUserId.value = userId
  await loadMessages(userId)
  await loadConversations()

  const userRes = await authAPI.getUser(userId)
  currentUser.value = userRes.data

  const [mySkills, otherSkills] = await Promise.all([
    skillAPI.getSkills({ userId: myId, type: 'teach' }),
    skillAPI.getSkills({ userId, type: 'teach' })
  ])
  myTeachSkills.value = mySkills.data
  otherTeachSkills.value = otherSkills.data
}

async function loadMessages(userId) {
  try {
    const res = await messageAPI.getMessages(userId)
    messages.value = res.data
    await nextTick()
    scrollToBottom()
  } catch (e) {}
}

async function sendMessage() {
  if (!messageInput.value.trim()) return
  try {
    sending.value = true
    await messageAPI.sendMessage({
      receiverId: currentUserId.value,
      content: messageInput.value.trim()
    })
    messageInput.value = ''
    await loadMessages(currentUserId.value)
    await loadConversations()
  } catch (e) {
    ElMessage.error('发送失败')
  } finally {
    sending.value = false
  }
}

function scrollToBottom() {
  if (messagesContainer.value) {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  }
}

function formatTime(time) {
  const now = dayjs()
  const msgTime = dayjs(time)
  if (now.diff(msgTime, 'day') === 0) {
    return msgTime.format('HH:mm')
  } else if (now.diff(msgTime, 'day') === 1) {
    return '昨天'
  } else {
    return msgTime.format('MM-DD')
  }
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
    conflicts.value = []
    ElMessage.warning('结束时间必须晚于开始时间')
    return
  }

  try {
    checkingConflict.value = true
    const res = await exchangeAPI.checkConflict({
      partnerId: currentUserId.value,
      startTime: exchangeForm.value.startTime,
      endTime: exchangeForm.value.endTime
    })
    conflicts.value = res.data.conflicts || []
  } catch (e) {
    conflicts.value = []
  } finally {
    checkingConflict.value = false
  }
}

async function createExchange() {
  if (!exchangeForm.value.teachSkill || !exchangeForm.value.learnSkill) {
    ElMessage.warning('请选择教授和学习的技能')
    return
  }

  const hasTime = exchangeForm.value.startTime && exchangeForm.value.endTime
  if (hasTime && conflicts.value.length > 0) {
    ElMessage.warning('时间存在冲突，请调整时间或保存为待协商')
    return
  }

  try {
    submitting.value = true
    const data = {
      partnerId: currentUserId.value,
      skills: {
        teach: [exchangeForm.value.teachSkill],
        learn: [exchangeForm.value.learnSkill]
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
    resetExchangeForm()
  } catch (e) {
    if (e.response?.status === 409) {
      conflicts.value = e.response.data.conflicts || []
      ElMessage.warning('时间存在冲突')
    } else {
      ElMessage.error(e.message || '发起失败')
    }
  } finally {
    submitting.value = false
  }
}

async function saveAsNegotiating() {
  if (!exchangeForm.value.teachSkill || !exchangeForm.value.learnSkill) {
    ElMessage.warning('请选择教授和学习的技能')
    return
  }

  try {
    submitting.value = true
    const data = {
      partnerId: currentUserId.value,
      skills: {
        teach: [exchangeForm.value.teachSkill],
        learn: [exchangeForm.value.learnSkill]
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
    resetExchangeForm()
  } catch (e) {
    ElMessage.error(e.message || '保存失败')
  } finally {
    submitting.value = false
  }
}

function resetExchangeForm() {
  exchangeForm.value = {
    teachSkill: '',
    learnSkill: '',
    startTime: null,
    endTime: null
  }
  conflicts.value = []
}
</script>

<style scoped>
.chat-page {
  height: calc(100vh - 120px);
}

.chat-layout {
  display: flex;
  height: 100%;
  background: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.sidebar {
  width: 320px;
  border-right: 1px solid #eee;
  display: flex;
  flex-direction: column;
}

.sidebar-header {
  padding: 20px;
  border-bottom: 1px solid #eee;
}

.sidebar-header h3 {
  margin: 0;
  font-size: 18px;
  color: #333;
}

.conversation-list {
  flex: 1;
  overflow-y: auto;
}

.conversation-item {
  display: flex;
  gap: 12px;
  padding: 16px 20px;
  cursor: pointer;
  transition: all 0.2s;
  border-bottom: 1px solid #f5f5f5;
}

.conversation-item:hover {
  background: #f5f7ff;
}

.conversation-item.active {
  background: #eef1ff;
}

.conv-info {
  flex: 1;
  min-width: 0;
}

.conv-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 4px;
}

.conv-name {
  font-weight: 600;
  color: #333;
}

.conv-time {
  font-size: 12px;
  color: #999;
}

.conv-message {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.last-msg {
  color: #666;
  font-size: 13px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  max-width: 180px;
}

.unread-badge {
  margin-left: 8px;
}

.chat-main {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.chat-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px 24px;
  border-bottom: 1px solid #eee;
}

.chat-username {
  font-weight: 600;
  font-size: 16px;
  color: #333;
  flex: 1;
}

.messages-container {
  flex: 1;
  padding: 24px;
  overflow-y: auto;
  background: #fafafa;
}

.message-row {
  display: flex;
  gap: 12px;
  margin-bottom: 16px;
}

.message-row.sent {
  flex-direction: row-reverse;
}

.chat-empty {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #999;
  gap: 16px;
}

.chat-input-area {
  padding: 16px 24px;
  border-top: 1px solid #eee;
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

.conflict-status-tag {
  display: inline-block;
  padding: 0 6px;
  border-radius: 4px;
  font-size: 11px;
  font-weight: 500;
  margin-left: 6px;
  line-height: 18px;
}

.conflict-status-tag.status-confirmed {
  background: #fff1f0;
  color: #f5222d;
  border: 1px solid #ffa39e;
}

.conflict-status-tag.status-negotiating {
  background: #fff7e6;
  color: #d46b08;
  border: 1px solid #ffd591;
}

.conflict-time {
  font-size: 13px;
  color: #666;
}
</style>
