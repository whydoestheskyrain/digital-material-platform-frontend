<template>
  <div class="order-list">
    <!-- 头部标签页 -->
    <div class="tabs">
      <div 
        v-for="tab in tabs" 
        :key="tab.value"
        :class="['tab-item', { active: activeTab === tab.value }]"
        @click="activeTab = tab.value"
      >
        {{ tab.label }}
        <span class="tab-count" v-if="tab.count > 0">{{ tab.count }}</span>
      </div>
    </div>

    <!-- 订单列表 -->
    <div class="order-container">
      <div 
        v-for="order in orderList" 
        :key="order.id" 
        class="order-card"
      >
        <div class="order-header">
          <span class="order-no">订单号：{{ order.orderNo }}</span>
          <span :class="['order-status', getStatusClass(order.status)]">
            {{ getStatusText(order.status) }}
          </span>
        </div>
        
        <div class="order-items">
          <div class="item-card" @click="goToProduct(order.productId)">
            <img :src="order.imageUrl || '/images/game-default.png'" :alt="order.productName" class="item-img" />
            <div class="item-info">
              <h3 class="item-name">{{ order.productName }}</h3>
              <p class="item-desc">游戏类型：{{ order.gameType }}</p>
              <p class="item-price">¥{{ order.price }}</p>
            </div>
          </div>
        </div>

        <div class="order-footer">
          <div class="order-total">
            <span class="total-label">实付金额：</span>
            <span class="total-amount">¥{{ order.price }}</span>
          </div>
          <div class="order-actions">
            <template v-if="order.status === 1">
              <el-button 
                type="primary" 
                class="pay-btn"
                @click="handlePay(order)"
              >
                💳 支付
              </el-button>
            </template>
            <template v-else-if="order.status === 2">
              <span class="status-text">⏳ 等待卖家发货</span>
            </template>
            <template v-else-if="order.status === 3">
              <el-button 
                type="info" 
                class="info-btn"
                @click="viewAccountInfo(order)"
              >
                👁️ 查看账号信息
              </el-button>
              <el-button 
                type="success" 
                class="confirm-btn"
                @click="handleConfirm(order)"
              >
                ✅ 确认收货
              </el-button>
            </template>
            <template v-else>
              <span class="status-text">{{ getStatusText(order.status) }}</span>
            </template>
          </div>
        </div>
      </div>

      <!-- 空状态 -->
      <div v-if="orderList.length === 0" class="empty-state">
        <div class="empty-icon">📦</div>
        <p>暂无订单</p>
      </div>
    </div>

    <!-- 账号信息弹窗 -->
    <el-dialog 
      title="🎮 账号信息" 
      v-model="showAccountModal"
      width="400px"
      :close-on-click-modal="false"
    >
      <div v-if="accountInfo" class="account-info">
        <div class="info-section">
          <label>订单号</label>
          <span>{{ currentAccountOrder?.orderNo }}</span>
        </div>
        <div class="info-section">
          <label>商品名称</label>
          <span>{{ currentAccountOrder?.productName }}</span>
        </div>
        <div class="info-section">
          <label>账号信息</label>
          <pre class="account-data">{{ accountInfo }}</pre>
        </div>
      </div>
      <div v-else class="no-info">
        <p>暂无账号信息</p>
      </div>

      <template #footer>
        <el-button @click="showAccountModal = false">关闭</el-button>
      </template>
    </el-dialog>

    <!-- 支付方式选择弹窗 -->
    <el-dialog 
      title="选择支付方式" 
      v-model="showPayModal"
      width="400px"
      :close-on-click-modal="false"
    >
      <div class="pay-methods">
        <div 
          :class="['pay-method', { selected: selectedPayMethod === 'balance' }]"
          @click="selectedPayMethod = 'balance'"
        >
          <div class="method-icon">💰</div>
          <div class="method-info">
            <h4>余额支付</h4>
            <p>当前余额：¥{{ userBalance.toFixed(2) }}</p>
          </div>
          <div :class="['method-check', { active: selectedPayMethod === 'balance' }]">✓</div>
        </div>
        
        <div 
          :class="['pay-method', { selected: selectedPayMethod === 'alipay' }]"
          @click="selectedPayMethod = 'alipay'"
        >
          <div class="method-icon">🌐</div>
          <div class="method-info">
            <h4>支付宝支付</h4>
            <p>跳转到支付宝沙箱支付</p>
          </div>
          <div :class="['method-check', { active: selectedPayMethod === 'alipay' }]">✓</div>
        </div>
      </div>
      
      <div class="pay-amount-info">
        <span class="label">订单金额：</span>
        <span class="amount">¥{{ currentOrder?.price }}</span>
      </div>

      <template #footer>
        <el-button @click="showPayModal = false">取消</el-button>
        <el-button 
          type="primary" 
          @click="confirmPayMethod"
          :disabled="!selectedPayMethod"
        >
          确认支付
        </el-button>
      </template>
    </el-dialog>

    
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';
import request from '@/utils/request';
import { useRouter } from 'vue-router';

const router = useRouter();
const activeTab = ref('all');
const orderList = ref([]);
const showPayModal = ref(false);
const selectedPayMethod = ref('');
const currentOrder = ref(null);
const userBalance = ref(0);
const showAccountModal = ref(false);
const currentAccountOrder = ref(null);
const accountInfo = ref('');

const statusCounts = ref({
  unpaid: 0,
  paid: 0,
  delivered: 0,
  completed: 0
});

const tabs = computed(() => [
  { label: '📋 全部', value: 'all', count: orderList.value.length },
  { label: '💰 待支付', value: 'unpaid', count: statusCounts.value.unpaid },
  { label: '✅ 已支付', value: 'paid', count: statusCounts.value.paid },
  { label: '📦 已发货', value: 'delivered', count: statusCounts.value.delivered },
  { label: '🎉 已完成', value: 'completed', count: statusCounts.value.completed }
]);

const getStatusText = (status) => {
  const statusMap = {
    1: '待支付',
    2: '已支付',
    3: '已发货',
    4: '已完成',
    5: '已取消'
  };
  return statusMap[status] || '未知';
};

const getStatusClass = (status) => {
  const map = {
    1: 'unpaid',
    2: 'paid',
    3: 'delivered',
    4: 'completed',
    5: 'cancelled'
  };
  return map[status] || '';
};

const getStatusValue = (tab) => {
  const map = { unpaid:1, paid:2, delivered:3, completed:4 };
  return map[tab];
};

import { getCurrentUser } from '@/utils/accountManager'

const getList = async () => {
  try {
    const user = getCurrentUser();
    if (!user || !user.id) {
      orderList.value = [];
      return;
    }
    const params = { buyerId: user.id };
    if (activeTab.value !== 'all') {
      params.status = getStatusValue(activeTab.value);
    }
    const res = await request.get('/order/listAll', { params });
    if (res.code === 200) {
      orderList.value = res.data;
      updateStatusCounts();
    }
  } catch (err) {
    console.error('获取订单列表失败:', err);
  }
};

const updateStatusCounts = () => {
  statusCounts.value = {
    unpaid: orderList.value.filter(o => o.status === 1).length,
    paid: orderList.value.filter(o => o.status === 2).length,
    delivered: orderList.value.filter(o => o.status === 3).length,
    completed: orderList.value.filter(o => o.status === 4).length
  };
};

const getUserBalance = async () => {
  try {
    const res = await request.get('/user/balance');
    if (res.code === 200) {
      userBalance.value = res.data.balance || 0;
    }
  } catch (err) {
    console.error('获取余额失败:', err);
  }
};

const goToProduct = (productId) => {
  router.push(`/product/${productId}`);
};

const handlePay = (order) => {
  currentOrder.value = order;
  selectedPayMethod.value = '';
  getUserBalance();
  showPayModal.value = true;
};

const confirmPayMethod = async () => {
  if (!selectedPayMethod.value) {
    ElMessage.warning('请选择支付方式');
    return;
  }
  if (!currentOrder.value) {
    showPayModal.value = false;
    return;
  }
  
  try {
    showPayModal.value = false;
    
    if (selectedPayMethod.value === 'balance') {
      await ElMessageBox.confirm(
        '确认使用余额支付该订单？\n\n订单号：' + currentOrder.value.orderNo + '\n金额：¥' + currentOrder.value.price, 
        '💳 余额支付确认', 
        {
          type: 'warning',
          confirmButtonText: '确认支付',
          cancelButtonText: '取消'
        }
      );
      
      const res = await request.put(`/order/pay/${currentOrder.value.id}`);
      if (res.code === 200) {
        ElMessage.success('支付成功！等待卖家发货');
        getList();
        getUserBalance();
      } else {
        ElMessage.error(res.msg || '支付失败');
      }
    } 
    else if (selectedPayMethod.value === 'alipay') {
      window.location.href = `/alipay/pay?orderId=${currentOrder.value.id}`;
    }
  } catch (err) {
    console.error('支付失败:', err);
    if (err !== 'cancel') {
      ElMessage.error('支付失败，请稍后重试');
    }
  }
};

const viewAccountInfo = async (order) => {
  currentAccountOrder.value = order;
  showAccountModal.value = true;
  
  try {
    const res = await request.get(`/order/${order.id}/account-info`);
    if (res.code === 200) {
      accountInfo.value = res.data.accountInfo || '暂无账号信息';
    } else {
      accountInfo.value = res.msg || '获取账号信息失败';
    }
  } catch (err) {
    console.error('获取账号信息失败:', err);
    accountInfo.value = '获取账号信息失败';
  }
};

const handleConfirm = async (order) => {
  try {
    await ElMessageBox.confirm(
      '请确认您已收到游戏账号且账号信息正确！',
      '📦 确认收货',
      { type: 'warning' }
    );
    
    const res = await request.post(`/order/confirm/${order.id}`);
    if (res.code === 200) {
      ElMessage.success('交易完成');
      getList();
    }
  } catch (e) {
    if (e !== 'cancel') ElMessage.error('操作失败');
  }
};

onMounted(() => {
  getList();
  getUserBalance();

  window.addEventListener('focus', () => {
    getList();
    getUserBalance();
  });
});
</script>

<style scoped>
.order-list {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
}

.tabs {
  display: flex;
  gap: 15px;
  margin-bottom: 20px;
  padding-bottom: 15px;
  border-bottom: 1px solid #eee;
}

.tab-item {
  padding: 10px 20px;
  border-radius: 20px;
  background: #f5f5f5;
  cursor: pointer;
  transition: all 0.3s;
  position: relative;
}

.tab-item:hover {
  background: #e8e8e8;
}

.tab-item.active {
  background: #409eff;
  color: white;
}

.tab-count {
  background: #f56c6c;
  color: white;
  font-size: 12px;
  padding: 2px 6px;
  border-radius: 10px;
  margin-left: 5px;
}

.tab-item.active .tab-count {
  background: #fff;
  color: #409eff;
}

.order-container {
  display: grid;
  gap: 20px;
}

.order-card {
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
  overflow: hidden;
}

.order-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  background: #fafafa;
  border-bottom: 1px solid #eee;
}

.order-no {
  font-size: 14px;
  color: #666;
}

.order-status {
  font-size: 14px;
  font-weight: 600;
  padding: 4px 12px;
  border-radius: 20px;
}

.order-status.unpaid {
  background: #fff7e6;
  color: #d48806;
}

.order-status.paid {
  background: #e6f7ff;
  color: #1890ff;
}

.order-status.delivered {
  background: #f6ffed;
  color: #52c41a;
}

.order-status.completed {
  background: #f0f5ff;
  color: #597ef7;
}

.order-items {
  padding: 15px 20px;
}

.item-card {
  display: flex;
  gap: 15px;
  cursor: pointer;
}

.item-img {
  width: 100px;
  height: 100px;
  object-fit: cover;
  border-radius: 8px;
}

.item-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.item-name {
  font-size: 16px;
  font-weight: 600;
  color: #333;
}

.item-desc {
  font-size: 13px;
  color: #999;
}

.item-price {
  font-size: 18px;
  font-weight: 600;
  color: #f56c6c;
}

.order-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  border-top: 1px solid #eee;
}

.order-total {
  display: flex;
  align-items: baseline;
  gap: 5px;
}

.total-label {
  font-size: 14px;
  color: #666;
}

.total-amount {
  font-size: 20px;
  font-weight: 600;
  color: #f56c6c;
}

.order-actions {
  display: flex;
  gap: 10px;
}

.pay-btn, .confirm-btn {
  padding: 8px 20px;
  border-radius: 6px;
}

.status-text {
  color: #999;
  font-size: 14px;
}

.empty-state {
  text-align: center;
  padding: 60px 0;
  color: #999;
}

.empty-icon {
  font-size: 48px;
  margin-bottom: 15px;
}

.pay-methods {
  display: flex;
  flex-direction: column;
  gap: 15px;
  margin-bottom: 20px;
}

.pay-method {
  display: flex;
  align-items: center;
  padding: 15px;
  border: 2px solid #eee;
  border-radius: 10px;
  cursor: pointer;
  transition: all 0.3s;
}

.pay-method:hover {
  border-color: #409eff;
}

.pay-method.selected {
  border-color: #409eff;
  background: #f0f5ff;
}

.method-icon {
  font-size: 32px;
  margin-right: 15px;
}

.method-info h4 {
  margin: 0 0 5px 0;
  font-size: 16px;
  color: #333;
}

.method-info p {
  margin: 0;
  font-size: 13px;
  color: #999;
}

.method-check {
  margin-left: auto;
  width: 24px;
  height: 24px;
  border: 2px solid #ddd;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  color: transparent;
}

.method-check.active {
  background: #409eff;
  border-color: #409eff;
  color: white;
}

.pay-amount-info {
  display: flex;
  justify-content: center;
  align-items: baseline;
  gap: 5px;
  padding: 15px;
  background: #fafafa;
  border-radius: 8px;
  margin-bottom: 10px;
}

.pay-amount-info .label {
  font-size: 14px;
  color: #666;
}

.pay-amount-info .amount {
  font-size: 24px;
  font-weight: 600;
  color: #f56c6c;
}

.account-info {
  padding: 10px;
}

.info-section {
  display: flex;
  margin-bottom: 15px;
}

.info-section label {
  width: 100px;
  font-weight: 600;
  color: #666;
  flex-shrink: 0;
}

.info-section span {
  flex: 1;
  color: #333;
}

.account-data {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 8px;
  font-size: 14px;
  line-height: 1.8;
  color: #333;
  white-space: pre-wrap;
  word-break: break-all;
  border: 1px solid #e9ecef;
}

.no-info {
  text-align: center;
  padding: 40px;
  color: #999;
}

.info-btn {
  padding: 8px 16px;
  border-radius: 6px;
}
</style>