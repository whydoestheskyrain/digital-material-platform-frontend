<template>
  <div class="mine-container">
    <!-- 左侧侧边栏菜单 -->
    <div class="mine-left">
      <div class="user-info">
        <!-- 头像上传 -->
        <el-upload
          class="avatar-upload"
          :http-request="customUploadAvatar"
          :show-file-list="false"
        >
          <img v-if="user.avatar" :src="user.avatar" class="avatar-img" />
          <img v-else src="/images/default-avatar.jpg" class="avatar-img" />
        </el-upload>

        <div class="name">{{ user.nickname || user.username }}</div>

        <!-- 余额信息 -->
        <div class="balance-info">
          <div class="balance-main">可用余额：¥{{ balance.toFixed(2) }}</div>
          <div class="balance-frozen">冻结余额：¥{{ frozenBalance.toFixed(2) }}</div>
          <el-button type="primary" size="small" @click="rechargeDialogVisible = true">充值</el-button>
        </div>
      </div>

      <div 
        class="menu-item" 
        :class="{active: menuKey === 'info'}"
        @click="menuKey = 'info'"
      >
        基本资料
      </div>
      <div 
        class="menu-item" 
        :class="{active: menuKey === 'buyOrder'}"
        @click="menuKey = 'buyOrder'"
      >
        我买到的订单
      </div>
      <div 
        class="menu-item" 
        :class="{active: menuKey === 'sellOrder'}"
        @click="menuKey = 'sellOrder'"
      >
        我卖出的订单
      </div>
      <div 
        class="menu-item" 
        :class="{active: menuKey === 'editNick'}"
        @click="menuKey = 'editNick'"
      >
        修改昵称
      </div>
      <div 
        class="menu-item" 
        :class="{active: menuKey === 'editPwd'}"
        @click="menuKey = 'editPwd'"
      >
        修改密码
      </div>
      <!-- 新增：修改手机号侧边菜单，放到左侧正确位置 -->
      <div 
        class="menu-item" 
        :class="{active: menuKey === 'editPhone'}"
        @click="menuKey = 'editPhone'"
      >
        修改手机号
      </div>
      <!-- 管理员专属菜单 -->
      <div 
        v-if="user.role === 'ADMIN'"
        class="menu-item"
        @click="goToAdmin"
      >
        🎛️ 管理控制台
      </div>
      <div class="menu-item logout" @click="handleLogout">
        退出登录
      </div>
    </div>

    <!-- 右侧内容区域 -->
    <div class="mine-right">
      <!-- 基本资料 -->
      <div v-if="menuKey === 'info'" class="content-card">
        <h3>基本资料</h3>
        <div class="info-row"><span>用户ID</span>{{ user.id }}</div>
        <div class="info-row"><span>用户名</span>{{ user.username }}</div>
        <div class="info-row"><span>昵称</span>{{ user.nickname }}</div>
        <div class="info-row"><span>手机号</span>{{ desensitizePhone(user.phone) }}</div>
      </div>

      <!-- 买家订单 -->
      <div v-if="menuKey === 'buyOrder'" class="content-card">
        <h3>我买到的订单</h3>
        <div v-loading="loadingBuyerOrders" element-loading-text="加载订单中...">
          <div v-if="!loadingBuyerOrders && buyerList.length === 0" class="empty-tip">
            <div class="empty-icon">📦</div>
            <span>暂无订单</span>
          </div>
          <el-table v-else :data="buyerList" border size="small" style="width:100%">
            <el-table-column label="订单ID" prop="id" />
            <el-table-column label="商品名称">
              <template #default="scope">
                {{ scope.row.productName || scope.row.product?.title || '未命名商品' }}
              </template>
            </el-table-column>
            <el-table-column label="价格" prop="price" />
            <el-table-column label="状态">
              <template #default="scope">
                <el-tag :type="getStatusTagType(scope.row.status)">
                  {{ getStatusText(scope.row.status) }}
                </el-tag>
              </template>
            </el-table-column>
            <el-table-column label="操作" width="260">
  <template #default="scope">
    <!-- 待支付订单显示支付按钮 -->
    <el-button
      v-if="scope.row.status === ORDER_STATUS.PENDING"
      type="success"
      size="small"
      @click="payOrder(scope.row.id)"
    >立即支付</el-button>
    <!-- 待支付订单显示取消按钮（可选） -->
    <el-button
      v-if="scope.row.status === ORDER_STATUS.PENDING"
      type="warning"
      size="small"
      @click="cancelOrder(scope.row.id)"
    >取消订单</el-button>
    <!-- 已发货订单显示确认收货按钮 -->
    <el-button
      v-if="scope.row.status === ORDER_STATUS.SHIPPED"
      type="primary"
      size="small"
      @click="confirmReceive(scope.row.id)"
    >确认收货</el-button>
  </template>
</el-table-column>
          </el-table>
        </div>
      </div>

      <!-- 卖家订单 -->
      <div v-if="menuKey === 'sellOrder'" class="content-card">
        <h3>我卖出的订单</h3>
        <div v-loading="loadingSellerOrders" element-loading-text="加载订单中...">
          <div v-if="!loadingSellerOrders && sellerList.length === 0" class="empty-tip">
            <div class="empty-icon">📦</div>
            <span>暂无订单</span>
          </div>
          <el-table v-else :data="sellerList" border size="small" style="width:100%">
            <el-table-column label="订单ID" prop="id" />
            <el-table-column label="商品名称">
              <template #default="scope">
                {{ scope.row.productName || scope.row.product?.title || '未命名商品' }}
              </template>
            </el-table-column>
            <el-table-column label="价格" prop="price" />
            <el-table-column label="状态">
              <template #default="scope">
                <el-tag :type="getStatusTagType(scope.row.status)">
                  {{ getStatusText(scope.row.status) }}
                </el-tag>
              </template>
            </el-table-column>
          </el-table>
        </div>
      </div>

      <!-- 修改昵称 -->
      <div v-if="menuKey === 'editNick'" class="content-card">
        <h3>修改昵称</h3>
        <el-input v-model="editNickname" placeholder="请输入新昵称" style="width:300px;margin:15px 0" />
        <el-button type="primary" @click="updateNickname">保存修改</el-button>
      </div>

      <!-- 修改密码 -->
      <div v-if="menuKey === 'editPwd'" class="content-card">
        <h3>修改密码</h3>
        <el-input v-model="oldPassword" placeholder="旧密码" type="password" style="width:300px;margin:10px 0" />
        <el-input v-model="newPassword" placeholder="新密码" type="password" style="width:300px;margin:10px 0" />
        <el-input v-model="confirmPassword" placeholder="确认新密码" type="password" style="width:300px;margin:10px 0" />
        <el-button type="primary" @click="updatePassword">确认修改</el-button>
      </div>

      <!-- 修改手机号面板（保留，删除上方错位menu-item） -->
      <div v-if="menuKey === 'editPhone'" class="content-card">
        <h3>修改绑定手机号</h3>
        <el-input 
          v-model="editPhone" 
          placeholder="请输入新的11位手机号" 
          style="width:300px;margin:15px 0"
          maxlength="11"
        />
        <el-button type="primary" @click="submitUpdatePhone">保存修改</el-button>
      </div>
    </div>

    <!-- 充值弹窗 -->
    <el-dialog title="充值" v-model="rechargeDialogVisible" width="300px">
      <el-input 
        v-model="rechargeAmount" 
        type="number" 
        placeholder="请输入充值金额"
        :precision="2"
        :min="0.01"
        step="0.01"
      />
      <template #footer>
        <el-button @click="rechargeDialogVisible = false">取消</el-button>
        <el-button type="primary" @click="handleRecharge">确认充值</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import request from '@/utils/request'
import { getCurrentToken, getCurrentUser, removeAccount, updateCurrentUser } from '@/utils/accountManager'
// 订单状态常量【和后端完全对齐】
const ORDER_STATUS = {
  PENDING: 1,    // 待支付
  PAID: 2,       // 待发货
  SHIPPED: 3,    // 待收货
  COMPLETED: 4,  // 已完成
  CANCELLED: 5   // 已取消
}

const statusMap = {
  [ORDER_STATUS.PENDING]: '待支付',
  [ORDER_STATUS.PAID]: '待发货',
  [ORDER_STATUS.SHIPPED]: '待收货',
  [ORDER_STATUS.COMPLETED]: '已完成',
  [ORDER_STATUS.CANCELLED]: '已取消'
}
const statusTypeMap = {
  [ORDER_STATUS.PENDING]: 'warning',
  [ORDER_STATUS.PAID]: 'primary',
  [ORDER_STATUS.SHIPPED]: 'success',
  [ORDER_STATUS.COMPLETED]: 'info',
  [ORDER_STATUS.CANCELLED]: 'danger'
}

const router = useRouter()
const menuKey = ref('info')
const user = ref({})
const buyerList = ref([])
const sellerList = ref([])
const editNickname = ref('')
const oldPassword = ref('')
const newPassword = ref('')
const confirmPassword = ref('')
const loadingBuyerOrders = ref(false)
const loadingSellerOrders = ref(false)
// 状态文本
const getStatusText = (s) => statusMap[s] || '未知'
const getStatusTagType = (s) => statusTypeMap[s] || ''

// 修改手机号弹窗变量
const editPhone = ref('')
const editPhoneDialogVisible = ref(false)
// 打开修改手机号弹窗，回填当前手机号
// 提交修改手机号
const submitUpdatePhone = async () => {
    const phone = editPhone.value.trim()
    // 前端格式校验
    const phoneReg = /^1[3-9]\d{9}$/
    if (!phoneReg.test(phone)) {
        ElMessage.warning('请输入合法11位手机号')
        return
    }
    try {
        const res = await request.put('/user/update/phone', { phone })
        if (res.code === 200) {
            ElMessage.success('手机号修改成功')
            // 更新本地用户缓存
            user.value.phone = phone
            updateCurrentUser(user.value)
            editPhoneDialogVisible.value = false
        } else {
            ElMessage.error(res.msg)
        }
    } catch (err) {
        ElMessage.error(err.response?.data?.msg || '修改失败')
    }
}



// 支付订单
const payOrder = async (orderId) => {
  try {
    const res = await request.put(`/order/pay/${orderId}`)
    if (res.code === 200) {
      ElMessage.success('支付成功，等待卖家发货')
      // 刷新订单列表和余额
      await refreshOrdersByMenu()
      await getBalance()
    } else {
      ElMessage.error(res.msg || '支付失败')
    }
  } catch (err) {
    ElMessage.error(err.response?.data?.msg || '支付失败，请稍后重试')
  }
}
// 手机号脱敏（增强健壮性）
const desensitizePhone = (phone) => {
  if (!phone || phone.length !== 11) return phone || '未绑定'
  return phone.replace(/(\d{3})\d{4}(\d{4})/, '$1****$2')
}

// ========== 余额相关 ==========
const balance = ref(0)
const frozenBalance = ref(0)
const rechargeAmount = ref(null)
const rechargeDialogVisible = ref(false)

// 获取余额
const getBalance = async () => {
  try {
    const res = await request.get('/user/balance')
    if (res.code === 200) {
      balance.value = res.data.balance || 0
      frozenBalance.value = res.data.frozenBalance || 0
    }
  } catch (err) {
    console.error('获取余额失败', err)
  }
}

// 充值（增强校验）
const handleRecharge = async () => {
  let amount = parseFloat(rechargeAmount.value)
  if (isNaN(amount) || amount < 0.01) {
    ElMessage.warning('充值金额至少为0.01元')
    return
  }
  // 保留两位小数
  amount = Math.round(amount * 100) / 100
  try {
    const res = await request.post('/recharge/create', { amount })
    if (res.code === 200) {
      // 后端返回的已经是完整的表单HTML，直接写入页面即可自动提交
      document.write(res.data)
      document.close()
      rechargeDialogVisible.value = false
      rechargeAmount.value = null
    } else {
      ElMessage.error(res.msg || '充值失败')
    }
  } catch (err) {
    ElMessage.error(err.response?.data?.msg || '充值失败，请稍后重试')
  }
}

// 加载买家订单
const loadBuyerOrders = async () => {
  if (!user.value?.id) {
    console.log('用户id为空')
    return
  }
  loadingBuyerOrders.value = true
  try {
    // 正确后端地址：listAll 传 buyerId 参数
    const res = await request.get(`/order/listAll`, {
      params: { buyerId: user.value.id }
    })
    console.log('买家订单返回', res)
    buyerList.value = res.code === 200 ? res.data : []
  } catch (e) {
    console.error(e)
    ElMessage.error('加载买家订单失败')
  } finally {
    loadingBuyerOrders.value = false
  }
}

// 加载卖家订单
const loadSellerOrders = async () => {
  if (!user.value?.id) {
    console.log('用户id为空')
    return
  }
  loadingSellerOrders.value = true
  try {
    // 正确后端地址 /seller/{id}/all
    const res = await request.get(`/order/seller/${user.value.id}/all`)
    console.log('卖家订单返回', res)
    sellerList.value = res.code === 200 ? res.data : []
  } catch (e) {
    console.error(e)
    ElMessage.error('加载卖家订单失败')
  } finally {
    loadingSellerOrders.value = false
  }
}

// 根据当前菜单刷新订单
const refreshOrdersByMenu = () => {
  if (menuKey.value === 'buyOrder') {
    loadBuyerOrders()
  } else if (menuKey.value === 'sellOrder') {
    loadSellerOrders()
  }
}

// 头像上传（添加文件类型和大小校验）
const customUploadAvatar = async (options) => {
  const file = options.file
  const isImage = file.type.startsWith('image/')
  const isLt2M = file.size / 1024 / 1024 < 2
  if (!isImage) {
    ElMessage.error('只能上传图片文件！')
    return
  }
  if (!isLt2M) {
    ElMessage.error('图片大小不能超过 2MB！')
    return
  }
  const formData = new FormData()
  formData.append('file', file)
  try {
    const res = await request.post('/user/upload/avatar', formData, {
      headers: { 'Content-Type': 'multipart/form-data' }
    })
    if (res.code === 200) {
      user.value.avatar = res.data
      updateCurrentUser(user.value)
      ElMessage.success('头像更新成功')
    } else {
      ElMessage.error(res.msg || '上传失败')
    }
  } catch (err) {
    console.error('上传失败', err)
    ElMessage.error('上传失败，请稍后重试')
  }
}

// 修改昵称
const updateNickname = async () => {
  if (!editNickname.value.trim()) {
    ElMessage.warning('请输入昵称')
    return
  }
  try {
    await request.put('/user/update/nickname', {
      id: user.value.id,
      nickname: editNickname.value
    })
    user.value.nickname = editNickname.value
    updateCurrentUser(user.value)
    ElMessage.success('昵称修改成功')
  } catch {
    ElMessage.error('修改失败')
  }
}

// 修改密码
const updatePassword = async () => {
  if (!oldPassword.value || !newPassword.value || !confirmPassword.value) {
    ElMessage.warning('请填写完整')
    return
  }
  if (newPassword.value !== confirmPassword.value) {
    ElMessage.error('两次密码不一致')
    return
  }
  try {
    await request.put('/user/update/password', {
      id: user.value.id,
      oldPassword: oldPassword.value,
      newPassword: newPassword.value
    })
    ElMessage.success('密码修改成功，请重新登录')
    // 清除当前账号并跳转登录
    const current = getCurrentUser()
    if (current) {
      removeAccount('user_' + current.id)
    }
    delete request.defaults.headers.common['Authorization']
    router.push('/login')
  } catch (err) {
    ElMessage.error(err.response?.data?.msg || '旧密码错误或修改失败')
  }
}

// 取消订单（添加二次确认）
const cancelOrder = async (orderId) => {
  try {
    await ElMessageBox.confirm('确定要取消该订单吗？', '提示', {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning'
    })
    await request.put(`/order/cancel/${orderId}`)
    ElMessage.success('已取消')
    refreshOrdersByMenu() // 刷新当前菜单订单
  } catch (err) {
    if (err !== 'cancel') {
      ElMessage.error('取消失败')
    }
  }
}

// 确认收货（添加二次确认）
const confirmReceive = async (orderId) => {
  try {
    await ElMessageBox.confirm('请确认已收到商品，确认后将完成交易。', '确认收货', {
      confirmButtonText: '确认收货',
      cancelButtonText: '取消',
      type: 'info'
    })
    await request.put(`/order/confirm/${orderId}`)
    ElMessage.success('收货成功')
    refreshOrdersByMenu()
  } catch (err) {
    if (err !== 'cancel') {
      ElMessage.error('确认收货失败')
    }
  }
}

// 跳转到管理控制台
const goToAdmin = () => {
  router.push('/admin/home')
}

// 退出登录
const handleLogout = () => {
  const current = getCurrentUser()
  if (current) {
    removeAccount('user_' + current.id)
  }
  delete request.defaults.headers.common['Authorization']
  ElMessage.success('已退出登录')
  router.push('/login')
}

// 监听菜单切换，自动刷新对应订单
watch(menuKey, (newKey) => {
  if (newKey === 'buyOrder') {
    loadBuyerOrders()
  } else if (newKey === 'sellOrder') {
    loadSellerOrders()
  }
})

// 生命周期
onMounted(async () => {
  const token = getCurrentToken()
  if (!token) {
    ElMessage.warning('请先登录')
    router.push('/login')
    return
  }
  try {
    const res = await request.get('/user/info')
    if (res.code === 200 && res.data?.id) {
      user.value = res.data
      editNickname.value = user.value.nickname
      // 并发加载余额和初始订单
      await Promise.allSettled([getBalance(), refreshOrdersByMenu()])
    } else {
      ElMessage.error('获取用户信息失败，请重新登录')
      router.push('/login')
    }
  } catch (err) {
    ElMessage.error('获取用户信息失败')
    router.push('/login')
  }
})
</script>

<style scoped>
.balance-info {
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid #eee;
  font-size: 14px;
}
.balance-main {
  font-size: 16px;
  font-weight: 600;
  color: #333;
  margin-bottom: 4px;
}
.balance-frozen {
  font-size: 13px;
  color: #409EFF;
  margin-bottom: 10px;
}
.mine-container {
  display: flex;
  max-width: 1200px;
  margin: 40px auto;
  gap: 20px;
  padding: 0 20px;
}

.mine-left {
  width: 220px;
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
  overflow: hidden;
}
.user-info {
  padding: 30px 20px;
  text-align: center;
  border-bottom: 1px solid #eee;
}
.avatar-img {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  object-fit: cover;
  margin: 0 auto 10px;
  display: block;
}
.avatar-upload {
  cursor: pointer;
}
.name {
  font-size: 16px;
  font-weight: 500;
  margin-bottom: 10px;
}
.menu-item {
  padding: 14px 20px;
  cursor: pointer;
  font-size: 14px;
  color: #333;
  transition: all 0.2s;
}
.menu-item.active {
  background: #409eff;
  color: #fff;
  border-left: 4px solid #2079cf;
}
.menu-item:hover:not(.active) {
  background: #f5f7fa;
  padding-left: 24px;
}
.logout {
  color: #f56c6c;
  margin-top: 20px;
}

.mine-right {
  flex: 1;
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
  padding: 30px;
}
.content-card h3 {
  font-size: 18px;
  margin-bottom: 20px;
  padding-bottom: 10px;
  border-bottom: 1px solid #eee;
}
.info-row {
  font-size: 15px;
  margin: 12px 0;
}
.info-row span {
  display: inline-block;
  width: 80px;
  color: #666;
}
.empty-tip {
  text-align: center;
  padding: 60px 0;
  color: #999;
}
.empty-icon {
  font-size: 50px;
  margin-bottom: 12px;
}
</style>