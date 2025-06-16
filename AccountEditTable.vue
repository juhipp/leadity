<template>
  <div v-if="selectedItems.length > 0" class="flex items-center justify-between">
    <span class="small">{{ tc('modul16d.amount.chosen.accounts', selectedItems.length) }}</span>
    <div class="w-1/2 py-s6">
      <PrimeSelect
        :id="'accEvaluationGroup-' + selectedItems"
        v-model="chosenMassUpdateEvaluationGroup"
        labelType="float"
        :readonly="readonly"
        :label="t('modul16d.accounts-edit.evaluationGroup')"
        :items="evaluationGroups ?? []"
        :titleGetter="item => item.title ?? ''"
        @update:modelValue="value => updateAccountsEvaluationGroup(value ? (value as EvaluationGroup) : undefined)"
      />
    </div>
  </div>
  <BaseDataTable
    :items="accounts"
    :columns="columns"
    :scrollable="true"
    scrollHeight="auto"
    :readonly="readonly"
    :defaultSelectedItems="selectedItems"
    dataKey="key"
    selectionMode="multiple"
    @updateSelection="handleSelectionUpdate"
  >
  </BaseDataTable>
</template>

<script lang="ts" setup>
import { storeToRefs } from 'pinia'
import { computed, type ComputedRef, ref } from 'vue'

import PrimeSelect from '@/components/shared/Primevue/Selects/PrimeSelect.vue'
import BaseDataTable from '@/components/shared/Table/BaseDataTable.vue'
import type { ColDef } from '@/components/shared/Table/models'
import type { SelectionDropdownCellConfig } from '@/components/shared/Table/Renderers/SelectionDropdownCell.vue'
import type Account from '@/models/EUTaxonomy/Account'
import type { EvaluationGroup } from '@/models/EUTaxonomy/EvaluationGroup'
import type HierarchyLevel from '@/models/EUTaxonomy/HierarchyLevel'
import type Type from '@/models/EUTaxonomy/Type'
import { t, tc } from '@/services/LanguageService'
import { doConfirm } from '@/statics/Alerts'
import { useEUTaxonomyStore } from '@/store/EUTaxonomyStore'

const { evaluationGroups, individualMappings, types } = storeToRefs(useEUTaxonomyStore())
const { updateAccount, deleteAccount } = useEUTaxonomyStore()
const chosenMassUpdateEvaluationGroup = ref<EvaluationGroup>()

const props = withDefaults(
  defineProps<{
    readonly?: boolean
    accounts: Account[]
    allAccounts: Account[]
    hierarchyLevel?: HierarchyLevel
  }>(),
  { readonly: false },
)

const individualMappingFieldName = [
  'first_individual_mapping_value',
  'second_individual_mapping_value',
  'third_individual_mapping_value',
] as const

const individualMappingFieldKey = [
  'firstIndividualMappingKey',
  'secondIndividualMappingKey',
  'thirdIndividualMappingKey',
] as const

const dynamicColumns: ComputedRef<ColDef<Account>[]> = computed(() => {
  const dynamicColumns: ColDef<Account>[] = applicableIndividualMapping.value.map((individualMapping, index) => {
    const fieldName = individualMappingFieldName[index]
    const fieldKey = individualMappingFieldKey[index]
    const column: ColDef<Account> = {
      field: fieldName,
      header: individualMapping.title,
      type: 'input',
      rendererConfig: {
        valueChangeFn: (data: Account, newValue: string) => {
          const account: Partial<Account> = { [fieldKey]: individualMapping.key, [fieldName]: newValue }
          updateAccount(data, account)
        },
      },
    }
    return column
  })
  if (applicableParentAccounts.value.length > 0) {
    dynamicColumns.push({
      field: 'parent',
      header: t('modul16d.accounts-edit.parentAccount'),
      type: 'selection',
      rendererConfig: {
        options: applicableParentAccounts.value,
        valueChangeFn: (data: Account, newValue: Account | null) =>
          updateAccount(data, { parentKey: newValue?.key ?? null, parent: newValue } as Partial<Account>),
      } as SelectionDropdownCellConfig<Account>,
    })
  }

  return dynamicColumns
})

const columns: ComputedRef<ColDef<Account>[]> = computed(() => {
  return [
    {
      field: 'external_identifier',
      header: t('modul16d.accounts-edit.external-identifier'),
      type: 'input',
      rendererConfig: {
        valueChangeFn: (data: Account, newValue: string) =>
          updateAccount(data, { external_identifier: newValue } as Partial<Account>),
      },
    },
    {
      field: 'title',
      header: t('modul16d.accounts-edit.title'),
      type: 'input',
      rendererConfig: {
        valueChangeFn: (data: Account, newValue: string) =>
          updateAccount(data, { title: newValue } as Partial<Account>),
      },
    },
    {
      field: 'description',
      header: t('modul16d.accounts-edit.description'),
      type: 'input',
      rendererConfig: {
        valueChangeFn: (data: Account, newValue: string) =>
          updateAccount(data, { description: newValue } as Partial<Account>),
      },
    },
    {
      field: 'taxType',
      header: t('modul16d.accounts-edit.types'),
      type: 'selection',
      rendererConfig: {
        options: types.value,
        valueChangeFn: (data: Account, value: Type | null) =>
          updateAccount(data, { taxTypeKey: value?.key ?? null, taxType: value }),
      } as SelectionDropdownCellConfig<Account>,
    },
    {
      field: 'amount',
      header: t('modul16d.accounts-edit.amount'),
      type: 'number',
      rendererConfig: {
        valueChangeFn: (data: Account, newValue: number) =>
          updateAccount(data, { amount: newValue } as Partial<Account>),
      },
    },
    {
      field: 'evaluationGroup',
      header: t('modul16d.accounts-edit.evaluationGroup'),
      type: 'selection',
      rendererConfig: {
        options: evaluationGroups.value,
        valueChangeFn: async (data: Account, newValue: EvaluationGroup | null): Promise<void> => {
          await updateAccount(data, {
            evaluationGroupKey: newValue?.key ?? null,
            evaluationGroup: newValue,
          })
        },
      },
    },
    ...dynamicColumns.value,
    {
      field: 'actions',
      header: t('Aktion'),
      type: 'actions',
      rendererConfig: { actions: [{ name: t('delete'), action: deleteAccount }] },
    },
  ]
})

const applicableParentAccounts = computed(() => {
  return props.allAccounts.filter(account => account.hierarchyLevel.key === props.hierarchyLevel?.parentKey)
})

const applicableIndividualMapping = computed(() =>
  individualMappings.value.filter(individualMapping => individualMapping.show_in_preview),
)
const selectedItems = ref<Account[]>([])

const handleSelectionUpdate = (value: Account[]): void => {
  selectedItems.value = value
}
const updateAccountsEvaluationGroup = async (selectedEvaluationGroup: EvaluationGroup | undefined): Promise<void> => {
  const accountsWithEvaluationGroup = selectedItems.value.filter(account => account.evaluationGroup)
  if (accountsWithEvaluationGroup.length > 0) {
    const response = await doConfirm(t('modul16d.accounts-edit.change-evaluation-group'))
    if (!response) return
  }
  selectedItems.value.forEach(account => {
    updateAccount(account, {
      evaluationGroupKey: selectedEvaluationGroup?.key ?? null,
      evaluationGroup: selectedEvaluationGroup,
    })
  })
}
</script>
