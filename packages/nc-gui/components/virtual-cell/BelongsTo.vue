<script setup lang="ts">
import type { ColumnType } from 'nocodb-sdk'
import type { Ref } from 'vue'
import {
  ActiveCellInj,
  CellValueInj,
  ColumnInj,
  IsFormInj,
  IsUnderLookupInj,
  ReadonlyInj,
  ReloadRowDataHookInj,
  RowInj,
  computed,
  createEventHook,
  inject,
  ref,
  useProvideLTARStore,
  useRoles,
  useSelectedCellKeyupListener,
  useSmartsheetRowStoreOrThrow,
} from '#imports'

const column = inject(ColumnInj)!

const reloadRowTrigger = inject(ReloadRowDataHookInj, createEventHook())

const cellValue = inject(CellValueInj, ref<any>(null))

const row = inject(RowInj)!

const active = inject(ActiveCellInj)!

const readOnly = inject(ReadonlyInj, ref(false))

const isForm = inject(IsFormInj, ref(false))

const isUnderLookup = inject(IsUnderLookupInj, ref(false))

const { isUIAllowed } = useRoles()

const listItemsDlg = ref(false)

const { state, isNew, removeLTARRef } = useSmartsheetRowStoreOrThrow()

const { relatedTableMeta, loadRelatedTableMeta, relatedTableDisplayValueProp, unlink } = useProvideLTARStore(
  column as Ref<Required<ColumnType>>,
  row,
  isNew,
  reloadRowTrigger.trigger,
)

await loadRelatedTableMeta()

const addIcon = computed(() => (cellValue?.value ? 'expand' : 'plus'))

const value = computed(() => {
  if (cellValue?.value) {
    return cellValue?.value
  } else if (isNew.value) {
    return state?.value?.[column?.value.title as string]
  }
  return null
})

const unlinkRef = async (rec: Record<string, any>) => {
  if (isNew.value) {
    await removeLTARRef(rec, column?.value as ColumnType)
  } else {
    await unlink(rec)
  }
}

useSelectedCellKeyupListener(active, (e: KeyboardEvent) => {
  switch (e.key) {
    case 'Enter':
      listItemsDlg.value = true
      e.stopPropagation()
      break
  }
})

const belongsToColumn = computed(
  () =>
    relatedTableMeta.value?.columns?.find((c: any) => c.title === relatedTableDisplayValueProp.value) as ColumnType | undefined,
)

const plusBtnRef = ref<HTMLElement | null>(null)

watch([listItemsDlg], () => {
  if (!listItemsDlg.value) {
    plusBtnRef.value?.focus()
  }
})
</script>

<template>
  <div class="flex w-full chips-wrapper items-center" :class="{ active }">
    <div class="nc-cell-field chips flex items-center flex-1">
      <template v-if="value && relatedTableDisplayValueProp">
        <VirtualCellComponentsItemChip
          :item="value"
          :value="!Array.isArray(value) && typeof value === 'object' ? value[relatedTableDisplayValueProp] : value"
          :column="belongsToColumn"
          :show-unlink-button="true"
          @unlink="unlinkRef(value)"
        />
      </template>
    </div>

    <div
      v-if="!readOnly && (isUIAllowed('dataEdit') || isForm) && !isUnderLookup"
      ref="plusBtnRef"
      class="flex justify-end group gap-1 min-h-[30px] items-center"
      tabindex="0"
      @keydown.enter.stop="listItemsDlg = true"
    >
      <GeneralIcon
        :icon="addIcon"
        class="text-sm nc-action-icon group-focus:visible invisible text-gray-500/50 hover:text-gray-500 select-none group-hover:(text-gray-500) nc-plus"
        @click.stop="listItemsDlg = true"
      />
    </div>

    <LazyVirtualCellComponentsListItems
      v-if="listItemsDlg"
      v-model="listItemsDlg"
      :column="belongsToColumn"
      @attach-record="listItemsDlg = true"
    />
  </div>
</template>

<style scoped lang="scss">
.nc-action-icon {
  @apply cursor-pointer;
}

.chips-wrapper:hover,
.chips-wrapper.active {
  .nc-action-icon {
    @apply inline-block;
  }
}

.chips-wrapper:hover {
  .nc-action-icon {
    @apply visible;
  }
}
</style>
