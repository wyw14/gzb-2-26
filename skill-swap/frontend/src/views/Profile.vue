<template>
  <div class="profile">
    <div class="card profile-header-card">
      <div class="profile-header">
        <el-avatar :src="userStore.user?.avatar" :size="100" />
        <div class="profile-info">
          <h1 class="username">{{ userStore.user?.username }}</h1>
          <div class="user-meta">
            <el-rate :model-value="userStore.user?.rating" disabled />
            <span class="rating">{{ userStore.user?.rating }}</span>
            <span class="divider">|</span>
            <span>{{ userStore.user?.exchangeCount || 0 }} 次交换</span>
            <span class="divider">|</span>
            <span>{{ userStore.user?.skillPoints || 0 }} 技能点</span>
          </div>
          <p class="bio">{{ userStore.user?.bio || '这个人很懒，什么都没写' }}</p>
        </div>
        <el-button type="primary" @click="showEditDialog = true">
          <el-icon><Edit /></el-icon>编辑资料
        </el-button>
      </div>
    </div>

    <div class="profile-grid">
      <div class="card skills-card">
        <div class="card-header">
          <h2 class="section-title">🎓 技能管理</h2>
          <div class="header-actions">
            <el-button type="primary" size="small" @click="$router.push('/publish')">
              <el-icon><Plus /></el-icon>发布技能
            </el-button>
          </div>
        </div>

        <div class="filter-bar">
          <div class="filter-item">
            <span class="filter-label">类型</span>
            <el-radio-group v-model="filters.type" size="small" @change="loadSkills">
              <el-radio-button value="">全部</el-radio-button>
              <el-radio-button value="teach">可教</el-radio-button>
              <el-radio-button value="learn">想学</el-radio-button>
            </el-radio-group>
          </div>
          <div class="filter-item">
            <span class="filter-label">分类</span>
            <el-select v-model="filters.category" placeholder="全部分类" size="small" clearable style="width: 140px" @change="loadSkills">
              <el-option v-for="cat in categories" :key="cat.id" :label="cat.name" :value="cat.id" />
            </el-select>
          </div>
          <div class="filter-item">
            <span class="filter-label">状态</span>
            <el-select v-model="filters.status" placeholder="全部状态" size="small" clearable style="width: 120px" @change="loadSkills">
              <el-option label="已启用" value="active" />
              <el-option label="已停用" value="inactive" />
            </el-select>
          </div>
          <div class="filter-item">
            <span class="filter-label">发布时间</span>
            <el-date-picker
              v-model="dateRange"
              type="daterange"
              range-separator="至"
              start-placeholder="开始日期"
              end-placeholder="结束日期"
              size="small"
              style="width: 240px"
              @change="onDateChange"
            />
          </div>
          <div class="filter-item">
            <span class="filter-label">排序</span>
            <el-select v-model="filters.sortBy" size="small" style="width: 130px" @change="loadSkills">
              <el-option label="最新发布" value="createdAt_desc" />
              <el-option label="最早发布" value="createdAt_asc" />
            </el-select>
          </div>
        </div>

        <div class="batch-actions" v-if="filteredSkills.length > 0">
          <el-checkbox v-model="isAllSelected" :indeterminate="isIndeterminate" @change="handleCheckAll">
            全选
          </el-checkbox>
          <span class="selected-count">已选 {{ selectedIds.length }} 项</span>
          <el-divider direction="vertical" />
          <el-button
            size="small"
            :disabled="selectedIds.length === 0"
            @click="handleBatchStatus('inactive')"
          >
            批量停用
          </el-button>
          <el-button
            size="small"
            :disabled="selectedIds.length === 0"
            @click="handleBatchStatus('active')"
          >
            批量启用
          </el-button>
          <el-button
            size="small"
            :disabled="selectedIds.length === 0"
            @click="showCategoryDialog = true"
          >
            批量改分类
          </el-button>
          <el-button
            size="small"
            type="danger"
            :disabled="selectedIds.length === 0"
            @click="handleBatchDelete"
          >
            批量删除
          </el-button>
        </div>

        <div v-if="filteredSkills.length" class="skills-list">
          <div v-for="skill in filteredSkills" :key="skill.id" class="skill-item" :class="{ inactive: skill.status === 'inactive' }">
            <el-checkbox :model-value="selectedIds.includes(skill.id)" @change="(val) => toggleSelect(skill.id, val)" />
            <div class="skill-main">
              <div class="skill-header">
                <span class="skill-name">{{ skill.name }}</span>
                <el-tag v-if="skill.status === 'inactive'" type="info" size="small">已停用</el-tag>
                <el-tag v-else type="success" size="small">已启用</el-tag>
                <el-tag :type="skill.type === 'teach' ? 'success' : 'warning'" size="small">
                  {{ skill.type === 'teach' ? '可教' : '想学' }}
                </el-tag>
                <el-tag size="small">{{ getCategoryName(skill.category) }}</el-tag>
              </div>
              <div class="skill-meta">
                <span class="skill-level" v-if="skill.level">{{ skill.level }}</span>
                <span class="skill-desc">{{ skill.description || '暂无描述' }}</span>
              </div>
              <div class="skill-footer">
                <span class="skill-date">发布于 {{ formatTime(skill.createdAt) }}</span>
                <div class="skill-actions">
                  <el-button link type="primary" size="small" @click="toggleSkillStatus(skill)">
                    {{ skill.status === 'inactive' ? '启用' : '停用' }}
                  </el-button>
                  <el-button link type="danger" size="small" @click="deleteSkill(skill)">删除</el-button>
                </div>
              </div>
            </div>
          </div>
        </div>
        <el-empty v-else description="暂无技能" />
      </div>

      <div class="card">
        <h2 class="section-title">⭐ 收到的评价</h2>
        <div v-if="reviews.length" class="reviews-list">
          <div v-for="review in reviews" :key="review.id" class="review-item">
            <div class="review-header">
              <el-avatar :src="review.reviewerAvatar" :size="40" />
              <div class="reviewer-info">
                <span class="reviewer-name">{{ review.reviewerName }}</span>
                <el-rate :model-value="review.rating" disabled size="small" />
              </div>
              <span class="review-time">{{ formatTime(review.createdAt) }}</span>
            </div>
            <p class="review-content">{{ review.comment }}</p>
          </div>
        </div>
        <el-empty v-else description="暂无评价" />
      </div>
    </div>

    <el-dialog v-model="showEditDialog" title="编辑个人资料" width="500px">
      <el-form :model="editForm" label-position="top">
        <el-form-item label="个人简介">
          <el-input v-model="editForm.bio" type="textarea" :rows="3" placeholder="介绍一下自己..." maxlength="200" show-word-limit />
        </el-form-item>
        <el-form-item label="所在城市">
          <el-input v-model="editForm.city" placeholder="城市" />
        </el-form-item>
        <el-form-item label="省份">
          <el-input v-model="editForm.province" placeholder="省份" />
        </el-form-item>
        <el-form-item label="可用时间">
          <el-checkbox-group v-model="editForm.availableTime">
            <el-checkbox value="工作日白天">工作日白天</el-checkbox>
            <el-checkbox value="工作日晚上">工作日晚上</el-checkbox>
            <el-checkbox value="周末白天">周末白天</el-checkbox>
            <el-checkbox value="周末晚上">周末晚上</el-checkbox>
          </el-checkbox-group>
        </el-form-item>
        <el-form-item label="偏好学习方式">
          <el-radio-group v-model="editForm.onlinePreference">
            <el-radio value="online">线上</el-radio>
            <el-radio value="offline">线下</el-radio>
            <el-radio value="both">都可以</el-radio>
          </el-radio-group>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="showEditDialog = false">取消</el-button>
        <el-button type="primary" @click="saveProfile" :loading="saving">保存</el-button>
      </template>
    </el-dialog>

    <el-dialog v-model="showCategoryDialog" title="批量修改分类" width="400px">
      <el-form label-position="top">
        <el-form-item label="选择分类">
          <el-select v-model="targetCategory" placeholder="请选择分类" style="width: 100%">
            <el-option v-for="cat in categories" :key="cat.id" :label="cat.name" :value="cat.id" />
          </el-select>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="showCategoryDialog = false">取消</el-button>
        <el-button type="primary" @click="handleBatchCategory" :loading="batchLoading">确定</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useUserStore } from '../stores/user'
import { skillAPI, reviewAPI } from '../api'
import { ElMessage, ElMessageBox } from 'element-plus'
import dayjs from 'dayjs'
import { Edit, Plus } from '@element-plus/icons-vue'

const userStore = useUserStore()
const mySkills = ref([])
const reviews = ref([])
const categories = ref([])
const showEditDialog = ref(false)
const saving = ref(false)
const batchLoading = ref(false)
const showCategoryDialog = ref(false)
const targetCategory = ref('')
const selectedIds = ref([])

const filters = ref({
  type: '',
  category: '',
  status: '',
  sortBy: 'createdAt_desc'
})

const dateRange = ref([])

const editForm = ref({
  bio: '',
  city: '',
  province: '',
  availableTime: [],
  onlinePreference: 'both'
})

const filteredSkills = computed(() => {
  return mySkills.value
})

const isAllSelected = computed(() => {
  return filteredSkills.value.length > 0 && selectedIds.value.length === filteredSkills.value.length
})

const isIndeterminate = computed(() => {
  return selectedIds.value.length > 0 && selectedIds.value.length < filteredSkills.value.length
})

onMounted(async () => {
  await loadCategories()
  await loadSkills()
  await loadReviews()
  initEditForm()
})

function initEditForm() {
  const prefs = userStore.user?.preferences || {}
  editForm.value = {
    bio: userStore.user?.bio || '',
    city: prefs.location?.city || '',
    province: prefs.location?.province || '',
    availableTime: prefs.time || [],
    onlinePreference: prefs.onlinePreference || 'both'
  }
}

async function loadCategories() {
  const res = await skillAPI.getCategories()
  categories.value = res.data
}

async function loadSkills() {
  const params = {
    userId: userStore.user.id,
    sortBy: filters.value.sortBy
  }
  if (filters.value.type) params.type = filters.value.type
  if (filters.value.category) params.category = filters.value.category
  if (filters.value.status) params.status = filters.value.status
  if (dateRange.value && dateRange.value.length === 2) {
    params.startDate = dateRange.value[0]
    params.endDate = dateRange.value[1]
  }
  const res = await skillAPI.getSkills(params)
  mySkills.value = res.data
  selectedIds.value = []
}

function onDateChange() {
  loadSkills()
}

function getCategoryName(categoryId) {
  const cat = categories.value.find(c => c.id === categoryId)
  return cat ? cat.name : categoryId
}

async function loadReviews() {
  try {
    const res = await reviewAPI.getReviews(userStore.user.id)
    reviews.value = res.data
  } catch (e) {}
}

async function saveProfile() {
  try {
    saving.value = true
    await userStore.updateProfile({
      bio: editForm.value.bio,
      preferences: {
        location: {
          city: editForm.value.city,
          province: editForm.value.province
        },
        time: editForm.value.availableTime,
        onlinePreference: editForm.value.onlinePreference
      }
    })
    ElMessage.success('保存成功')
    showEditDialog.value = false
  } catch (e) {
    ElMessage.error('保存失败')
  } finally {
    saving.value = false
  }
}

function formatTime(time) {
  return dayjs(time).format('YYYY-MM-DD')
}

function toggleSelect(id, checked) {
  if (checked) {
    if (!selectedIds.value.includes(id)) {
      selectedIds.value.push(id)
    }
  } else {
    selectedIds.value = selectedIds.value.filter(sid => sid !== id)
  }
}

function handleCheckAll(checked) {
  if (checked) {
    selectedIds.value = filteredSkills.value.map(s => s.id)
  } else {
    selectedIds.value = []
  }
}

async function toggleSkillStatus(skill) {
  const newStatus = skill.status === 'inactive' ? 'active' : 'inactive'
  const action = newStatus === 'active' ? '启用' : '停用'
  try {
    await skillAPI.updateSkill(skill.id, { status: newStatus })
    ElMessage.success(`${action}成功`)
    loadSkills()
  } catch (e) {
    ElMessage.error(`${action}失败`)
  }
}

async function deleteSkill(skill) {
  try {
    await ElMessageBox.confirm('确定要删除这个技能吗？', '提示', {
      type: 'warning'
    })
    await skillAPI.deleteSkill(skill.id)
    ElMessage.success('删除成功')
    loadSkills()
  } catch (e) {
    if (e !== 'cancel') {
      ElMessage.error('删除失败')
    }
  }
}

async function handleBatchStatus(status) {
  const action = status === 'active' ? '启用' : '停用'
  try {
    await ElMessageBox.confirm(`确定要${action}选中的 ${selectedIds.value.length} 个技能吗？`, '提示', {
      type: 'warning'
    })
    batchLoading.value = true
    await skillAPI.batchUpdateStatus(selectedIds.value, status)
    ElMessage.success(`批量${action}成功`)
    loadSkills()
  } catch (e) {
    if (e !== 'cancel') {
      ElMessage.error(`批量${action}失败`)
    }
  } finally {
    batchLoading.value = false
  }
}

async function handleBatchDelete() {
  try {
    await ElMessageBox.confirm(`确定要删除选中的 ${selectedIds.value.length} 个技能吗？删除后无法恢复。`, '提示', {
      type: 'warning',
      confirmButtonText: '确定删除',
      confirmButtonClass: 'el-button--danger'
    })
    batchLoading.value = true
    await skillAPI.batchDelete(selectedIds.value)
    ElMessage.success('批量删除成功')
    loadSkills()
  } catch (e) {
    if (e !== 'cancel') {
      ElMessage.error('批量删除失败')
    }
  } finally {
    batchLoading.value = false
  }
}

async function handleBatchCategory() {
  if (!targetCategory.value) {
    ElMessage.warning('请选择分类')
    return
  }
  try {
    batchLoading.value = true
    await skillAPI.batchChangeCategory(selectedIds.value, targetCategory.value)
    ElMessage.success('批量修改分类成功')
    showCategoryDialog.value = false
    targetCategory.value = ''
    loadSkills()
  } catch (e) {
    ElMessage.error('批量修改分类失败')
  } finally {
    batchLoading.value = false
  }
}
</script>

<style scoped>
.profile {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.profile-header-card {
  padding: 32px;
}

.profile-header {
  display: flex;
  align-items: center;
  gap: 24px;
}

.profile-info {
  flex: 1;
}

.username {
  font-size: 28px;
  font-weight: 700;
  color: #333;
  margin-bottom: 8px;
}

.user-meta {
  display: flex;
  align-items: center;
  gap: 12px;
  color: #666;
  font-size: 14px;
  margin-bottom: 12px;
}

.rating {
  font-weight: 600;
  color: #ff9800;
}

.divider {
  color: #ddd;
}

.bio {
  color: #666;
  line-height: 1.6;
}

.profile-grid {
  display: grid;
  grid-template-columns: 1.5fr 1fr;
  gap: 24px;
}

.skills-card {
  display: flex;
  flex-direction: column;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.section-title {
  font-size: 18px;
  font-weight: 600;
  color: #333;
  margin: 0;
}

.header-actions {
  display: flex;
  gap: 8px;
}

.filter-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
  padding: 16px;
  background: #f8f9fa;
  border-radius: 8px;
  margin-bottom: 16px;
}

.filter-item {
  display: flex;
  align-items: center;
  gap: 8px;
}

.filter-label {
  font-size: 13px;
  color: #666;
  white-space: nowrap;
}

.batch-actions {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  background: #e8f4fd;
  border-radius: 8px;
  margin-bottom: 16px;
}

.selected-count {
  font-size: 13px;
  color: #667eea;
  font-weight: 500;
}

.skills-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.skill-item {
  display: flex;
  gap: 12px;
  padding: 16px;
  background: #fafafa;
  border-radius: 8px;
  transition: all 0.2s;
  border: 2px solid transparent;
}

.skill-item:hover {
  background: #f0f2ff;
  border-color: #e0e7ff;
}

.skill-item.inactive {
  opacity: 0.6;
}

.skill-main {
  flex: 1;
  min-width: 0;
}

.skill-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
  flex-wrap: wrap;
}

.skill-name {
  font-weight: 600;
  color: #333;
  font-size: 15px;
}

.skill-meta {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
}

.skill-level {
  background: #e3f2fd;
  color: #1565c0;
  padding: 2px 10px;
  border-radius: 10px;
  font-size: 12px;
}

.skill-desc {
  color: #666;
  font-size: 13px;
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.skill-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.skill-date {
  font-size: 12px;
  color: #999;
}

.skill-actions {
  display: flex;
  gap: 8px;
}

.reviews-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.review-item {
  padding: 16px;
  background: #fafafa;
  border-radius: 8px;
}

.review-header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 12px;
}

.reviewer-info {
  flex: 1;
}

.reviewer-name {
  font-weight: 600;
  color: #333;
  display: block;
  margin-bottom: 4px;
}

.review-time {
  font-size: 12px;
  color: #999;
}

.review-content {
  color: #666;
  line-height: 1.6;
  margin: 0;
}

@media (max-width: 1024px) {
  .profile-grid {
    grid-template-columns: 1fr;
  }

  .filter-bar {
    gap: 12px;
  }
}
</style>
