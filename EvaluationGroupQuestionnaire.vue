<template>
  <Questionnaire
    :readonly="readonly"
    v-if="evaluationGroup.questions.length > 0"
    :dialogState="isQuestionnaireStarted"
    :evaluationGroup="evaluationGroup"
    @close="isQuestionnaireStarted = false"
  />
  <div class="white-rounded-container p-5">
    <div class="flex">
      <div class="col">
        <div class="headline fw-bold">
          {{ evaluationGroup.title }}
        </div>
        <div v-for="tag in tags" :key="tag" class="font-weight-bold col-head">
          <PrimeChip variant="secondary">
            {{ tag }}
          </PrimeChip>
        </div>
        <div class="description py-4">{{ evaluationGroup.description }}</div>
        <div class="flex">
          <div class="col">
            <div class="font-weight-bold col-head">
              {{ t('modul16e.zugeordnete-wirtschaftstaetigkeit') }}
            </div>
            {{ evaluationGroup.economicActivity?.title }}
          </div>
          <div class="col">
            <div class="font-weight-bold col-head">
              {{ t('modul16e.zugeordnete-ziel') }}
            </div>
            {{ evaluationGroup.goal?.text }}
          </div>
          <div class="col">
            <div class="font-weight-bold col-head">
              {{ t('modul16e.ergebnis') }}
            </div>
            <div>
              <span v-if="!hasEvaluationGroupAnsweredQuestions" class="result-text neutral-box">{{
                t('modul16e.elligible-for-taxonomy')
              }}</span>
              <span v-else-if="!isEvaluationGroupTaxonomyCompliant" class="result-text red-box">{{
                t('modul16e.not-elligible-for-taxonomy')
              }}</span>
              <span v-else class="result-text green-box">{{ t('modul16e.taxonomiekonform') }}</span>
            </div>
          </div>
        </div>
      </div>
      <div class="align-items-baseline flex text-end">
        <div class="col pl-2">
          <MultiSelectOwnerAvatar
            :readonly="readonly"
            :owners="evaluationGroup?.owners ?? []"
            :groupedUsers="true"
            :showCurrentUserAvatarFirst="true"
            @selectedOwner="value => handleAddOwner(value)"
            @deletedOwner="value => handleRemoveOwner(value)"
          />
        </div>
        <div class="col pl-2">
          <StatusIndicatorSelect
            :modelValue="evaluationGroup?.status ?? ProcessingStatus.Pending"
            @update:modelValue="(value: ProcessingStatus) => updateEvaluationGroup(evaluationGroup, { status: value })"
            :readonly="readonly"
          />
        </div>
      </div>
    </div>
    <div class="font-weight-bold col-head py-4 text-start">
      <PrimeButton
        v-if="props.evaluationGroup.questions.length > 0"
        :disabled="readonly"
        variant="secondary"
        @click="startQuestionnaire"
      >
        {{ t('modul16e.konformitaetspruefung-starten') }}
      </PrimeButton>
    </div>
  </div>
</template>

<script lang="ts" setup>
import Questionnaire from '@/components/modul16/Questionnaire.vue'
import PrimeButton from '@/components/shared/Primevue/Buttons/PrimeButton.vue'
import PrimeChip from '@/components/shared/Primevue/Chips/PrimeChip.vue'
import MultiSelectOwnerAvatar from '@/components/shared/Selects/MultiSelectOwnerAvatar.vue'
import StatusIndicatorSelect from '@/components/shared/StatusIndicatorSelect.vue'
import { useOwner } from '@/composables/Owner'
import useOwnership from '@/composables/Ownership'
import config from '@/config'
import { AnswerPositions } from '@/models/enums/EUTaxonomy/AnswerPositions'
import type ChosenAnswer from '@/models/EUTaxonomy/ChosenAnswer'
import type { EvaluationGroup } from '@/models/EUTaxonomy/EvaluationGroup'
import type Question from '@/models/EUTaxonomy/Question'
import { ProcessingStatus } from '@/models/ProcessingStatus'
import type User from '@/models/User'
import EUTaxonomyService from '@/services/EUTaxonomy/EUTaxonomyService'
import { t } from '@/services/LanguageService'
import { useEUTaxonomyStore } from '@/store/EUTaxonomyStore'
import { storeToRefs } from 'pinia'
import { computed, ref } from 'vue'

const { updateEvaluationGroup } = useEUTaxonomyStore()
const { addOwner, removeOwner } = useOwnership(
  config.apiEndpoints.euTaxonomy.addEvaluationGroupOwnership,
  config.apiEndpoints.euTaxonomy.removeEvaluationGroupOwnership,
)
const { deleteOwnerWithConfirmation } = useOwner()
const euTaxonomyService = new EUTaxonomyService()

const props = withDefaults(
  defineProps<{
    evaluationGroup: EvaluationGroup
    passedReadonly: boolean
  }>(),
  { passedReadonly: false },
)

const readonly = computed(() => euTaxonomyService.isReadonly(props.evaluationGroup?.owners))
const { chosenAnswers } = storeToRefs(useEUTaxonomyStore())
const isQuestionnaireStarted = ref(false)

const handleRemoveOwner = async (user: User): Promise<void> => {
  if (!props.evaluationGroup) return

  await deleteOwnerWithConfirmation(
    user,
    t('modul16e.ownership-title') + ' ' + props.evaluationGroup?.title,
    removeOwner.bind(window, props.evaluationGroup, user),
  )
}

const handleAddOwner = async (user: User): Promise<void> => {
  if (!props.evaluationGroup) return
  await addOwner(props.evaluationGroup, user)
}

const startQuestionnaire = () => {
  isQuestionnaireStarted.value = true
}

const isEvaluationGroupTaxonomyCompliant = computed(() => {
  if (applicableAnswers.value.length !== totalAnswerableQuestions.value.length) return false
  return !applicableAnswers.value.some(
    (chosenAnswer: ChosenAnswer) =>
      chosenAnswer?.answer?.position === AnswerPositions.no && !chosenAnswer.question?.tag?.noable,
  )
})

const tags = computed(() => {
  return [
    ...new Set(
      props.evaluationGroup.questions.reduce<string[]>((acc, question) => {
        const title = question?.tag?.title
        if (title) {
          acc.push(title)
        }
        return acc
      }, []),
    ),
  ]
})

const totalAnswerableQuestions = computed(() =>
  props.evaluationGroup.questions.filter((question: Question) => question.answers?.length > 0),
)
const applicableAnswers = computed(() =>
  chosenAnswers.value.filter(
    (chosenAnswer: ChosenAnswer) =>
      chosenAnswer.answerKey &&
      chosenAnswer.evaluationGroupKey === props.evaluationGroup.key &&
      props.evaluationGroup.questions.some((question: Question) => chosenAnswer.questionKey === question.key),
  ),
)
const hasEvaluationGroupAnsweredQuestions = computed(() => applicableAnswers.value.length > 0)
</script>
<style lang="scss" scoped>
.white-rounded-container {
  background-color: white;
  border-radius: 20px;
  margin-bottom: var(--spacing-8);
  padding: var(--spacing-4);
}
.headline {
  font-size: 18px;
}
.v-list-item__content {
  word-break: break-all;
  text-align: left !important;
}
.col-head {
  min-height: 50px;
}
.result-text {
  display: inline-block;
  text-align: center;
  border-radius: var(--border-radius-lg);
  padding: var(--spacing-4);

  &.green-box {
    border: 1px solid var(--success-background-light);
    background-color: var(--success-background-light);
    color: var(--success-text);
  }
  &.red-box {
    border: 1px solid var(-error-background-light);
    background-color: var(--error-background-light);
    color: var(--error-text);
  }
  &.neutral-box {
    border: 1px solid var(--neutral-20);
    background-color: var(--neutral-20);
  }
}
</style>
