<template>
  <ImportMask
    v-if="!readonly"
    :show="isImportMaskVisible"
    :readonly="readonly"
    @close="closeEUTaxonomyExcelImportMask"
  />
  <TaskPageHeaderCard :title="t('modul16d.title')" :description="t('modul16d.description')" customclass="items-end">
    <template #footerLeft>
      <PrimeButton
        variant="secondary"
        :prependIcon="IconVariant.RightLeftLarge"
        class="my-s2"
        @click="openEUTaxonomyExcelImportMask()"
      >
        {{ t('modul16d.import.title') }}
      </PrimeButton>
    </template>
    <template #right>
      <TaxTypeProgressBars :title="t(`modul16d.progressbar.title`)" :progressBarSum="true" />
    </template>
  </TaskPageHeaderCard>
</template>
<script lang="ts" setup>
import { onMounted, onUnmounted, ref } from 'vue'

import ImportMask from '@/components/modul16/ImportMask.vue'
import TaxTypeProgressBars from '@/components/modul16/TaxTypeProgressBars.vue'
import { IconVariant } from '@/components/shared/Icons/IconVariant'
import PrimeButton from '@/components/shared/Primevue/Buttons/PrimeButton.vue'
import TaskPageHeaderCard from '@/components/shared/TaskPageHeader/TaskPageHeaderCard.vue'
import { useExportImport } from '@/composables/ExportImport'
import type Account from '@/models/EUTaxonomy/Account'
import { t } from '@/services/LanguageService'

defineProps<{
  readonly: boolean
  accounts: Account[]
}>()
const importMethodKey = 'eut-excel-import'
const isImportMaskVisible = ref<boolean>(false)
const { registerMethod, unregisterMethod } = useExportImport()

const openEUTaxonomyExcelImportMask = (): void => {
  isImportMaskVisible.value = true
}

const closeEUTaxonomyExcelImportMask = (): void => {
  isImportMaskVisible.value = false
}
onMounted(() => {
  registerMethod(
    {
      key: importMethodKey,
      titel: t('modul16d.import.title'),
      excelIcon: true,
      action: openEUTaxonomyExcelImportMask,
    },
    true,
  )
})

onUnmounted(() => {
  unregisterMethod(importMethodKey)
})
</script>
