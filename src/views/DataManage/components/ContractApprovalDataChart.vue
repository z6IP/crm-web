<template>
  <div id="approvalTradeBar" style="height: 100%; width: 100%"></div>
</template>

<script lang="ts" name="ContractApprovalDataChart" setup>
import * as echarts from 'echarts'
import { onMounted, onUnmounted, PropType, ref, watch } from 'vue'
import { TradeArray } from '@/api/interface'

// 接收父组件传递的审核趋势数据（与CustomerDataChart保持一致的Prop结构）
const props = defineProps({
  data: {
    type: Object as PropType<TradeArray>,
    required: true,
    default: () => ({ timeList: [], countList: [] })
  }
})

// 存储echarts图表实例，用于销毁、重绘
const myChart = ref<echarts.ECharts | null>(null)

// 初始化图表核心方法
const initChart = () => {
  // 校验数据：时间列表和数量列表长度一致且非空
  if (props.data.timeList.length === 0 || props.data.countList.length === 0) {
    console.warn('审核统计数据为空，无法渲染图表')
    return
  }

  const chartDom = document.getElementById('approvalTradeBar')
  if (!chartDom) return

  // 销毁已有实例，避免重复渲染导致内存泄漏
  if (myChart.value) {
    myChart.value.dispose()
  }

  // 初始化echarts实例
  myChart.value = echarts.init(chartDom)

  // 图表配置项（适配审核统计场景，保持与CustomerDataChart视觉统一）
  const option: echarts.EChartsOption = {
    // 提示框：鼠标悬浮显示详细数据
    tooltip: {
      trigger: 'axis',
      axisPointer: {
        type: 'cross',
        crossStyle: { color: '#999', width: 1 }
      },
      textStyle: { fontSize: 12 },
      padding: 10
    },
    // 图例：区分柱状图和折线图系列
    legend: {
      data: ['审核总数（柱状）', '审核趋势（折线）'],
      top: 0,
      textStyle: { fontSize: 12 }
    },
    // 网格：调整图表边距，避免内容溢出
    grid: {
      left: '3%',
      right: '4%',
      bottom: '8%',
      containLabel: true
    },
    // X轴：时间轴（日/周/月/自定义）
    xAxis: [
      {
        type: 'category',
        data: props.data.timeList,
        axisPointer: { type: 'shadow' },
        // 坐标轴标签：避免文字重叠，自动旋转
        axisLabel: {
          interval: 0,
          rotate: props.data.timeList.length > 10 ? 30 : 0, // 时间点超过10个时旋转30度
          fontSize: 11
        },
        axisLine: { lineStyle: { color: '#eaeaea' } }
      }
    ],
    // Y轴：审核数量（只允许非负整数）
    yAxis: [
      {
        type: 'value',
        name: '审核数量',
        nameTextStyle: { fontSize: 11, color: '#666' },
        min: 0, // 最小值为0，避免负数显示
        axisLabel: {
          fontSize: 11,
          formatter: '{value}' // 整数显示，无小数
        },
        axisLine: { lineStyle: { color: '#eaeaea' } },
        splitLine: { lineStyle: { color: '#f5f5f5' } }
      }
    ],
    // 系列：柱状图（显示总数）+ 折线图（显示趋势）
    series: [
      {
        name: '审核总数（柱状）',
        type: 'bar',
        data: props.data.countList,
        itemStyle: {
          color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
            { offset: 0, color: '#1989fa' },
            { offset: 1, color: '#40a9ff' }
          ]),
          borderRadius: 4 // 柱状图圆角，优化视觉
        },
        barWidth: '40%', // 柱状图宽度，避免过宽或过窄
        label: {
          show: props.data.countList.length <= 15, // 数据点≤15个时显示数值标签
          position: 'top',
          fontSize: 10,
          color: '#333'
        }
      },
      {
        name: '审核趋势（折线）',
        type: 'line',
        data: props.data.countList,
        itemStyle: { color: '#f56c6c' }, // 折线图颜色（红色系，突出趋势）
        lineStyle: { width: 2, type: 'solid' },
        symbol: 'circle', // 标记点样式：圆形
        symbolSize: 4, // 标记点大小
        emphasis: {
          symbolSize: 6 // 鼠标悬浮时标记点放大
        },
        areaStyle: {
          // 折线图下方填充色（渐变透明），增强视觉效果
          color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
            { offset: 0, color: 'rgba(245, 108, 108, 0.2)' },
            { offset: 1, color: 'rgba(245, 108, 108, 0)' }
          ])
        }
      }
    ]
  }

  // 渲染图表
  myChart.value.setOption(option)

  // 窗口大小变化时，重绘图表适配布局
  const handleResize = () => {
    myChart.value?.resize()
  }
  window.addEventListener('resize', handleResize)

  // 组件卸载时清理资源
  onUnmounted(() => {
    window.removeEventListener('resize', handleResize)
    myChart.value?.dispose()
  })
}

// 监听父组件数据变化，数据更新时重新渲染图表
watch(
  () => props.data,
  (newData) => {
    // 确保新数据有效才初始化
    if (newData.timeList.length > 0 && newData.countList.length > 0) {
      initChart()
    }
  },
  { immediate: true } // 组件初始化时立即执行一次
)

// 组件挂载完成后初始化图表（兼容初始数据加载时机）
onMounted(() => {
  if (props.data.timeList.length > 0 && props.data.countList.length > 0) {
    initChart()
  }
})
</script>
