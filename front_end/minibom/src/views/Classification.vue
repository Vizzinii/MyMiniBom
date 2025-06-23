<template>
  <div class="page-wrapper">
    <!-- 顶部操作区 -->
    <div class="top-action-bar">
      <h2 class="page-title">分类管理</h2>
      <div class="actions">
        <!-- 搜索表单 -->
        <el-form :inline="true" class="search-form">
          <el-form-item label="名称/编码：">
            <el-input v-model="searchQuery" placeholder="请输入分类名称或编码" @keyup.enter="onSearch" />
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="onSearch">搜索</el-button>
            <el-button @click="onReset">重置</el-button>
          </el-form-item>
        </el-form>
        <el-button class="add-button" @click="onAddClassification">
          <el-icon><Plus /></el-icon><span>添加分类</span>
        </el-button>
      </div>
    </div>
    <!-- 分类表格 -->
    <el-table
        v-loading="loading"
        :data="classificationList"
        style="width: 100%"
        row-key="id"
        :tree-props="{children: 'children', hasChildren: 'hasChildren'}"
    >
      <el-table-column label="名称" prop="name" />
      <el-table-column label="编码" prop="code" />
      <el-table-column label="英文名称" prop="englishName" />
      <el-table-column label="描述" prop="description" />
      <el-table-column label="操作" width="200">
        <template #default="{ row }">
          <el-button :icon="Edit" circle plain type="primary" @click="onEditClassification(row)" title="编辑"></el-button>
          <el-button :icon="Link" circle plain type="success" @click="onLinkAttributes(row)" title="关联属性"></el-button>
          <el-button :icon="Delete" circle plain type="danger" @click="onDeleteClassification(row)" title="删除"></el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 添加/编辑分类 对话框 -->
    <el-dialog :title="dialogTitle" v-model="dialogVisible" width="50%" @close="onDialogClose">
      <el-form :model="classificationForm" :rules="formRules" ref="formRef" label-width="100px">
        <el-form-item label="编码" prop="code">
          <el-input v-model="classificationForm.code" placeholder="请输入编码"></el-input>
        </el-form-item>
        <el-form-item label="名称" prop="name">
          <el-input v-model="classificationForm.name" placeholder="请输入名称"></el-input>
        </el-form-item>
        <el-form-item label="英文名称" prop="englishName">
          <el-input v-model="classificationForm.englishName" placeholder="请输入英文名称"></el-input>
        </el-form-item>
        <el-form-item label="描述" prop="description">
          <el-input v-model="classificationForm.description" type="textarea" placeholder="请输入描述"></el-input>
        </el-form-item>
        <el-form-item label="父分类" prop="parentId">
          <el-select v-model="classificationForm.parentId" placeholder="请选择父分类 (不选则为根分类)" clearable>
            <el-option v-for="item in flatClassificationList" :key="item.id" :label="item.name" :value="item.id" />
          </el-select>
        </el-form-item>
      </el-form>
      <template #footer>
                <span class="dialog-footer">
                    <el-button @click="dialogVisible = false">取消</el-button>
                    <el-button type="primary" @click="onSubmit">确认</el-button>
                </span>
      </template>
    </el-dialog>

    <!-- 关联属性 对话框 (统一列表，已关联默认勾选) -->
    <el-dialog title="关联属性" v-model="linkDialogVisible" width="60%">
      <!-- 测试标题：显示当前状态 -->
      <div style="background: #f0f9ff; padding: 10px; margin-bottom: 15px; border-radius: 4px;">
        <h4 style="color: #1890ff; margin: 0;">🔧 调试信息面板</h4>
        <p style="margin: 5px 0;">总属性数量: {{ allAttributesList.length }}</p>
        <p style="margin: 5px 0;">已关联属性数量: {{ currentLinkedAttributeIds.length }}</p>
        <p style="margin: 5px 0;">当前选中数量: {{ selectedAttributeIds.length }}</p>
        <p style="margin: 5px 0;">已关联的属性ID: {{ currentLinkedAttributeIds.join(', ') || '无' }}</p>
      </div>

      <div class="attribute-list-container">
        <el-tag v-if="allAttributesList.length === 0" type="info">暂无可用属性</el-tag>
        <el-checkbox-group v-model="selectedAttributeIds">
          <el-checkbox
              v-for="attr in allAttributesList"
              :key="attr.id"
              :label="attr.id"
              border
              :class="[
                            'attribute-checkbox',
                            { 'linked-attribute': currentLinkedAttributeIds.includes(attr.id) }
                        ]"
          >
            {{ attr.name }} ({{ attr.nameEn }})
            <el-tag v-if="currentLinkedAttributeIds.includes(attr.id)" size="small" type="success" class="linked-tag">
              已关联
            </el-tag>
          </el-checkbox>
        </el-checkbox-group>
      </div>

      <template #footer>
                <span class="dialog-footer">
                    <el-button @click="linkDialogVisible = false">取消</el-button>
                    <el-button type="primary" @click="onLinkSubmit">确认关联</el-button>
                </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue'
import { Edit, Delete, Link, Plus } from '@element-plus/icons-vue'
import {
  classificationListService,
  classificationCreateService,
  classificationUpdateService,
  classificationDeleteService,
  classificationLinkAttrService,
  classificationDetailService
} from '@/api/classificationAPI.js'
import { attributeQueryService } from '@/api/attributeAPI.js' // 使用正确的 attributeQueryService
import { ElMessage, ElMessageBox } from 'element-plus'

// --- Data ---
const loading = ref(false)
const classificationList = ref([])
const searchQuery = ref('')
const flatClassificationList = ref([])

// --- Dialog Control ---
const dialogVisible = ref(false)
const dialogTitle = ref('')
const formRef = ref(null)
const classificationForm = ref({
  id: '', code: '', name: '', englishName: '', description: '', parentId: ''
})

const formRules = {
  code: [{ required: true, message: '请输入编码', trigger: 'blur' }],
  name: [{ required: true, message: '请输入名称', trigger: 'blur' }],
  englishName: [{ required: true, message: '请输入英文名称', trigger: 'blur' }]
}

// --- Link Attributes Dialog ---
const linkDialogVisible = ref(false)
const allAttributesList = ref([]) // 所有属性列表
const selectedAttributeIds = ref([]) // 当前选中的属性ID列表（包含已关联和新选择的）
const currentClassificationId = ref('')
const currentLinkedAttributeIds = ref([]) // 当前已关联的属性ID列表

// --- Methods ---

const flattenTree = (tree) => {
  return tree.reduce((acc, node) => {
    const { children, ...rest } = node;
    acc.push(rest);
    if (children && children.length > 0) {
      acc.push(...flattenTree(children));
    }
    return acc;
  }, []);
};

const getClassificationList = async () => {
  loading.value = true
  const params = {}
  if (searchQuery.value) {
    params.name = searchQuery.value
  }
  try {
    const res = await classificationListService(params)
    // 修正: 新的拦截器会直接返回核心数据
    classificationList.value = res
    flatClassificationList.value = flattenTree(classificationList.value)
  } catch (error) {
    // 错误信息已由拦截器统一处理
    console.error(error)
  } finally {
    loading.value = false
  }
}

onMounted(getClassificationList)

const onSearch = () => getClassificationList()
const onReset = () => {
  searchQuery.value = ''
  getClassificationList()
}

const onAddClassification = () => {
  dialogTitle.value = '添加分类'
  dialogVisible.value = true
}

const onEditClassification = (row) => {
  dialogTitle.value = '编辑分类'
  dialogVisible.value = true
  nextTick(() => {
    classificationForm.value = { ...row }
  })
}

const onDialogClose = () => {
  formRef.value.resetFields()
  classificationForm.value = { id: '', code: '', name: '', englishName: '', description: '', parentId: '' }
}

const onSubmit = async () => {
  await formRef.value.validate()
  const isEdit = !!classificationForm.value.id
  try {
    if (isEdit) {
      await classificationUpdateService(classificationForm.value.id, classificationForm.value)
      ElMessage.success('更新成功')
    } else {
      await classificationCreateService(classificationForm.value)
      ElMessage.success('添加成功')
    }
    dialogVisible.value = false
    getClassificationList()
  } catch (error) {
    ElMessage.error(isEdit ? '更新失败' : '添加失败')
    console.error(error)
  }
}

const onDeleteClassification = async (row) => {
  try {
    await ElMessageBox.confirm('您确认要删除该分类吗？其子分类将一并删除。', '温馨提示', {
      type: 'warning', confirmButtonText: '确认', cancelButtonText: '取消'
    })
    await classificationDeleteService(row.id)
    ElMessage.success('删除成功')
    getClassificationList()
  } catch (error) {
    if (error !== 'cancel') {
      ElMessage.error('删除失败')
      console.error(error)
    }
  }
}

const onLinkAttributes = async (row) => {
  currentClassificationId.value = row.id;
  linkDialogVisible.value = true;

  // 清空旧数据
  allAttributesList.value = [];
  selectedAttributeIds.value = [];

  try {
    const [allAttrRes, classificationDetailRes] = await Promise.all([
      attributeQueryService(),
      classificationDetailService(row.id)
    ]);

    // 设置所有属性列表
    allAttributesList.value = allAttrRes || [];

    // 设置已关联的属性ID为默认选中状态
    const linkedIds = classificationDetailRes?.attributes?.map(attr => attr.id) || [];
    selectedAttributeIds.value = [...linkedIds];

    // 保存已关联的属性ID，用于后续计算新增的属性
    currentLinkedAttributeIds.value = [...linkedIds];

    console.log('所有属性:', allAttributesList.value);
    console.log('已关联属性ID:', linkedIds);
    console.log('默认选中ID:', selectedAttributeIds.value);

  } catch (error) {
    console.error("获取属性或分类详情失败", error);
    ElMessage.error('获取属性数据失败');
  }
};

const onLinkSubmit = async () => {
  // 计算新增的属性ID：当前选中的 - 已关联的
  const newAttributeIds = selectedAttributeIds.value.filter(id =>
      !currentLinkedAttributeIds.value.includes(id)
  );

  // 如果没有新增的属性，则不需要调用后端接口
  if (newAttributeIds.length === 0) {
    ElMessage.info('没有新的属性需要关联');
    linkDialogVisible.value = false;
    return;
  }

  const data = {
    classificationId: currentClassificationId.value,
    attributeIds: newAttributeIds
  };

  console.log('已关联的属性ID:', currentLinkedAttributeIds.value);
  console.log('当前选中的属性ID:', selectedAttributeIds.value);
  console.log('新增的属性ID:', newAttributeIds);
  console.log('提交的关联数据:', data);

  try {
    await classificationLinkAttrService(data);
    ElMessage.success('关联成功');
    linkDialogVisible.value = false;
    getClassificationList();
  } catch (error) {
    // 错误已由拦截器处理
    console.error('关联失败:', error);
  }
};
</script>

<style scoped>
.page-wrapper {
  position: relative;
  z-index: 1;
}

.top-action-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.page-title {
  font-size: 28px;
  font-weight: 600;
  color: #333;
}

.actions {
  display: flex;
  align-items: center;
  gap: 16px;
}

.search-form {
  display: flex;
  gap: 10px;
}
.search-form :deep(.el-form-item) {
  margin-bottom: 0;
}
.search-form :deep(.el-form-item__label) {
  color: #606266;
  font-weight: 600;
}

/* 按钮样式 */
.tech-button, .search-button, .reset-button, .add-button {
  border-radius: 6px;
  font-weight: 600;
  transition: all 0.3s ease;
  border: none;
  padding: 10px 20px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.search-button, .add-button {
  background-color: #409EFF;
  color: white;
}
.search-button:hover, .add-button:hover {
  background-color: #66b1ff;
  box-shadow: 0 4px 8px rgba(64, 158, 255, 0.2);
}

.reset-button {
  background-color: #f5f7fa;
  color: #606266;
  border: 1px solid #dcdfe6;
}

.reset-button:hover {
  background-color: #e4e7ed;
}

.add-button {
  display: flex;
  align-items: center;
  gap: 8px;
  border: none;
  font-weight: 600;
  color: #000;
  background: linear-gradient(90deg, #00bfff 0%, #0affa7 100%);
  transition: all 0.3s ease;
}
.add-button:hover {
  filter: brightness(1.2);
  box-shadow: 0 0 15px rgba(10, 255, 167, 0.6);
}

/* 表格样式 */
.tech-table {
  background: transparent !important;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid #e4e7ed;
}
.tech-table :deep(th.el-table__cell) {
  background: rgba(0, 191, 255, 0.15) !important;
  color: #333;
  font-weight: 600;
  border-bottom: 1px solid #409eff;
}
.tech-table :deep(td.el-table__cell) {
  border-bottom: 1px solid #e4e7ed;
}
.tech-table :deep(tr) {
  background: transparent !important;
  color: #606266;
  transition: background-color 0.3s;
}
.tech-table :deep(tr:hover > td.el-table__cell) {
  background: rgba(0, 191, 255, 0.1) !important;
}
.tech-table :deep(.el-table__inner-wrapper::before) {
  display: none;
}

/* 表格内操作按钮 */
.action-buttons {
  display: flex;
  gap: 10px;
  justify-content: center;
}
.action-button {
  font-size: 18px;
  transition: all 0.2s;
  padding: 4px;
}
.action-button.edit { color: #409eff; }
.action-button.delete { color: #ff6b6b; }
.action-button:hover {
  transform: scale(1.2);
  filter: drop-shadow(0 0 5px currentColor);
}
</style>

<!-- 全局样式 -->
<style>
.el-message-box.tech-messagebox {
  background: #fff !important;
  border: 1px solid #e4e7ed !important;
}
.tech-messagebox .el-message-box__title,
.tech-messagebox .el-message-box__content {
  color: #333 !important;
}
</style>
