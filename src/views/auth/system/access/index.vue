<template>
  <dynamic-table ref="tableRef" @expand="expand" :columns="columns" :get-list-func="getAdminAccess" rowKey="id"
                 :row-selection="rowSelection">
    <template v-slot:title>
      <a-button v-permission="{ action: 'create', effect: 'disabled' }" @click="addItem" type="primary">
        添加
      </a-button>
      <a-button @click="deleteItems" v-permission="{ action: 'delete' }" :disabled="isDisabled" type="primary">
        删除
      </a-button>
    </template>
    <template v-slot:moduleName="{record}">
      <span :ref="el => itemRefs[record.id] = el">
        {{ record.moduleName || record.actionName }}
      </span>
    </template>
  </dynamic-table>
</template>
<script lang="ts">
import {defineComponent, reactive, toRefs, createVNode, render, nextTick, computed, ref} from 'vue'
import {Modal} from 'ant-design-vue'
import {QuestionCircleOutlined, LoadingOutlined} from '@ant-design/icons-vue'
import {DynamicTable} from '@/components/dynamic-table'
import {useCreateModal} from "@/hooks";
import {delAdminAccess, getAdminAccess} from '@/api/system/access'
import AddModal from './add-modal.vue'
import {columns} from "./columns";

export default defineComponent({
  name: 'system-access',
  components: {
    DynamicTable,
  },
  setup() {
    const tableRef = ref<any>(null)

    const state = reactive({
      itemRefs: {},
      expandedRowKeys: [] as string[],
      tableLoading: false,
      rowSelection: {
        onChange: (selectedRowKeys, selectedRows) => {
          state.rowSelection.selectedRowKeys = selectedRowKeys;
        },
        selectedRowKeys: []
      },
    })

    // 删除多项
    const deleteItems = () => {
      Modal.confirm({
        title: '提示',
        icon: createVNode(QuestionCircleOutlined),
        content: '您确定要删除所有选中吗？',
        onOk: async () => {
          await delAdminAccess(state.rowSelection.selectedRowKeys.toString())
          tableRef.value.refreshTableData()
          state.rowSelection.selectedRowKeys = []
        }
      })
    }
    // 添加资源
    const addItem = () => {
      useCreateModal(AddModal, {
        callback: () => {
          tableRef.value.refreshTableData()
        }
      })
    }

    // 是否禁用批量删除按钮
    const isDisabled = computed(() => state.rowSelection.selectedRowKeys.length == 0)

    // 点击展开图标
    const expand = async (expanded, record) => {
      console.log(expanded, record)
      // 如果是第一次展开
      if (expanded && record.children && !Array.isArray(record.children)) {
        const iconEle = state.itemRefs[record.id].parentElement.querySelector('.ant-table-row-expand-icon')
        render(createVNode(LoadingOutlined), iconEle)
        await nextTick()
        iconEle.classList.add('loading-icon')
        const {data} = await getAdminAccess({id: record.id,limit: 100})
        record.children = data
        render(null, iconEle)
        await nextTick()
        iconEle.classList.remove('loading-icon')
      }
    }

    return {
      ...toRefs(state),
      columns,
      tableRef,
      expand,
      getAdminAccess,
      isDisabled,
      addItem,
      deleteItems,
    }
  }
})
</script>

<style lang="scss">
.loading-icon {
  border: none;
  &.ant-table-row-expanded::after {
    content: none;
  }
}
</style>
