<template>
  <div class="calculator-container">
    <el-card class="calculator-card">
      <template #header>
        <div class="card-header">
          <h2>购物计算器</h2>
          <div class="header-buttons">
            <el-button type="danger" @click="clearAll" plain>清空所有</el-button>
          </div>
        </div>
      </template>
      
      <div class="section-header">
        <div class="section-title">必要项目</div>
        <el-button type="success" @click="addItem(true)" plain>添加项目</el-button>
      </div>
      <el-table :data="essentialItems" row-key="id" class="items-table essential-table">
        <el-table-column width="40">
          <template #default="{ row }">
            <el-icon class="drag-handle"><Rank /></el-icon>
          </template>
        </el-table-column>
        
        <el-table-column label="项目名称" prop="name">
          <template #default="{ row }">
            <el-input v-model="row.name"  placeholder="输入项目名称"></el-input>
          </template>
        </el-table-column>

        <el-table-column label="产品型号" prop="model">
          <template #default="{ row }">
            <el-input v-model="row.model"  placeholder="输入产品型号"></el-input>
          </template>
        </el-table-column>
        
        <el-table-column label="数量" width="auto">
          <template #default="{ row }">
            <el-input-number v-model="row.quantity" :min="0" :step="1" @change="validateNumber(row, 'quantity')"></el-input-number>
          </template>
        </el-table-column>
        
        <el-table-column label="单价" width="auto">
          <template #default="{ row }">
            <el-input-number v-model="row.price" :min="0" :precision="2" :step="0.01" @change="validateNumber(row, 'price')"></el-input-number>
          </template>
        </el-table-column>
        
        <el-table-column label="小计" width="auto">
          <template #default="{ row }">
            <span>{{ (row.quantity * row.price).toFixed(2) }}</span>
          </template>
        </el-table-column>
        
        <el-table-column label="操作" width="120">
          <template #default="{ row, $index }">
            <el-popconfirm title="确定删除此项目吗?" @confirm="removeItem(true, $index)" confirm-button-text="确定" cancel-button-text="取消">
              <template #reference>
                <el-button type="danger">删除</el-button>
              </template>
            </el-popconfirm>
          </template>
        </el-table-column>
      </el-table>
      
      <div class="section-total">
        必要项目总计: ¥{{ essentialTotal.toFixed(2) }}
      </div>

      <div class="section-divider"></div>

      <div class="section-header">
        <div class="section-title">非必要项目</div>
        <el-button type="warning" @click="addItem(false)" plain>添加项目</el-button>
      </div>
      <el-table :data="nonEssentialItems" row-key="id" class="items-table non-essential-table">
        <el-table-column width="40">
          <template #default="{ row }">
            <el-icon class="drag-handle"><Rank /></el-icon>
          </template>
        </el-table-column>
        
        <el-table-column label="项目名称" prop="name">
          <template #default="{ row }">
            <el-input v-model="row.name"  placeholder="输入项目名称"></el-input>
          </template>
        </el-table-column>

        <el-table-column label="产品型号" prop="model">
          <template #default="{ row }">
            <el-input v-model="row.model" placeholder="输入产品型号"></el-input>
          </template>
        </el-table-column>
        
        <el-table-column label="数量" width="auto">
          <template #default="{ row }">
            <el-input-number v-model="row.quantity" :min="0" :step="1" @change="validateNumber(row, 'quantity')"></el-input-number>
          </template>
        </el-table-column>
        
        <el-table-column label="单价" width="auto">
          <template #default="{ row }">
            <el-input-number v-model="row.price" :min="0" :precision="2" :step="0.01" @change="validateNumber(row, 'price')"></el-input-number>
          </template>
        </el-table-column>
        
        <el-table-column label="小计" width="auto">
          <template #default="{ row }">
            <span>{{ (row.quantity * row.price).toFixed(2) }}</span>
          </template>
        </el-table-column>
        
        <el-table-column label="操作" width="120">
          <template #default="{ row, $index }">
            <el-popconfirm title="确定删除此项目吗?" @confirm="removeItem(false, $index)" confirm-button-text="确定" cancel-button-text="取消">
              <template #reference>
                <el-button type="danger">删除</el-button>
              </template>
            </el-popconfirm>
          </template>
        </el-table-column>
      </el-table>
      
      <div class="section-total">
        非必要项目总计: ¥{{ nonEssentialTotal.toFixed(2) }}
      </div>

      <div class="final-total-section">
        <h3>最终总计: ¥{{ finalTotal.toFixed(2) }}</h3>
      </div>
    </el-card>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Rank } from '@element-plus/icons-vue'
import 'element-plus/dist/index.css'
import Sortable from 'sortablejs'

const STORAGE_KEY = 'shopping-calculator-items'

const validateNumber = (item, field) => {
  if (item[field] < 0) {
    item[field] = 0.00
    ElMessage.warning(`${field === 'quantity' ? '数量' : '价格'}不能小于0`)
    return false
  }
  // 确保保留两位小数
  item[field] = Number(item[field]).toFixed(2)
  return true
}

const validateItem = (item) => {
  if (!item.name) {
    ElMessage.warning('项目名称为必填项')
    return false
  }
  return true
}

const essentialItems = ref([])
const nonEssentialItems = ref([])

// 从本地存储加载数据
const loadItems = () => {
  const savedData = localStorage.getItem(STORAGE_KEY)
  if (savedData) {
    const { essential, nonEssential } = JSON.parse(savedData)
    essentialItems.value = essential || []
    nonEssentialItems.value = nonEssential || []
  }
}

// 保存数据到本地存储
const saveItems = () => {
  const data = {
    essential: essentialItems.value,
    nonEssential: nonEssentialItems.value
  }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data))
}

// 监听数据变化，自动保存
watch([essentialItems, nonEssentialItems], saveItems, { deep: true })

const addItem = (isEssential) => {
  const newItem = {
    id: Date.now(),
    name: '',
    model: '',
    quantity: 1.00,
    price: 0.00
  }
  
  if (isEssential) {
    essentialItems.value.push(newItem)
    ElMessage.success('已添加必要项目')
  } else {
    nonEssentialItems.value.push(newItem)
    ElMessage.success('已添加非必要项目')
  }
}

const removeItem = (isEssential, index) => {
  if (isEssential) {
    essentialItems.value.splice(index, 1)
  } else {
    nonEssentialItems.value.splice(index, 1)
  }
  ElMessage.success('删除成功')
}

const clearAll = () => {
  ElMessageBox.confirm('此操作将清空所有项目，是否继续?', '提示', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    type: 'warning',
  }).then(() => {
    essentialItems.value = []
    nonEssentialItems.value = []
    ElMessage.success('已清空所有项目')
  }).catch(() => {
    ElMessage.info('已取消清空操作')
  })
}

const essentialTotal = computed(() => {
  return essentialItems.value.reduce((sum, item) => sum + (item.quantity * item.price), 0)
})

const nonEssentialTotal = computed(() => {
  return nonEssentialItems.value.reduce((sum, item) => sum + (item.quantity * item.price), 0)
})

const finalTotal = computed(() => {
  return essentialTotal.value + nonEssentialTotal.value
})

onMounted(() => {
  loadItems()
  
  const setupSortable = (el, items, isEssential) => {
    if (el) {
      Sortable.create(el, {
        handle: '.drag-handle',
        animation: 150,
        group: 'items',
        onAdd({ newIndex, oldIndex, from, to }) {
          const fromItems = from.classList.contains('essential-table') ? essentialItems : nonEssentialItems
          const toItems = to.classList.contains('essential-table') ? essentialItems : nonEssentialItems
          const movedItem = fromItems.value[oldIndex]
          fromItems.value.splice(oldIndex, 1)
          toItems.value.splice(newIndex, 0, movedItem)
        },
        onEnd({ newIndex, oldIndex, from, to }) {
          if (from === to) {
            const itemCopy = items.value[oldIndex]
            items.value.splice(oldIndex, 1)
            items.value.splice(newIndex, 0, itemCopy)
          }
        }
      })
    }
  }

  // 设置两个表格的拖拽功能
  const tables = document.querySelectorAll('.el-table__body-wrapper tbody')
  setupSortable(tables[0], essentialItems, true)
  setupSortable(tables[1], nonEssentialItems, false)
})
</script>

<style scoped>
.calculator-container {
  max-width: 1200px;
  margin: 20px auto;
  padding: 0 20px;
}

.calculator-card {
  margin-bottom: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.header-buttons {
  display: flex;
  gap: 10px;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin: 20px 0 10px;
}

.section-title {
  font-size: 18px;
  font-weight: bold;
  color: #409EFF;
}

.section-divider {
  height: 1px;
  background-color: #EBEEF5;
  margin: 30px 0;
}

.items-table {
  margin-bottom: 20px;
}

.section-total {
  text-align: right;
  margin-top: 10px;
  font-size: 16px;
  color: #606266;
}

.final-total-section {
  text-align: right;
  margin-top: 20px;
  padding-top: 20px;
  border-top: 2px solid #409EFF;
}

.final-total-section h3 {
  color: #409EFF;
  margin: 0;
  font-size: 20px;
}

.drag-handle {
  cursor: move;
}

.el-select {
  width: 100%;
}

.is-error .el-input__wrapper {
  box-shadow: 0 0 0 1px var(--el-color-danger) inset !important;
}

.el-table__empty-block {
  min-height: 100px;
}

.el-table__empty-text {
  color: var(--el-text-color-secondary);
}
</style>
