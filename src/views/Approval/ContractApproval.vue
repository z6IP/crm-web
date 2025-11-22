<template>
  <div class="table-box">
    <ProTable
      ref="proTable"
      title="合同列表"
      :columns="columns"
      :requestApi="ContractApi.page"
      :initParam="initParam"
      :dataCallback="dataCallback"
      :searchCol="{ xs: 2, sm: 3, md: 4, lg: 6, xl: 8 }"
    >
      <!-- 操作栏按钮 -->
      <template #operation="scope">
        <el-button type="success" link :icon="CircleCheckFilled" v-hasPermi="['sys:contract:pass']" @click="openApprovalDialog(scope.row, 0)"> 审核通过 </el-button>
        <el-button type="danger" link :icon="CircleCloseFilled" v-hasPermi="['sys:contract:reject']" @click="openApprovalDialog(scope.row, 1)"> 审核不通过 </el-button>
      </template>
    </ProTable>

    <!-- 审核意见输入弹窗 -->
    <el-dialog v-model="isApprovalDialogOpen" :title="dialogTitle" width="550px" :append-to-body="true" destroy-on-close>
      <el-form :model="approvalForm" :rules="approvalRules" ref="approvalFormRef" label-width="90px">
        <el-form-item label="审核意见" prop="opinion">
          <el-input v-model="approvalForm.opinion" type="textarea" :rows="6" placeholder="请输入审核通过/不通过的具体原因（最多500字）" resize="none" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="isApprovalDialogOpen = false">取消</el-button>
        <el-button type="primary" @click="submitApproval">确认提交</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts" name="ContractManage">
import { reactive, ref, unref } from 'vue'
import { ColumnProps } from '@/components/ProTable/interface'
import ProTable from '@/components/ProTable/index.vue'
import { ContractApi } from '@/api/modules/contract'
import { ContractStatusList } from '@/configs/enum'
import { CircleCheckFilled, CircleCloseFilled } from '@element-plus/icons-vue'
import { useHandleData } from '@/hooks/useHandleData'
import { ElForm } from 'element-plus'

// 获取 ProTable 元素
const proTable = ref()

// 审核弹窗相关状态
const isApprovalDialogOpen = ref(false) // 弹窗显示状态
const dialogTitle = ref('') // 弹窗标题
const approvalFormRef = ref<InstanceType<typeof ElForm>>() // 表单引用
const currentRow = ref<any>(null) // 当前操作的合同数据
const currentType = ref(0) // 审核类型：0=通过，1=不通过

// 审核表单数据
const approvalForm = reactive({
  opinion: '' // 审核意见
})

// 表单校验规则
const approvalRules = reactive({
  opinion: [
    { required: true, message: '请输入审核意见', trigger: 'blur' },
    { max: 500, message: '审核意见不能超过500字', trigger: 'blur' }
  ]
})

// 表格初始化参数
const initParam = reactive({ status: 1 })

// 表格数据处理
const dataCallback = (data: any) => {
  return {
    list: data.list,
    total: data.total
  }
}

// 表格列配置（保持原有逻辑不变）
const columns: ColumnProps[] = [
  { type: 'selection', fixed: 'left', width: 60 },
  {
    prop: 'name',
    label: '合同名称',
    minWidth: 120,
    search: { el: 'input' }
  },
  {
    prop: 'number',
    label: '合同编号',
    minWidth: 120,
    search: { el: 'input' }
  },
  {
    prop: 'customerName',
    label: '客户姓名',
    minWidth: 120,
    search: { el: 'input' }
  },
  {
    prop: 'amount',
    label: '合同金额',
    minWidth: 100,
    formatter: (row: any) => `¥${row.amount.toFixed(2)}`
  },
  {
    prop: 'receivedAmount',
    label: '已收款项',
    minWidth: 140,
    formatter: (row: any) => `¥${row.receivedAmount.toFixed(2)}`
  },
  {
    prop: 'status',
    label: '合同状态',
    minWidth: 120,
    enum: Object.values(ContractStatusList),
    search: { el: 'select' }
  },
  {
    prop: 'signTime',
    label: '签约时间',
    minWidth: 140
  },
  {
    prop: 'startTime',
    label: '合同开始时间',
    minWidth: 140
  },
  {
    prop: 'endTime',
    label: '合同结束时间',
    minWidth: 140
  },
  {
    prop: 'ownerEmail',
    label: '销售邮箱',
    minWidth: 160,
    search: { el: 'input' } // 可选：支持邮箱搜索
  },
  {
    prop: 'operation',
    label: '操作',
    fixed: 'right',
    width: 330
  }
]

// 打开审核弹窗
const openApprovalDialog = (row: any, type: number) => {
  currentRow.value = row
  currentType.value = type
  // 设置弹窗标题
  dialogTitle.value = type === 0 ? '合同审核通过' : '合同审核不通过'
  // 重置表单
  approvalForm.opinion = ''
  if (approvalFormRef.value) {
    approvalFormRef.value.clearValidate()
  }
  // 显示弹窗
  isApprovalDialogOpen.value = true
}

// 提交审核（核心修改：携带审核意见）
const submitApproval = async () => {
  // 表单校验
  const formRef = unref(approvalFormRef)
  if (!formRef) return
  await formRef.validate() // 触发表单校验
  // 调用审核接口，新增 opinion 参数传递审核意见
  await useHandleData(
    ContractApi.approvalContract,
    {
      id: currentRow.value.id,
      type: currentType.value,
      opinion: approvalForm.opinion // 传递审核意见
    },
    currentType.value === 0 ? '合同审核通过' : '合同审核不通过'
  )
  // 操作成功：关闭弹窗 + 刷新表格 + 提示
  isApprovalDialogOpen.value = false
  proTable.value.getTableList()
}
</script>
