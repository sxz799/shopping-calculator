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

      <el-row :gutter="20">
        <el-col :span="12">
          <el-card class="item-card">
            <template #header>
              <div class="card-header">
                <div class="essential-section-title">必要项目</div>
                <el-button type="success" @click="addItem(true)" plain>添加项目</el-button>
              </div>
            </template>
            <el-table :data="essentialItems" row-key="id" class="items-table essential-table"
                      @sort-change="(column) => handleSortChange(essentialItems, column)">
              <el-table-column width="40">
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

              <el-table-column label="数量" width="90">
                <template #default="{ row }">
                  <el-input type="number" v-model="row.quantity" @change="validateNumber(row, 'quantity')"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="单价" prop="price" sortable width="120">
                <template #default="{ row }">
                  <el-input type="number" v-model="row.price" @change="validateNumber(row, 'price')"></el-input>
                </template>
              </el-table-column>


              <el-table-column label="已购" width="60">
                <template #default="{ row }">
                  <el-checkbox v-model="row.purchased"></el-checkbox>
                </template>
              </el-table-column>
              <el-table-column label="操作" :width="80">
                <template #default="{ row, $index }">
                  <el-popconfirm title="确定删除此项目吗?" @confirm="removeItem(true, $index)"
                                 confirm-button-text="确定" cancel-button-text="取消">
                    <template #reference>
                      <el-button size="small" type="danger">删除</el-button>
                    </template>
                  </el-popconfirm>
                </template>
              </el-table-column>
            </el-table>

            <div class="section-total">
              <h3>已支出: ¥{{ essentialTotal }}</h3>
              <h3>未支出: ¥{{
                  essentialItems.reduce((sum, item) => item.purchased ? sum : sum + (item.quantity * item.price), 0)
                }}</h3>
              <h3>全部总计: ¥{{ essentialItems.reduce((sum, item) => sum + (item.quantity * item.price), 0) }}</h3>
            </div>
          </el-card>
        </el-col>

        <el-col :span="12">
          <el-card class="item-card">
            <template #header>
              <div class="card-header">
                <div class="non-essential-section-title">非必要项目</div>
                <el-button type="warning" @click="addItem(false)" plain>添加项目</el-button>
              </div>
            </template>
            <el-table :data="nonEssentialItems" row-key="id" class="items-table non-essential-table"
                      @sort-change="(column) => handleSortChange(nonEssentialItems, column)">
              <el-table-column width="40">
                <template #default="{ row }">
                  <el-icon class="drag-handle">
                    <Rank/>
                  </el-icon>
                </template>
              </el-table-column>

              <el-table-column label="项目名称" prop="name">
                <template #default="{ row }">
                  <el-input v-model="row.name" placeholder="输入项目名称"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="产品型号" prop="model">
                <template #default="{ row }">
                  <el-input v-model="row.model" placeholder="输入产品型号"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="数量" width="90">
                <template #default="{ row }">
                  <el-input type="number" v-model="row.quantity" @change="validateNumber(row, 'quantity')"></el-input>
                </template>
              </el-table-column>

              <el-table-column label="单价" prop="price" sortable width="120">
                <template #default="{ row }">
                  <el-input type="number" v-model="row.price" @change="validateNumber(row, 'price')"></el-input>
                </template>
              </el-table-column>


              <el-table-column label="已购" width="60">
                <template #default="{ row }">
                  <el-checkbox v-model="row.purchased"></el-checkbox>
                </template>
              </el-table-column>
              <el-table-column label="操作" :width="80">
                <template #default="{ row, $index }">
                  <el-popconfirm title="确定删除此项目吗?" @confirm="removeItem(false, $index)"
                                 confirm-button-text="确定" cancel-button-text="取消">
                    <template #reference>
                      <el-button size="small" type="danger">删除</el-button>
                    </template>
                  </el-popconfirm>
                </template>
              </el-table-column>
            </el-table>

            <div class="section-total">
              <h3>已支出: ¥{{ nonEssentialTotal }}</h3>
              <h3>未支出: ¥{{
                  nonEssentialItems.reduce((sum, item) => item.purchased ? sum : sum + (item.quantity * item.price), 0)
                }}</h3>
              <h3>全部总计: ¥{{ nonEssentialItems.reduce((sum, item) => sum + (item.quantity * item.price), 0) }}</h3>

            </div>
          </el-card>
        </el-col>
      </el-row>

      <div class="section-divider"></div>

      <div class="final-total-section">
        <h3>全部总计: ¥{{ totalAllItems }}</h3>
        <h3>已支出总计: ¥{{ finalTotal }}</h3>
        <h3>未支出总计: ¥{{ totalUnpurchasedItems }}</h3>
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
const nonEssentialItems = ref([])

// 处理表格排序
const handleSortChange = (list, {prop, order}) => {
  if (prop === 'price') {
    list.sort((a, b) => {
      if (order === 'ascending') {
        return a.price - b.price;
      } else if (order === 'descending') {
        return b.price - a.price;
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
  // const savedData = localStorage.getItem(STORAGE_KEY)
  // if (savedData) {
  //   const { essential, nonEssential } = JSON.parse(savedData)
  //   essentialItems.value = essential || []
  //   nonEssentialItems.value = nonEssential || []
  // }
  axios.get('https://redis.sxz799.asia/api/get?key=' + STORAGE_KEY)
      .then(response => {
        const {essential, nonEssential} = response.data.value
        essentialItems.value = essential || []
        nonEssentialItems.value = nonEssential || []
      })
      .catch(error => {
        console.error('加载数据失败:', error)
        ElMessage.error('加载数据失败，请稍后重试')
      })
}

// 保存数据到本地存储
const saveItems = () => {
  const data = {
    essential: essentialItems.value,
    nonEssential: nonEssentialItems.value
  }
  // localStorage.setItem(STORAGE_KEY, JSON.stringify(data))
  axios.post('https://redis.sxz799.asia/api/set', {
    key: STORAGE_KEY,
    value: data
  }).then(() => {
        ElMessage.success('数据已保存')
      })
      .catch(error => {
        console.error('保存数据失败:', error)
        ElMessage.error('保存数据失败，请稍后重试')
      })
}

// 监听数据变化，自动保存
watch([essentialItems, nonEssentialItems], saveItems, {deep: true})

onMounted(() => {
  loadItems()
  // 初始化必要项目拖拽
  new Sortable(document.querySelector('.essential-table .el-table__body > tbody'), {
    animation: 150,
    handle: '.drag-handle',
    onEnd: (event) => handleSort(essentialItems.value, event)
  })
  // 初始化非必要项目拖拽
  new Sortable(document.querySelector('.non-essential-table .el-table__body > tbody'), {
    animation: 150,
    handle: '.drag-handle',
    onEnd: (event) => handleSort(nonEssentialItems.value, event)
  })
})

const addItem = (isEssential) => {
  const newItem = {
    id: Date.now(),
    name: '',
    model: '',
    quantity: 1.00,
    price: 0.00,
    purchased: false // 新增的购买状态
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
  return essentialItems.value.reduce((sum, item) => {
    if (item.purchased) {
      return sum + (item.quantity * item.price)
    } else {
      return sum
    }
  }, 0)
})

const nonEssentialTotal = computed(() => {
  return nonEssentialItems.value.reduce((sum, item) => {
    if (item.purchased) {
      return sum + (item.quantity * item.price)
    } else {
      return sum
    }
  }, 0)
})

// 计算所有项目的总价
const totalAllItems = computed(() => {
  const essentialAll = essentialItems.value.reduce((sum, item) => sum + (item.quantity * item.price), 0)
  const nonEssentialAll = nonEssentialItems.value.reduce((sum, item) => sum + (item.quantity * item.price), 0)
  return essentialAll + nonEssentialAll
})

// 计算未购买项目的总价
const totalUnpurchasedItems = computed(() => {
  const essentialUnpurchased = essentialItems.value.reduce((sum, item) => {
    if (!item.purchased) {
      return sum + (item.quantity * item.price)
    } else {
      return sum
    }
  }, 0)
  const nonEssentialUnpurchased = nonEssentialItems.value.reduce((sum, item) => {
    if (!item.purchased) {
      return sum + (item.quantity * item.price)
    } else {
      return sum
    }
  }, 0)
  return essentialUnpurchased + nonEssentialUnpurchased
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
        onAdd({newIndex, oldIndex, from, to}) {
          const fromItems = from.classList.contains('essential-table') ? essentialItems : nonEssentialItems
          const toItems = to.classList.contains('essential-table') ? essentialItems : nonEssentialItems
          const movedItem = fromItems.value[oldIndex]
          fromItems.value.splice(oldIndex, 1)
          toItems.value.splice(newIndex, 0, movedItem)
        },
        onEnd({newIndex, oldIndex, from, to}) {
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

.essential-section-title {
  font-size: 18px;
  font-weight: bold;
  color: rgb(255, 0, 0);
}

.non-essential-section-title {
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
  text-align: center;
  margin-top: 20px;
  padding-top: 20px;
  border-top: 2px solid rgb(126, 195, 162);
}

.final-total-section h3 {
  color: rgb(17, 192, 232);
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
