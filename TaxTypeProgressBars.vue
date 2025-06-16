<template>
  <div class="flex flex-col gap-s4">
    <strong>{{ title }}</strong>
    <div v-if="progressBarSum">
      <div v-for="(items, taxType) in percentageOfSumPerTaxType" :key="taxType">
        <div class="flex flex-row items-center gap-s4 p-s3">
          <span class="col-3">{{ items.taxType?.title }}</span>
          <div class="col flex-grow">
            <PrimeProgressBar
              :value="items.percentageOfTotal"
              :showValue="false"
              class="h-s3"
              :dt="progressBarInProgressToken"
            />
          </div>
          <span class="col-3">{{ items.percentageOfTotal }} %</span>
        </div>
      </div>
    </div>
    <div v-else>
      <div v-for="(items, taxType) in countTaxTypeElligibility" :key="taxType">
        <div class="flex flex-row items-center gap-s4 p-s3">
          <span class="col-3">{{ items.title }}</span>
          <div class="col flex-grow">
            <PrimeProgressBar
              :value="items.elligibleAccounts"
              :showValue="false"
              class="h-s3"
              :dt="progressBarInProgressToken"
            />
          </div>
          <span class="col-3">{{ items.percentage }} %</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { ProgressBarDesignTokens } from '@primevue/themes/aura/progressbar'
import { storeToRefs } from 'pinia'
import { computed, ref } from 'vue'

import PrimeProgressBar from '@/components/shared/Primevue/PrimeProgressBar.vue'
import type Account from '@/models/EUTaxonomy/Account'
import type { EvaluationGroup } from '@/models/EUTaxonomy/EvaluationGroup'
import type Type from '@/models/EUTaxonomy/Type'
import EUTaxonomyService from '@/services/EUTaxonomy/EUTaxonomyService'
import { t } from '@/services/LanguageService'
import { useEUTaxonomyStore } from '@/store/EUTaxonomyStore'

withDefaults(
  defineProps<{
    title: string
    statustext?: string
    readonly?: boolean
    progressBarSum?: boolean
    dt?: ProgressBarDesignTokens
    evaluationGroup?: EvaluationGroup
  }>(),
  { readonly: false, progressBarSum: false },
)

const euTaxonomyService = new EUTaxonomyService()
const { accounts, chosenAnswers, chosenTypeValues } = storeToRefs(useEUTaxonomyStore())
const rawResults = computed<Account[]>(() => accounts.value ?? [])

const groupedByTaxType = computed<Record<string, Account[]>>(() => {
  return rawResults.value.reduce(
    (acc, item) => {
      const taxType = item.taxTypeKey
      if (!taxType) return acc
      if (!acc[taxType]) {
        acc[taxType] = []
      }
      acc[taxType].push(item)
      return acc
    },
    {} as Record<string, Account[]>,
  )
})

const statusDetailsPerTaxType = computed<Record<string, { isTaxonomyElligible: boolean; title: string }[]>>(() => {
  return Object.entries(groupedByTaxType.value).reduce(
    (acc, [taxType, accounts]) => {
      acc[taxType] = accounts.map(account => {
        if (!account.evaluationGroup) {
          return {
            isTaxonomyElligible: false,
            title: account.taxType?.title || '',
          }
        }
        const filteredChosenAnswers = euTaxonomyService.filterChosenAnswersByEvaluationGroup(
          chosenAnswers.value,
          account.evaluationGroup,
        )

        const isTaxonomyCompliant = euTaxonomyService.isEvaluationGroupTaxonomyCompliant(
          filteredChosenAnswers,
          account.evaluationGroup.questions,
        )

        return {
          isTaxonomyElligible: filteredChosenAnswers.length === 0 || isTaxonomyCompliant,
          title: account.taxType?.title || '',
        }
      })
      return acc
    },
    {} as Record<string, { isTaxonomyElligible: boolean; title: string }[]>,
  )
})

const countTaxTypeElligibility = computed<
  {
    elligibleAccounts: number
    totalAccounts: number
    title: string
    percentage: number
  }[]
>(() => {
  return Object.entries(statusDetailsPerTaxType.value).map(([, details]) => {
    const totalAccounts = details.length
    const title = details[0].title
    const elligibleAccounts = details.filter(detail => detail.isTaxonomyElligible).length
    const percentage = totalAccounts > 0 ? parseFloat(((elligibleAccounts / totalAccounts) * 100).toFixed(2)) : 0
    return {
      elligibleAccounts,
      totalAccounts,
      title,
      percentage,
    }
  })
})
const defaultTaxType: Type = {
  title: t('modul16-defaultTaxType'),
  description: '',
  tooltip: '',
  key: '',
}

const percentageOfSumPerTaxType = computed<
  {
    taxType?: Type
    accountSum: number
    percentageOfTotal: number
  }[]
>(() => {
  return chosenTypeValues.value.map(item => {
    if (!item.taxTypeKey) {
      return {
        accountSum: 0,
        taxTypeKey: undefined,
        percentageOfTotal: 0,
      }
    }
    const accountGroup = groupedByTaxType.value[item.taxTypeKey] || []
    const accountSum = accountGroup.reduce((carry, item) => carry + item.amount, 0)
    const taxType = accountGroup.length > 0 && accountGroup[0].taxType ? accountGroup[0].taxType : defaultTaxType

    return {
      taxType,
      accountSum,
      percentageOfTotal: (accountSum / item.total) * 100,
    }
  })
})

const progressBarInProgressToken = ref<ProgressBarDesignTokens>({
  value: {
    background: 'var(--neutral-60)',
  },
})
</script>
