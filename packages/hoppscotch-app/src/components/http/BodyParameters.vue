<template>
  <div>
    <div
      class="sticky z-10 flex items-center justify-between pl-4 border-b bg-primary border-dividerLight top-upperMobileStickyFold sm:top-upperMobileTertiaryStickyFold"
    >
      <label class="font-semibold text-secondaryLight">
        {{ t("request.body") }}
      </label>
      <div class="flex">
        <ButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          to="https://docs.hoppscotch.io/features/body"
          blank
          :title="t('app.wiki')"
          :icon="IconHelpCircle"
        />
        <ButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.clear_all')"
          :icon="IconTrash2"
          @click="clearContent"
        />
        <ButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          :title="t('add.new')"
          :icon="IconPlus"
          @click="addBodyParam"
        />
      </div>
    </div>

    <draggable
      v-model="workingParams"
      item-key="id"
      animation="250"
      handle=".draggable-handle"
      draggable=".draggable-content"
      ghost-class="cursor-move"
      chosen-class="bg-primaryLight"
      drag-class="cursor-grabbing"
    >
      <template #item="{ element: { entry }, index }">
        <div
          class="flex border-b divide-x divide-dividerLight border-dividerLight draggable-content group"
        >
          <span>
            <ButtonSecondary
              v-tippy="{
                theme: 'tooltip',
                delay: [500, 20],
                content:
                  index !== workingParams?.length - 1
                    ? t('action.drag_to_reorder')
                    : null,
              }"
              :icon="IconGripVertical"
              class="cursor-auto text-primary hover:text-primary"
              :class="{
                'draggable-handle group-hover:text-secondaryLight !cursor-grab':
                  index !== workingParams?.length - 1,
              }"
              tabindex="-1"
            />
          </span>
          <SmartEnvInput
            v-model="entry.key"
            :placeholder="`${t('count.parameter', { count: index + 1 })}`"
            @change="
              updateBodyParam(index, {
                key: $event,
                value: entry.value,
                active: entry.active,
                isFile: entry.isFile,
              })
            "
          />
          <div v-if="entry.isFile" class="file-chips-container">
            <div class="space-x-2 file-chips-wrapper">
              <SmartFileChip
                v-for="(file, fileIndex) in entry.value"
                :key="`param-${index}-file-${fileIndex}`"
                >{{ file.name }}</SmartFileChip
              >
            </div>
          </div>
          <span v-else class="flex flex-1">
            <SmartEnvInput
              v-model="entry.value"
              :placeholder="`${t('count.value', { count: index + 1 })}`"
              @change="
                updateBodyParam(index, {
                  key: entry.key,
                  value: $event,
                  active: entry.active,
                  isFile: entry.isFile,
                })
              "
            />
          </span>
          <span>
            <label :for="`attachment${index}`" class="p-0">
              <input
                :id="`attachment${index}`"
                :name="`attachment${index}`"
                type="file"
                multiple
                class="p-1 cursor-pointer transition file:transition file:cursor-pointer text-secondaryLight hover:text-secondaryDark file:mr-2 file:py-1 file:px-4 file:rounded file:border-0 file:text-tiny text-tiny file:text-secondary hover:file:text-secondaryDark file:bg-primaryLight hover:file:bg-primaryDark"
                @change="setRequestAttachment(index, entry, $event)"
              />
            </label>
          </span>
          <span>
            <ButtonSecondary
              v-tippy="{ theme: 'tooltip' }"
              :title="
                entry.hasOwnProperty('active')
                  ? entry.active
                    ? t('action.turn_off')
                    : t('action.turn_on')
                  : t('action.turn_off')
              "
              :icon="
                entry.hasOwnProperty('active')
                  ? entry.active
                    ? IconCheckCircle
                    : IconCircle
                  : IconCheckCircle
              "
              color="green"
              @click="
                updateBodyParam(index, {
                  key: entry.key,
                  value: entry.value,
                  active: entry.hasOwnProperty('active')
                    ? !entry.active
                    : false,
                  isFile: entry.isFile,
                })
              "
            />
          </span>
          <span>
            <ButtonSecondary
              v-tippy="{ theme: 'tooltip' }"
              :title="t('action.remove')"
              :icon="IconTrash"
              color="red"
              @click="deleteBodyParam(index)"
            />
          </span>
        </div>
      </template>
    </draggable>

    <div
      v-if="workingParams.length === 0"
      class="flex flex-col items-center justify-center p-4 text-secondaryLight"
    >
      <img
        :src="`/images/states/${colorMode.value}/upload_single_file.svg`"
        loading="lazy"
        class="inline-flex flex-col object-contain object-center w-16 h-16 my-4"
        :alt="`${t('empty.body')}`"
      />
      <span class="pb-4 text-center">{{ t("empty.body") }}</span>
      <ButtonSecondary
        :label="`${t('add.new')}`"
        filled
        :icon="IconPlus"
        class="mb-4"
        @click="addBodyParam"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import IconHelpCircle from "~icons/lucide/help-circle"
import IconTrash2 from "~icons/lucide/trash-2"
import IconPlus from "~icons/lucide/plus"
import IconGripVertical from "~icons/lucide/grip-vertical"
import IconCheckCircle from "~icons/lucide/check-circle"
import IconCircle from "~icons/lucide/circle"
import IconTrash from "~icons/lucide/trash"
import { ref, watch } from "vue"
import { flow, pipe } from "fp-ts/function"
import * as O from "fp-ts/Option"
import * as A from "fp-ts/Array"
import { FormDataKeyValue } from "@hoppscotch/data"
import { isEqual, clone } from "lodash-es"
import draggable from "vuedraggable"
import { pluckRef } from "@composables/ref"
import { useI18n } from "@composables/i18n"
import { useToast } from "@composables/toast"
import { useColorMode } from "@composables/theming"
import { useRESTRequestBody } from "~/newstore/RESTSession"

type WorkingFormDataKeyValue = { id: number; entry: FormDataKeyValue }

const colorMode = useColorMode()
const t = useI18n()

const toast = useToast()

const idTicker = ref(0)

const deletionToast = ref<{ goAway: (delay: number) => void } | null>(null)

const bodyParams = pluckRef<any, any>(useRESTRequestBody(), "body")

// The UI representation of the parameters list (has the empty end param)
const workingParams = ref<WorkingFormDataKeyValue[]>([
  {
    id: idTicker.value++,
    entry: {
      key: "",
      value: "",
      active: true,
      isFile: false,
    },
  },
])

// Rule: Working Params always have last element is always an empty param
watch(workingParams, (paramsList) => {
  if (
    paramsList.length > 0 &&
    paramsList[paramsList.length - 1].entry.key !== ""
  ) {
    workingParams.value.push({
      id: idTicker.value++,
      entry: {
        key: "",
        value: "",
        active: true,
        isFile: false,
      },
    })
  }
})

// Sync logic between params and working params
watch(
  bodyParams,
  (newParamsList) => {
    if (!Array.isArray(newParamsList)) return

    // Sync should overwrite working params
    const filteredWorkingParams = pipe(
      workingParams.value,
      A.filterMap(
        flow(
          O.fromPredicate((e) => e.entry.key !== "" || e.entry.isFile),
          O.map((e) => e.entry)
        )
      )
    )

    if (!isEqual(newParamsList, filteredWorkingParams)) {
      workingParams.value = pipe(
        newParamsList,
        A.map((x) => ({ id: idTicker.value++, entry: x }))
      )
    }
  },
  { immediate: true }
)

watch(workingParams, (newWorkingParams) => {
  const fixedParams = pipe(
    newWorkingParams,
    A.filterMap(
      flow(
        O.fromPredicate((e) => e.entry.key !== "" || e.entry.isFile),
        O.map((e) => e.entry)
      )
    )
  )

  if (!isEqual(bodyParams.value, fixedParams)) {
    bodyParams.value = fixedParams
  }
})

const addBodyParam = () => {
  workingParams.value.push({
    id: idTicker.value++,
    entry: {
      key: "",
      value: "",
      active: true,
      isFile: false,
    },
  })
}

const updateBodyParam = (index: number, entry: FormDataKeyValue) => {
  workingParams.value = workingParams.value.map((h, i) =>
    i === index ? { id: h.id, entry } : h
  )
}

const deleteBodyParam = (index: number) => {
  const paramsBeforeDeletion = clone(workingParams.value)

  if (
    !(
      paramsBeforeDeletion.length > 0 &&
      index === paramsBeforeDeletion.length - 1
    )
  ) {
    if (deletionToast.value) {
      deletionToast.value.goAway(0)
      deletionToast.value = null
    }

    deletionToast.value = toast.success(`${t("state.deleted")}`, {
      action: [
        {
          text: `${t("action.undo")}`,
          onClick: (_, toastObject) => {
            workingParams.value = paramsBeforeDeletion
            toastObject.goAway(0)
            deletionToast.value = null
          },
        },
      ],

      onComplete: () => {
        deletionToast.value = null
      },
    })
  }

  workingParams.value = workingParams.value.filter(
    (_, arrIndex) => arrIndex != index
  )
}

const clearContent = () => {
  // set params list to the initial state
  workingParams.value = [
    {
      id: idTicker.value++,
      entry: {
        key: "",
        value: "",
        active: true,
        isFile: false,
      },
    },
  ]
}

const setRequestAttachment = (
  index: number,
  entry: FormDataKeyValue,
  event: InputEvent
) => {
  // check if file exists or not
  if ((event.target as HTMLInputElement).files?.length === 0) {
    updateBodyParam(index, {
      ...entry,
      isFile: false,
      value: "",
    })
    return
  }

  const fileEntry: FormDataKeyValue = {
    ...entry,
    isFile: true,
    value: Array.from((event.target as HTMLInputElement).files!),
  }
  updateBodyParam(index, fileEntry)
}
</script>

<style lang="scss" scoped>
.file-chips-container {
  @apply flex flex-1;
  @apply whitespace-nowrap;
  @apply overflow-auto;
  @apply bg-transparent;

  .file-chips-wrapper {
    @apply flex;
    @apply p-1;
    @apply w-0;
  }
}
</style>
