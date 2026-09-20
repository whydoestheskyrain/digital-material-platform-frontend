<template>
  <div class="page-wrapper">
    <div class="container">
      <div class="page-header">
        <h1>用户管理</h1>
        <div class="header-actions">
          <el-button @click="$router.push('/admin/home')">返回控制台</el-button>
        </div>
      </div>

      <el-table 
        :data="list" 
        border 
        stripe 
        class="data-table"
        :empty-text="list.length === 0 ? '暂无用户' : ''"
      >
        <el-table-column prop="id" label="ID" width="80" />
        <el-table-column prop="username" label="用户名" width="120" />
        <el-table-column prop="nickname" label="昵称" width="120" />
        <el-table-column prop="phone" label="手机号" width="140" />
        <el-table-column prop="role" label="角色" width="100">
          <template #default="scope">
            <el-tag :type="scope.row.role === 'ADMIN' ? 'danger' : 'success'">
              {{ scope.row.role === 'ADMIN' ? '管理员' : '普通用户' }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column label="操作" width="150">
          <template #default="scope">
            <el-button 
              size="small" 
              type="danger" 
              @click="del(scope.row.id)"
              :disabled="scope.row.role === 'ADMIN'"
            >删除</el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage } from 'element-plus'
import request from '@/utils/request'
import { getCurrentUser } from '@/utils/accountManager'

const router = useRouter()
const list = ref([])

onMounted(() => {
  const user = getCurrentUser()
  if (!user || user.role !== 'ADMIN') {
    ElMessage.warning('无管理员权限！')
    router.push('/login')
    return
  }

  request.get('/user/list').then(res => {
    list.value = res.data || []
  }).catch(() => {
    ElMessage.error('获取用户列表失败')
  })
})

const del = async (id) => {
  try {
    await request.get('/user/delete/' + id)
    ElMessage.success('删除成功')
    list.value = list.value.filter(i => i.id !== id)
  } catch (err) {
    ElMessage.error('删除失败')
  }
}
</script>

<style scoped>
.page-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.page-header h1 {
  font-size: 24px;
  color: #333;
}

.header-actions {
  display: flex;
  gap: 10px;
}

.data-table {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
}

@media (max-width: 768px) {
  .page-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 15px;
  }
}
</style>
