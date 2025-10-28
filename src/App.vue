<template>
  <div class="calculator-container">
    <el-card class="calculator-card">
      <template #header>
        <div class="card-header">
          <h2>装修计算器</h2>
        
        </div>
      </template>

      <el-row :gutter="20">
        <el-col >
          <el-card class="item-card">
            <template #header>
              <div class="card-header">
                <el-button type="success" @click="addItem()" plain>添加项目</el-button>
              </div>
            </template>
            <el-table :data="essentialItems" row-key="id" class="items-table essential-table"
                      @sort-change="(column) => handleSortChange(essentialItems, column)">
              <el-table-column width="40px">
                <template #default="{ row }">
                  <el-icon class="drag-handle">
                    <Rank/>
                  </el-icon>
                </template>
              </el-table-column>

              <el-table-column label="项目名称" prop="name" width="auto">
                <template #default="{ row }">
                  <el-input v-model="row.name" placeholder="输入项目名称"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="产品型号" prop="model" width="auto">
                <template #default="{ row }">
                  <el-input v-model="row.model" placeholder="输入产品型号"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="位置" prop="location" sortable width="200px">
                <template #default="{ row }">
                  <el-input v-model="row.location" placeholder="输入位置"></el-input>
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


              <el-table-column label="必要" sortable width="60px">
                <template #default="{ row }">
                  <el-checkbox v-model="row.isEssential"></el-checkbox>
                </template>
              </el-table-column>
              <el-table-column label="已购" sortable width="60px">
                <template #default="{ row }">
                  <el-checkbox v-model="row.purchased"></el-checkbox>
                </template>
              </el-table-column>
              <el-table-column label="操作" >
                <template #default="{ row, $index }">
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
import {ElMessage, ElMessageBox} from 'element-plus'
import {Rank} from '@element-plus/icons-vue'
import 'element-plus/dist/index.css'
import Sortable from 'sortablejs'
import axios from "axios";

const STORAGE_KEY = 'shopping-calculator-items'

const validateNumber = (item, field) => {
  if (item[field] < 0) {
    item[field] = 0
    ElMessage.warning(`${field === 'quantity' ? '数量' : '价格'}不能小于0`)
    return false
  }
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

// 处理表格排序
const handleSortChange = (list, {prop, order}) => {
  if (prop === 'price' || prop === 'location' || prop === 'isEssential') {
    list.sort((a, b) => {
      const valA = a[prop];
      const valB = b[prop];

      if (order === 'ascending') {
        if (typeof valA === 'string' && typeof valB === 'string') {
          return valA.localeCompare(valB);
        } else if (typeof valA === 'boolean' && typeof valB === 'boolean') {
          return valA === valB ? 0 : (valA ? -1 : 1);
        } else {
          return valA - valB;
        }
      } else if (order === 'descending') {
        if (typeof valA === 'string' && typeof valB === 'string') {
          return valB.localeCompare(valA);
        } else if (typeof valA === 'boolean' && typeof valB === 'boolean') {
          return valA === valB ? 0 : (valA ? 1 : -1);
        } else {
          return valB - valA;
        }
      } else {
        return 0;
      }
    });
  }
};

// 处理拖动排序
const handleSort = (list, event) => {
  const {oldIndex, newIndex} = event
  const movedItem = list.splice(oldIndex, 1)[0]
  list.splice(newIndex, 0, movedItem)
}

// 从本地存储加载数据
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

// 保存数据到本地存储
const saveItems = () => {
  const data = {
    essential: essentialItems.value
  }
  axios.post('https://redis.sxz799.asia/api/set', {
    key: STORAGE_KEY,
    value: JSON.stringify(data)
  }).then(() => {
    // ElMessage.success('数据已保存') // 避免频繁提示
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

const debouncedSaveItems = debounce(saveItems, 2000)

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
    isEssential: true // Default to essential
  }
  essentialItems.value.push(newItem)
  ElMessage.success('项目添加成功')
}

const removeItem = (index) => {
  essentialItems.value.splice(index, 1)
  ElMessage.success('项目删除成功')
}


const essentialTotal = computed(() => {
  return essentialItems.value
      .filter(item => item.isEssential)
      .reduce((sum, item) => sum + item.quantity * item.price, 0)
      .toFixed(2)
})

const nonEssentialTotal = computed(() => {
  return essentialItems.value
      .filter(item => !item.isEssential)
      .reduce((sum, item) => sum + item.quantity * item.price, 0)
      .toFixed(2)
})

const finalTotal = computed(() => {
  return essentialItems.value
      .filter(item => item.purchased)
      .reduce((sum, item) => sum + item.quantity * item.price, 0)
      .toFixed(2)
})

const totalAllItems = computed(() => {
  return essentialItems.value
      .reduce((sum, item) => sum + item.quantity * item.price, 0)
      .toFixed(2)
})

const totalUnpurchasedItems = computed(() => {
  return essentialItems.value
      .filter(item => !item.purchased)
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


.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin: 20px 0 10px;
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
