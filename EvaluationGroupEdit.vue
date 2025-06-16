<template>
  <div class="d-flex col align-items-baseline justify-end px-2">
    <h5 class="grow py-6">{{ t('modul16c.bewertungsgruppen-header') }}</h5>
    <div class="col-1 pl-2 pt-1 text-end">
      <MultiSelectOwnerAvatar
        :readonly="readonly"
        :owners="evaluationGroup?.owners ?? []"
        :groupedUsers="true"
        :showCurrentUserAvatarFirst="true"
        @selectedOwner="value => handleAddOwner(value)"
        @deletedOwner="value => handleRemoveOwner(value)"
      />
    </div>
    <div class="col-1 min-w-32 pl-2 pt-1 text-end">
      <StatusIndicatorSelect
        :modelValue="evaluationGroup?.status ?? ProcessingStatus.Pending"
        @update:modelValue="
          (value: ProcessingStatus) => evaluationGroup && updateEvaluationGroup(evaluationGroup, { status: value })
        "
        :readonly="readonly"
      />
    </div>
  </div>
  <div class="d-flex py-2">
    <div class="col px-2">
      <PrimeInputText
        labelType="float"
        :readonly="readonly"
        :disabled="readonly"
        :label="t('modul16c.bewertungsgruppe-edit.titel')"
        :modelValue="model?.title"
        @update:modelValue="value => update({ title: value ?? '' })"
      />
    </div>
    <div class="col px-2">
      <PrimeSelect
        :readonly="readonly"
        :disabled="readonly"
        :label="t('modul16c.bewertungsgruppe-edit.ziel')"
        :titleGetter="(value: Goal) => value.text"
        :keyGetter="(value: Goal) => value.key"
        :modelValue="model?.goal as Goal"
        :items="chosenEconomicActivity.economicActivity.goals"
        :showClear="false"
        :hasAutocomplete="false"
        @update:modelValue="
          value => update({ goalKey: value ? (value as Goal).key : null, goal: value ? (value as Goal) : null })
        "
        :id="'evaluationGroupGoalEdit-' + evaluationGroup?.key"
        labelType="float"
      />
    </div>
    <div class="col w-30 px-2">
      <PrimeSelect
        :readonly="readonly"
        :disabled="readonly"
        :label="t('modul16c.bewertungsgruppe-edit.art')"
        :titleGetter="value => (value as Type).title"
        :keyGetter="value => (value as Type).key"
        :modelValue="model?.taxType"
        :items="types"
        :showClear="false"
        :hasAutocomplete="false"
        @update:modelValue="
          value => update({ taxTypeKey: value ? (value as Type).key : null, taxType: value ? (value as Type) : null })
        "
        :id="'evaluationGroupTaxTypeEdit-' + evaluationGroup?.key"
        labelType="float"
      />
    </div>
    <div class="col-1">
      <PrimeButton :disabled="readonly" variant="secondary" v-if="model?.key" @click="deleteFunc">
        <font-awesome-icon :icon="['fal', 'trash-can']" />
      </PrimeButton>
    </div>
  </div>
  <div>
    <div class="col-12 px-2">
      <PrimeTextArea
        labelType="float"
        :readonly="readonly"
        :disabled="readonly"
        :label="t('modul16c.bewertungsgruppe-edit.beschreibung')"
        :modelValue="model?.description"
        rows="3"
        @update:modelValue="value => update({ description: value ?? '' })"
      />
    </div>
  </div>
</template>

<script lang="ts" setup>
import PrimeButton from '@/components/shared/Primevue/Buttons/PrimeButton.vue'
import PrimeInputText from '@/components/shared/Primevue/Inputs/PrimeInputText.vue'
import PrimeTextArea from '@/components/shared/Primevue/Inputs/PrimeTextArea.vue'
import PrimeSelect from '@/components/shared/Primevue/Selects/PrimeSelect.vue'
import MultiSelectOwnerAvatar from '@/components/shared/Selects/MultiSelectOwnerAvatar.vue'
import StatusIndicatorSelect from '@/components/shared/StatusIndicatorSelect.vue'
import { useOwner } from '@/composables/Owner'
import useOwnership from '@/composables/Ownership'
import config from '@/config'
import type ChosenEconomicActivity from '@/models/EUTaxonomy/ChosenEconomicActivity'
import type { EvaluationGroup } from '@/models/EUTaxonomy/EvaluationGroup'
import type { Goal } from '@/models/EUTaxonomy/Goal'
import type Type from '@/models/EUTaxonomy/Type'
import { ProcessingStatus } from '@/models/ProcessingStatus'
import type User from '@/models/User'
import DisplayableError from '@/pojos/DisplayableError'
import EUTaxonomyService from '@/services/EUTaxonomy/EUTaxonomyService'
import { t } from '@/services/LanguageService'
import { useEUTaxonomyStore } from '@/store/EUTaxonomyStore'
import { storeToRefs } from 'pinia'
import { computed, onMounted, ref } from 'vue'

const euTaxonomyService = new EUTaxonomyService()
const readonly = computed(() => euTaxonomyService.isReadonly(props.evaluationGroup?.owners))

const props = withDefaults(
  defineProps<{
    evaluationGroup?: EvaluationGroup
    chosenEconomicActivity: ChosenEconomicActivity
    passedReadonly: boolean
  }>(),
  { passedReadonly: false },
)

const { types } = storeToRefs(useEUTaxonomyStore())
const { updateEvaluationGroup, deleteEvaluationGroup } = useEUTaxonomyStore()
const model = ref<EvaluationGroup | undefined>()

const { addOwner, removeOwner } = useOwnership(
  config.apiEndpoints.euTaxonomy.addEvaluationGroupOwnership,
  config.apiEndpoints.euTaxonomy.removeEvaluationGroupOwnership,
)
const { deleteOwnerWithConfirmation } = useOwner()

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

const update = (data: Partial<EvaluationGroup>) => {
  if (!model.value) return
  if (model.value.key) {
    updateEvaluationGroup(model.value, data)
  }
}

const deleteFunc = () => {
  if (!model.value) return
  if (model.value.key === undefined) throw new DisplayableError(t('now-selection-provided-error'))
  deleteEvaluationGroup(model.value)
  model.value = undefined
}
onMounted(() => {
  if (!props.evaluationGroup) {
    model.value = {
      title: '',
      description: '',
      economicActivity: props.chosenEconomicActivity.economicActivity,
      questions: props.chosenEconomicActivity.economicActivity.questions,
      chosenEconomicActivity: props.chosenEconomicActivity,
      position: 0,
      status: ProcessingStatus.Pending,
      owners: [],
      key: '',
    }
  } else {
    model.value = props.evaluationGroup
  }
})
</script>

<style scoped>
label {
  width: 100%;
}
.w-30 {
  width: 30%;
}
</style>
