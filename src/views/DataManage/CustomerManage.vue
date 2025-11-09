<template>
  <div class="container-box">
    <div class="select-box">
      <span>筛选：</span>
      <el-select v-model="selectedItem" placeholder="Select" style="width: 240px; margin-left: 20px" @change="clearSelectedItem()">
        <el-option v-for="item in selectCondition" :key="item.value" :label="item.label" :value="item.value" />
      </el-select>

      <el-date-picker
        v-model="checkedValue"
        :type="selectedItem"
        range-separator="至"
        start-placeholder="开始时间"
        end-placeholder="结束时间"
        placeholder="请选择统计时间"
        v-if="selectedItem !== 'day' && selectedItem !== 'week'"
        style="margin-left: 20px"
        unlink-panels
        :disabled-date="disableFutureDates"
      />

      <div v-if="selectedItem !== 'day' && selectedItem === 'week'">
        <!-- 周选择器，v-model 绑定的数据模型不同 -->
        <el-date-picker
          v-model="startWeekValue"
          :type="selectedItem"
          format="[第] ww [周]"
          placeholder="请选择开始周"
          style="margin-left: 20px"
          :disabled-date="disableFutureDates"
        />
        <span style="margin-left: 20px">至</span>
        <el-date-picker
          v-model="endWeekValue"
          :type="selectedItem"
          format="[第] ww [周]"
          placeholder="请选择结束周"
          style="margin-left: 20px"
          :disabled-date="disableEndWeekDates"
        />
      </div>

      <el-button type="primary" @click="getStatisData" style="margin-left: 20px"> 搜索 </el-button>
    </div>

    <div style="height: 80vh; width: 100%">
      <CustomerDataChart :data="tradeData || {}" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue'
import CustomerDataChart from './components/CustomerDataChart.vue'
import { CustomerApi } from '@/api/modules/customer'
import { TradeArray } from '@/api/interface'
import moment from 'moment'
import { getFormattedDateRange } from '@/hooks/useMergeTime'
import { ElMessage } from 'element-plus'

// 选中的统计类型（日/周/月/自定义）
const selectedItem = ref('day')
// 自定义日期/月选择的时间（时间范围数组）
const checkedValue = ref<Array<Date>>([])
// 周选择的开始时间
const startWeekValue = ref<Date>()
// 周选择的结束时间
const endWeekValue = ref<Date>()

// 统计类型下拉选项
const selectCondition = ref([
  { label: '日客户数据统计', value: 'day' },
  { label: '周客户数据统计', value: 'week' },
  { label: '月客户数据统计', value: 'monthrange' },
  { label: '自定义时间客户数据统计', value: 'daterange' }
])

// 图表展示的趋势数据
const tradeData = ref<TradeArray>({
  timeList: [],
  countList: []
})

// 接口响应类型定义
interface TradeResponse extends IResponse {
  data: TradeArray
}

// 接口请求参数类型定义
interface TradeParams {
  transactionType: string
  timeRange?: [string, string]
}

// 初始化加载数据
const initData = async () => {
  try {
    const res = (await CustomerApi.trendData({
      transactionType: selectedItem.value
    })) as TradeResponse
    tradeData.value = res.data
  } catch (error) {
    console.error('获取客户统计数据失败:', error)
  }
}

// 组件挂载时初始化数据
onMounted(() => {
  initData()
})

// 禁止选择未来的日期
const disableFutureDates = (time: Date) => {
  const now = Date.now()
  // 计算当前时间当天结束的时间戳（23:59:59）
  const endOfToday = new Date(now)
  endOfToday.setHours(23, 59, 59, 999)
  // 日期在今天之后则禁用
  return time.getTime() > endOfToday.getTime()
}

// 结束周不能早于开始周，且不能是未来日期
const disableEndWeekDates = (date: Date) => {
  if (!startWeekValue.value) {
    return date > new Date()
  }
  const start = new Date(startWeekValue.value)
  return date < start || date > new Date()
}

// 切换统计类型时清空已选时间
const clearSelectedItem = () => {
  checkedValue.value = []
  startWeekValue.value = undefined
  endWeekValue.value = undefined
}

// 点击搜索获取统计数据
const getStatisData = async () => {
  try {
    let param: TradeParams = {
      transactionType: selectedItem.value
    }

    // 校验时间选择
    if (
      (checkedValue.value.length === 0 && (selectedItem.value === 'daterange' || selectedItem.value === 'monthrange')) ||
      (startWeekValue.value === undefined && endWeekValue.value === undefined && selectedItem.value === 'week')
    ) {
      ElMessage({
        type: 'warning',
        message: '请选择有效的时间范围'
      })
      return
    }

    // 处理月统计参数
    if (selectedItem.value === 'monthrange') {
      const [start, end] = checkedValue.value
      const startMonth = moment(start).format('YYYY-MM-01 00:00:00')
      const endMonth = moment(end).endOf('month').format('YYYY-MM-DD 23:59:59')
      param = { ...param, timeRange: [startMonth, endMonth] }
    }
    // 处理自定义日期统计参数
    else if (selectedItem.value === 'daterange') {
      const [start, end] = checkedValue.value
      const startDay = moment(start).format('YYYY-MM-DD 00:00:00')
      const endDay = moment(end).format('YYYY-MM-DD 23:59:59')
      param = { ...param, timeRange: [startDay, endDay] }
    }
    // 处理周统计参数
    else if (selectedItem.value === 'week') {
      if (startWeekValue.value instanceof Date && endWeekValue.value instanceof Date) {
        const [startWeek] = getFormattedDateRange(startWeekValue.value)
        const [, endWeek] = getFormattedDateRange(endWeekValue.value)
        param = { ...param, timeRange: [startWeek, endWeek] }
      } else {
        console.warn('请选择有效的周时间')
        return
      }
    }

    // 发起请求并更新图表数据
    const res = (await CustomerApi.trendData(param)) as TradeResponse
    tradeData.value = res.data
  } catch (error) {
    console.error('获取交易统计数据失败:', error)
  }
}
</script>

<style>
.container-box {
  display: flex;
  flex-direction: column;
  justify-content: space-around;
  align-items: center;
  width: 100%;
  height: 100%;
}

.select-box {
  display: flex;
  justify-content: space-around;
  align-items: center;
  height: 5vh;
}
</style>
