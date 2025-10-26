<template>
  <div class="table-box">
    <ProTable
      ref="proTable"
      title="商品列表"
      :columns="columns"
      :requestApi="ProductApi.page"
      :initParams="initParam"
      :dataCallback="dataCallback"
      :searchCols="{ xs: 2, sm: 3, md: 4, lg: 6, xl: 8 }"
    ></ProTable>
  </div>
</template>

<script setup lang="ts" name="CustomerManage">
import { reactive, ref } from 'vue'
import { ColumnProps } from '@/components/ProTable/interface'
import ProTable from '@/components/ProTable/index.vue'
import { ProductStatusList } from '@/configs/enum'
import { ProductApi } from '@/api/modules/product'

const proTable = ref()

const initParam = reactive({})

const dataCallback = (data: any) => {
  return {
    list: data.list,
    total: data.total
  }
}

const columns: ColumnProps[] = [
  {
    type: 'selection',
    fixed: 'left',
    width: 60
  },
  {
    prop: 'name',
    label: '商品名称',
    minWidth: 120,
    search: { el: 'input' }
  },
  {
    prop: 'price',
    label: '价格',
    minWidth: 120
  },
  {
    prop: 'sales',
    label: '销量',
    minWidth: 120
  },
  {
    prop: 'stock',
    label: '库存数量',
    minWidth: 120
  },
  {
    prop: 'status',
    label: '商品状态',
    minWidth: 120,
    enum: Object.values(ProductStatusList),
    search: { el: 'select' }
  },
  {
    prop: 'operation',
    label: '操作',
    fixed: 'right',
    width: 230
  }
]
</script>
