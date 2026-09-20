<template>
  <div class="notification-container">
    <div class="notification-header">
      <div class="header-left" @click="goBack">
        <span class="back-icon">←</span>
        <span>返回</span>
      </div>
      <div class="header-title">消息中心</div>
      <div class="header-right">
        <el-button type="text" @click="markAllRead" v-if="unreadCount > 0">
          全部已读
        </el-button>
      </div>
    </div>

    <div class="notification-tabs">
      <div 
        class="tab-item" 
        :class="{ active: activeTab === 'notifications' }"
        @click="activeTab = 'notifications'"
      >
        <span>消息通知</span>
        <span v-if="unreadCount > 0" class="badge">{{ unreadCount }}</span>
      </div>
      <div 
        class="tab-item" 
        :class="{ active: activeTab === 'feedback' }"
        @click="activeTab = 'feedback'"
      >
        <span>我的反馈</span>
      </div>
      <div 
        class="tab-item" 
        :class="{ active: activeTab === 'broadcast' }"
        v-if="isAdmin"
        @click="activeTab = 'broadcast'"
      >
        <span>管理员广播</span>
      </div>
    </div>

    <!-- 消息通知列表 -->
    <div v-show="activeTab === 'notifications'" class="notification-content">
      <div v-if="loading" class="loading-state">
        <div class="loading-spinner"></div>
        <p>加载中...</p>
      </div>
      <div v-else-if="notifications.length === 0" class="empty-state">
        <p>暂无消息</p>
      </div>
      <div v-else class="notification-list">
        <div 
          v-for="item in notifications" 
          :key="item.id" 
          class="notification-item"
          :class="{ unread: item.status === 0 }"
          @click="markRead(item)"
        >
          <div class="notification-icon">
            <span v-if="item.type === 'order'">📦</span>
            <span v-else-if="item.type === 'feedback'">💬</span>
            <span v-else-if="item.type === 'broadcast'">📢</span>
            <span v-else>📩</span>
          </div>
          <div class="notification-body">
            <div class="notification-title">{{ item.title }}</div>
            <div class="notification-content">{{ item.content }}</div>
            <div class="notification-time">{{ formatTime(item.createTime) }}</div>
          </div>
          <div class="notification-status">
            <span v-if="item.status === 0" class="dot"></span>
          </div>
        </div>
      </div>
    </div>

    <!-- 反馈列表 -->
    <div v-show="activeTab === 'feedback'" class="notification-content">
      <div class="feedback-header">
        <el-button type="primary" @click="showFeedbackForm = true">提交反馈</el-button>
      </div>
      <div v-if="feedbackLoading" class="loading-state">
        <div class="loading-spinner"></div>
        <p>加载中...</p>
      </div>
      <div v-else-if="feedbacks.length === 0" class="empty-state">
        <p>暂无反馈</p>
      </div>
      <div v-else class="feedback-list">
        <div v-for="item in feedbacks" :key="item.id" class="feedback-item">
          <div class="feedback-header">
            <div class="feedback-title">{{ item.title }}</div>
            <span class="status-tag" :class="getStatusClass(item.status)">
              {{ getStatusText(item.status) }}
            </span>
          </div>
          <div class="feedback-content">{{ item.content }}</div>
          <div v-if="item.reply" class="feedback-reply">
            <div class="reply-label">管理员回复：</div>
            <div class="reply-content">{{ item.reply }}</div>
          </div>
          <div class="feedback-time">{{ formatTime(item.createTime) }}</div>
        </div>
      </div>
    </div>

    <!-- 管理员广播 -->
    <div v-show="activeTab === 'broadcast'" class="notification-content">
      <div class="broadcast-form">
        <h3 class="form-title">📢 发送广播</h3>
        <el-form :model="broadcastForm" label-width="80px">
          <el-form-item label="标题">
            <el-input v-model="broadcastForm.title" placeholder="输入广播标题" />
          </el-form-item>
          <el-form-item label="内容">
            <textarea v-model="broadcastForm.content" rows="6" placeholder="输入广播内容" class="custom-textarea" />
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="sendBroadcast" :loading="broadcasting">
              发送广播
            </el-button>
          </el-form-item>
        </el-form>
      </div>
    </div>

    <!-- 提交反馈弹窗 -->
    <el-dialog v-model="showFeedbackForm" title="提交反馈" width="500px">
      <el-form :model="feedbackForm" label-width="80px">
        <el-form-item label="标题" required>
          <el-input v-model="feedbackForm.title" placeholder="输入反馈标题" />
        </el-form-item>
        <el-form-item label="内容" required>
          <textarea v-model="feedbackForm.content" rows="6" placeholder="详细描述您的问题或建议" class="custom-textarea" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="showFeedbackForm = false">取消</el-button>
        <el-button type="primary" @click="submitFeedback" :loading="submitting">
          提交反馈
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage } from 'element-plus'
import request from '@/utils/request'
import { getCurrentUser } from '@/utils/accountManager'

const router = useRouter()
const currentUser = getCurrentUser()
const isAdmin = currentUser?.role === 'ADMIN'

const activeTab = ref('notifications')
const notifications = ref([])
const feedbacks = ref([])
const loading = ref(false)
const feedbackLoading = ref(false)
const unreadCount = ref(0)

const showFeedbackForm = ref(false)
const feedbackForm = ref({ title: '', content: '' })
const submitting = ref(false)

const broadcastForm = ref({ title: '', content: '' })
const broadcasting = ref(false)

const formatTime = (time) => {
  if (!time) return ''
  const date = new Date(time)
  const now = new Date()
  const diff = now - date
  const minutes = Math.floor(diff / 60000)
  const hours = Math.floor(diff / 3600000)
  const days = Math.floor(diff / 86400000)

  if (minutes < 1) return '刚刚'
  if (minutes < 60) return `${minutes}分钟前`
  if (hours < 24) return `${hours}小时前`
  if (days < 7) return `${days}天前`
  return date.toLocaleDateString('zh-CN')
}

const getStatusText = (status) => {
  if (status === 0) return '待处理'
  if (status === 1) return '已回复'
  if (status === 2) return '已关闭'
  return '未知'
}

const getStatusClass = (status) => {
  if (status === 0) return 'pending'
  if (status === 1) return 'replied'
  if (status === 2) return 'closed'
  return ''
}

const goBack = () => {
  router.back()
}

const loadNotifications = async () => {
  if (!currentUser) return
  loading.value = true
  try {
    const res = await request.get(`/notification/list/${currentUser.id}`)
    if (res.code === 200) {
      notifications.value = res.data || []
    }
    const countRes = await request.get(`/notification/count/${currentUser.id}`)
    if (countRes.code === 200) {
      unreadCount.value = countRes.data || 0
    }
  } catch (e) {
    console.error('加载消息失败:', e)
  } finally {
    loading.value = false
  }
}

const loadFeedbacks = async () => {
  if (!currentUser) return
  feedbackLoading.value = true
  try {
    const res = await request.get(`/feedback/list/${currentUser.id}`)
    if (res.code === 200) {
      feedbacks.value = res.data || []
    }
  } catch (e) {
    console.error('加载反馈失败:', e)
  } finally {
    feedbackLoading.value = false
  }
}

const markRead = async (item) => {
  if (item.status === 0) {
    try {
      await request.put(`/notification/read/${item.id}`)
      item.status = 1
      unreadCount.value = Math.max(0, unreadCount.value - 1)
    } catch (e) {
      console.error('标记已读失败:', e)
    }
  }
}

const markAllRead = async () => {
  if (!currentUser) return
  try {
    await request.put(`/notification/readAll/${currentUser.id}`)
    notifications.value.forEach(n => n.status = 1)
    unreadCount.value = 0
    ElMessage.success('已全部标记为已读')
  } catch (e) {
    console.error('全部已读失败:', e)
  }
}

const submitFeedback = async () => {
  if (!feedbackForm.value.title || !feedbackForm.value.content) {
    ElMessage.warning('请填写完整信息')
    return
  }
  submitting.value = true
  try {
    const res = await request.post('/feedback/submit', {
      userId: currentUser.id,
      title: feedbackForm.value.title,
      content: feedbackForm.value.content
    })
    if (res.code === 200) {
      ElMessage.success('反馈提交成功')
      showFeedbackForm.value = false
      feedbackForm.value = { title: '', content: '' }
      loadFeedbacks()
    } else {
      ElMessage.error(res.msg || '提交失败')
    }
  } catch (e) {
    ElMessage.error('提交失败')
  } finally {
    submitting.value = false
  }
}

const sendBroadcast = async () => {
  if (!broadcastForm.value.title || !broadcastForm.value.content) {
    ElMessage.warning('请填写完整信息')
    return
  }
  broadcasting.value = true
  try {
    const res = await request.post('/admin/stats/broadcast', null, {
      params: { 
        title: broadcastForm.value.title, 
        content: broadcastForm.value.content 
      } 
    })
    if (res.success || res.code === 200) {
      ElMessage.success('广播发送成功')
      broadcastForm.value = { title: '', content: '' }
    } else {
      ElMessage.error(res.message || '发送失败')
    }
  } catch (e) {
    ElMessage.error('发送失败')
  } finally {
    broadcasting.value = false
  }
}

onMounted(() => {
  loadNotifications()
})
</script>

<style scoped>
.notification-container {
  min-height: 100vh;
  background: linear-gradient(135deg, #f5f7fa 0%, #e4e8ec 100%);
}

.notification-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 24px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  position: sticky;
  top: 0;
  z-index: 100;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  font-size: 14px;
}

.back-icon {
  font-size: 18px;
}

.header-title {
  font-size: 18px;
  font-weight: 600;
}

.notification-tabs {
  display: flex;
  background: white;
  border-bottom: 1px solid #f0f0f0;
}

.tab-item {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 16px 24px;
  cursor: pointer;
  font-size: 14px;
  color: #666;
  position: relative;
  transition: all 0.3s;
}

.tab-item.active {
  color: #667eea;
  font-weight: 600;
}

.tab-item.active::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 2px;
  background: #667eea;
}

.badge {
  background: #ff6b6b;
  color: white;
  font-size: 11px;
  padding: 1px 6px;
  border-radius: 10px;
  min-width: 16px;
  text-align: center;
}

.notification-content {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.loading-state,
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 60px 20px;
  color: #999;
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 3px solid #e0e0e0;
  border-top-color: #667eea;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 16px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* 消息列表 */
.notification-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.notification-item {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding: 16px;
  background: white;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s;
}

.notification-item:hover {
  box-shadow: 0 2px 12px rgba(0,0,0,0.08);
}

.notification-item.unread {
  background: #f8fafc;
  border-left: 3px solid #667eea;
}

.notification-icon {
  font-size: 24px;
  flex-shrink: 0;
}

.notification-body {
  flex: 1;
  min-width: 0;
}

.notification-title {
  font-size: 14px;
  font-weight: 600;
  color: #1a1a2e;
  margin-bottom: 4px;
}

.notification-content {
  font-size: 13px;
  color: #666;
  line-height: 1.5;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}

.notification-time {
  font-size: 12px;
  color: #999;
  margin-top: 8px;
}

.notification-status {
  flex-shrink: 0;
}

.dot {
  display: block;
  width: 8px;
  height: 8px;
  background: #667eea;
  border-radius: 50%;
}

/* 反馈列表 */
.feedback-header {
  margin-bottom: 16px;
}

.feedback-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.feedback-item {
  padding: 16px;
  background: white;
  border-radius: 12px;
}

.feedback-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.feedback-title {
  font-size: 14px;
  font-weight: 600;
  color: #1a1a2e;
}

.status-tag {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 10px;
}

.status-tag.pending {
  background: #fff3e0;
  color: #e65100;
}

.status-tag.replied {
  background: #e3f2fd;
  color: #1976d2;
}

.status-tag.closed {
  background: #f5f5f5;
  color: #999;
}

.feedback-content {
  font-size: 13px;
  color: #666;
  line-height: 1.5;
  margin-bottom: 8px;
}

.feedback-reply {
  background: #f8fafc;
  padding: 12px;
  border-radius: 8px;
  margin-bottom: 8px;
}

.reply-label {
  font-size: 12px;
  color: #999;
  margin-bottom: 4px;
}

.reply-content {
  font-size: 13px;
  color: #333;
  line-height: 1.5;
}

.feedback-time {
  font-size: 12px;
  color: #999;
}

/* 广播表单 */
.broadcast-form {
  background: white;
  padding: 24px;
  border-radius: 12px;
}

.form-title {
  font-size: 16px;
  font-weight: 600;
  color: #1a1a2e;
  margin-bottom: 16px;
}

.custom-textarea {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid #dcdfe6;
  border-radius: 4px;
  font-size: 14px;
  resize: vertical;
  transition: border-color 0.3s;
}

.custom-textarea:focus {
  outline: none;
  border-color: #409eff;
}

.custom-textarea::placeholder {
  color: #c0c4cc;
}
</style>