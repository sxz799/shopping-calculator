<template>
  <div class="calculator-container">
    <el-card class="calculator-card">
      <template #header>
        <div class="card-header">
          <h2>装修计算器</h2>
        </div>
      </template>

      <el-row>
        <el-col>
          <el-card class="item-card">
            <template #header>
              <div class="card-header">
                <el-button type="success" @click="addItem()" plain>添加项目</el-button>
                <div class="filter-options">
                  <span class="filter-label">必要性:</span>
                  <el-select v-model="essentialFilter" style="width: 120px; margin-right: 10px;">
                    <el-option label="全部" value="all"></el-option>
                    <el-option label="必要" value="essential"></el-option>
                    <el-option label="非必要" value="nonEssential"></el-option>
                  </el-select>
                  <span class="filter-label">购买状态:</span>
                  <el-select v-model="purchaseFilter" style="width: 120px;">
                    <el-option label="全部" value="all"></el-option>
                    <el-option label="已购" value="purchased"></el-option>
                    <el-option label="未购" value="unpurchased"></el-option>
                  </el-select>
                  <div v-if="showFilteredTotal" class="filtered-total">
                    <span>当前筛选金额: ¥{{ filteredTotal }}</span>
                  </div>
                </div>
              </div>
            </template>
            <el-table :data="filteredItems" row-key="id" class="items-table essential-table"
                      @sort-change="(column) => handleSortChange(essentialItems, column)">
              <el-table-column width="40px">
                <template #default>
                  <el-icon class="drag-handle">
                    <Rank/>
                  </el-icon>
                </template>
              </el-table-column>

              <el-table-column label="项目名称" prop="name" width="auto">
                <template #default="{ row }">
                  <el-input v-model="row.name"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="产品型号" prop="model" width="auto">
                <template #default="{ row }">
                  <el-input v-model="row.model"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="位置" prop="location" sortable width="200px">
                <template #default="{ row }">
                  <el-input v-model="row.location"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="数量" width="80px">
                <template #default="{ row }">
                  <el-input type="number" v-model="row.quantity" @change="validateNumber(row, 'quantity')"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="单价" prop="price" sortable width="130px">
                <template #default="{ row }">
                  <el-input type="number" v-model="row.price" @change="validateNumber(row, 'price')"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="是否必要" width="80px">
                <template #default="{ row }">
                  <el-checkbox v-model="row.isEssential"></el-checkbox>
                </template>
              </el-table-column>
              <el-table-column label="购买状态" width="80px">
                <template #default="{ row }">
                  <el-checkbox v-model="row.purchased"></el-checkbox>
                </template>
              </el-table-column>
              <el-table-column label="操作">
                <template #default="{ $index }">
                  <el-popconfirm title="确定删除此项目吗?" @confirm="removeItem($index)"
                                 confirm-button-text="确定" cancel-button-text="取消">
                    <template #reference>
                      <el-button size="small" type="danger">删除</el-button>
                    </template>
                  </el-popconfirm>
                </template>
              </el-table-column>
            </el-table>
          </el-card>
        </el-col>
      </el-row>

      <div class="section-divider"></div>

      <div class="final-total-section">
        <h3>全部总计: ¥{{ totalAllItems }}</h3>
        <h3>已支出总计: ¥{{ finalTotal }}</h3>
        <h3>未支出总计: ¥{{ totalUnpurchasedItems }}</h3>
        <div class="essential-totals">
          <h4>必要项目</h4>
          <p>总计: ¥{{ essentialTotal }}</p>
          <p>已支出: ¥{{ essentialPurchasedTotal }}</p>
          <p>未支出: ¥{{ essentialUnpurchasedTotal }}</p>
        </div>
        <div class="non-essential-totals">
          <h4>非必要项目</h4>
          <p>总计: ¥{{ nonEssentialTotal }}</p>
          <p>已支出: ¥{{ nonEssentialPurchasedTotal }}</p>
          <p>未支出: ¥{{ nonEssentialUnpurchasedTotal }}</p>
        </div>
      </div>

      <div class="location-totals-section">
        <h3>各位置总计:</h3>
        <div v-for="(total, location) in locationTotals" :key="location">
          <p>{{ location || '未指定位置' }}: ¥{{ total }}</p>
        </div>
      </div>
    </el-card>
  </div>
</template>

<script setup>
import {computed, onMounted, ref, watch} from 'vue'
import {ElMessage} from 'element-plus'
import {Rank} from '@element-plus/icons-vue'
import 'element-plus/dist/index.css'
import Sortable from 'sortablejs'
import axios from "axios"

const STORAGE_KEY = 'shopping-calculator-items'

// 过滤选项
const essentialFilter = ref('all')
const purchaseFilter = ref('all')

const validateNumber = (item, field) => {
  if (item[field] < 0) {
    item[field] = 0
    ElMessage.warning(`${field === 'quantity' ? '数量' : '价格'}不能小于0`)
    return false
  }
  return true
}

const essentialItems = ref([])

// 处理表格排序
const handleSortChange = (list, {prop, order}) => {
  if (prop === 'price' || prop === 'location' || prop === 'isEssential') {
    list.sort((a, b) => {
      let valA = a[prop]
      let valB = b[prop]
      
      // 确保price字段使用数值比较
      if (prop === 'price') {
        valA = Number(valA)
        valB = Number(valB)
      }

      if (order === 'ascending') {
        if (typeof valA === 'string' && typeof valB === 'string') {
          return valA.localeCompare(valB)
        } else if (typeof valA === 'boolean' && typeof valB === 'boolean') {
          return valA === valB ? 0 : (valA ? -1 : 1)
        } else {
          return valA - valB
        }
      } else if (order === 'descending') {
        if (typeof valA === 'string' && typeof valB === 'string') {
          return valB.localeCompare(valA)
        } else if (typeof valA === 'boolean' && typeof valB === 'boolean') {
          return valA === valB ? 0 : (valA ? 1 : -1)
        } else {
          return valB - valA
        }
      }
      return 0
    })
  }
}

// 处理拖动排序
const handleSort = (list, event) => {
  const {oldIndex, newIndex} = event
  const movedItem = list.splice(oldIndex, 1)[0]
  list.splice(newIndex, 0, movedItem)
}

// 加载数据
const loadItems = () => {
  axios.get('https://redis.sxz799.asia/api/get?key=' + STORAGE_KEY)
    .then(response => {
      const data = JSON.parse(response.data.value)
      essentialItems.value = data.essential ? data.essential.map(item => ({
        ...item,
        isEssential: item.isEssential !== undefined ? item.isEssential : true
      })) : []
    })
    .catch(error => {
      console.error('加载数据失败:', error)
      ElMessage.error('加载数据失败，请稍后重试')
    })
}

// 保存数据
const saveItems = () => {
  const data = {
    essential: essentialItems.value
  }
  axios.post('https://redis.sxz799.asia/api/set', {
    key: STORAGE_KEY,
    value: JSON.stringify(data)
  }).catch(error => {
    console.error('保存数据失败:', error)
    ElMessage.error('保存数据失败，请稍后重试')
  })
}

function debounce(fn, delay) {
  let timer = null
  return function(...args) {
    clearTimeout(timer)
    timer = setTimeout(() => {
      fn.apply(this, args)
    }, delay)
  }
}

// 监听数据变化，自动保存
watch(essentialItems, debounce(saveItems, 500), {deep: true})

onMounted(() => {
  loadItems()
  // 初始化必要项目拖拽
  new Sortable(document.querySelector('.essential-table .el-table__body > tbody'), {
    animation: 150,
    handle: '.drag-handle',
    onEnd: (event) => handleSort(essentialItems.value, event)
  })
})

const addItem = () => {
  const newItem = {
    id: Date.now(),
    name: '',
    model: '',
    location: '',
    quantity: 1,
    price: 0,
    purchased: false,
    isEssential: false
  }
  essentialItems.value.push(newItem)
  ElMessage.success('项目添加成功')
}

const removeItem = (index) => {
  essentialItems.value.splice(index, 1)
  ElMessage.success('项目删除成功')
}

// 计算总计
const totalAllItems = computed(() => {
  return essentialItems.value
    .reduce((sum, item) => sum + item.quantity * item.price, 0)
    .toFixed(2)
})

const finalTotal = computed(() => {
  return essentialItems.value
    .filter(item => item.purchased)
    .reduce((sum, item) => sum + item.quantity * item.price, 0)
    .toFixed(2)
})

const totalUnpurchasedItems = computed(() => {
  return essentialItems.value
    .filter(item => !item.purchased)
    .reduce((sum, item) => sum + item.quantity * item.price, 0)
    .toFixed(2)
})

const essentialTotal = computed(() => {
  return essentialItems.value
    .filter(item => item.isEssential)
    .reduce((sum, item) => sum + item.quantity * item.price, 0)
    .toFixed(2)
})

const essentialPurchasedTotal = computed(() => {
  return essentialItems.value
    .filter(item => item.isEssential && item.purchased)
    .reduce((sum, item) => sum + item.quantity * item.price, 0)
    .toFixed(2)
})

const essentialUnpurchasedTotal = computed(() => {
  return essentialItems.value
    .filter(item => item.isEssential && !item.purchased)
    .reduce((sum, item) => sum + item.quantity * item.price, 0)
    .toFixed(2)
})

const nonEssentialTotal = computed(() => {
  return essentialItems.value
    .filter(item => !item.isEssential)
    .reduce((sum, item) => sum + item.quantity * item.price, 0)
    .toFixed(2)
})

const nonEssentialPurchasedTotal = computed(() => {
  return essentialItems.value
    .filter(item => !item.isEssential && item.purchased)
    .reduce((sum, item) => sum + item.quantity * item.price, 0)
    .toFixed(2)
})

const nonEssentialUnpurchasedTotal = computed(() => {
  return essentialItems.value
    .filter(item => !item.isEssential && !item.purchased)
    .reduce((sum, item) => sum + item.quantity * item.price, 0)
    .toFixed(2)
})

// 过滤后的商品列表
const filteredItems = computed(() => {
  return essentialItems.value.filter(item => {
    // 必要性过滤
    if (essentialFilter.value === 'essential' && !item.isEssential) return false;
    if (essentialFilter.value === 'nonEssential' && item.isEssential) return false;
    
    // 购买状态过滤
    if (purchaseFilter.value === 'purchased' && !item.purchased) return false;
    if (purchaseFilter.value === 'unpurchased' && item.purchased) return false;
    
    return true;
  });
});

// 是否显示过滤金额
const showFilteredTotal = computed(() => {
  return essentialFilter.value !== 'all' || purchaseFilter.value !== 'all';
});

// 当前过滤项目的总金额
const filteredTotal = computed(() => {
  return filteredItems.value
    .reduce((sum, item) => sum + item.quantity * item.price, 0)
    .toFixed(2);
});

const locationTotals = computed(() => {
  const totals = {}
  essentialItems.value.forEach(item => {
    const location = item.location || '未指定位置'
    if (!totals[location]) {
      totals[location] = 0
    }
    totals[location] += item.quantity * item.price
  })
  for (const location in totals) {
    totals[location] = totals[location].toFixed(2)
  }
  return totals
})
</script>

<style scoped>
.calculator-container {
  padding: 0;
}

.calculator-card {
  margin-bottom: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.filter-options {
  display: flex;
  align-items: center;
}

.filter-label {
  margin-right: 5px;
  font-size: 14px;
  color: #606266;
}

.filtered-total {
  margin-left: 15px;
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

.final-total-section {
  text-align: center;
  margin-top: 20px;
  padding-top: 20px;
  border-top: 2px solid rgb(126, 195, 162);
}

.final-total-section h3 {
  color: rgb(17, 192, 232);
  margin: 10px 0;
  font-size: 20px;
}

.essential-totals,
.non-essential-totals {
  border: 1px solid #EBEEF5;
  padding: 10px;
  margin-bottom: 10px;
  border-radius: 4px;
}

.essential-totals h4,
.non-essential-totals h4 {
  color: #409EFF;
  margin-top: 0;
  margin-bottom: 5px;
  font-size: 18px;
}

.essential-totals p,
.non-essential-totals p {
  margin: 2px 0;
  color: #606266;
}

.location-totals-section {
  text-align: center;
  margin-top: 20px;
  padding-top: 20px;
  border-top: 2px solid #EBEEF5;
}

.location-totals-section h3 {
  color: #606266;
  margin-bottom: 10px;
}

.location-totals-section p {
  margin: 5px 0;
  color: #606266;
}

.drag-handle {
  cursor: move;
}

.el-select {
  width: 100%;
}
</style>
