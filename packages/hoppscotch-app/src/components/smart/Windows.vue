<template>
  <div class="flex flex-col flex-1 h-auto overflow-y-hidden flex-nowrap">
    <div class="relative sticky top-0 z-10 tabs bg-primaryLight">
      <div class="flex flex-1 w-0 overflow-x-auto">
        <div class="flex justify-between divide-x divide-dividerLight">
          <div class="flex">
            <draggable
              v-bind="dragOptions"
              :list="tabEntries"
              :style="tabsWidth"
              :item-key="'window-'"
              class="flex overflow-x-auto transition divide-x divide-dividerLight"
              @sort="sortTabs"
            >
              <template #item="{ element: [tabID, tabMeta] }">
                <button
                  :key="`removable-tab-${tabID}`"
                  class="tab"
                  :class="[{ active: modelValue === tabID }]"
                  :aria-label="tabMeta.label || ''"
                  role="button"
                  @keyup.enter="selectTab(tabID)"
                  @click="selectTab(tabID)"
                >
                  <div class="flex items-stretch truncate">
                    <span
                      v-if="tabMeta.icon"
                      class="flex items-center justify-center mx-4 cursor-pointer"
                    >
                      <component :is="tabMeta.icon" class="w-4 h-4 svg-icons" />
                    </span>
                    <span class="truncate">
                      {{ tabMeta.label }}
                    </span>
                  </div>
                  <ButtonSecondary
                    v-tippy="{ theme: 'tooltip', delay: [500, 20] }"
                    :icon="IconX"
                    :style="{
                      visibility: tabMeta.isRemovable ? 'visible' : 'hidden',
                    }"
                    :title="t('action.close')"
                    :class="[{ active: modelValue === tabID }, 'close']"
                    class="rounded mx-2 !py-0.5 !px-1"
                    @click.stop="emit('removeTab', tabID)"
                  />
                </button>
              </template>
            </draggable>
          </div>
          <div class="sticky right-0 flex items-center justify-center z-8">
            <slot name="actions">
              <span
                v-if="canAddNewTab"
                class="flex items-center justify-center px-2 py-1.5 bg-primaryLight z-8"
              >
                <ButtonSecondary
                  v-tippy="{ theme: 'tooltip' }"
                  :title="t('action.new')"
                  :icon="IconPlus"
                  class="rounded !p-1"
                  filled
                  @click="addTab"
                />
              </span>
            </slot>
          </div>
        </div>
      </div>
    </div>
    <div class="w-full h-full contents">
      <slot></slot>
    </div>
  </div>
</template>

<script setup lang="ts">
import IconPlus from "~icons/lucide/plus"
import IconX from "~icons/lucide/x"
import { pipe } from "fp-ts/function"
import { not } from "fp-ts/Predicate"
import * as A from "fp-ts/Array"
import * as O from "fp-ts/Option"
import { ref, ComputedRef, computed, provide } from "vue"
import type { Slot } from "vue"
import draggable from "vuedraggable"
import { throwError } from "~/helpers/functional/error"
import { useI18n } from "~/composables/i18n"

export type TabMeta = {
  label: string | null
  icon: Slot | undefined
  info: string | null
  isRemovable: boolean
}
export type TabProvider = {
  activeTabID: ComputedRef<string>
  addTabEntry: (tabID: string, meta: TabMeta) => void
  updateTabEntry: (tabID: string, newMeta: TabMeta) => void
  removeTabEntry: (tabID: string) => void
}

const t = useI18n()

const props = defineProps({
  styles: {
    type: String,
    default: "",
  },
  modelValue: {
    type: String,
    required: true,
  },
  canAddNewTab: {
    type: Boolean,
    default: true,
  },
})
const emit = defineEmits<{
  (e: "update:modelValue", newTabID: string): void
  (e: "sort", body: { oldIndex: number; newIndex: number }): void
  (e: "removeTab", tabID: string): void
  (e: "addTab"): void
}>()
const tabEntries = ref<Array<[string, TabMeta]>>([])
const tabsWidth = computed(() => ({
  maxWidth: `${tabEntries.value.length * 184}px`,
  width: "100%",
  minWidth: "0px",
  transition: "max-width 0.2s",
}))
const dragOptions = {
  group: "tabs",
  animation: 250,
  handle: ".tab",
  draggable: ".tab",
  ghostClass: "cursor-move",
}
const addTabEntry = (tabID: string, meta: TabMeta) => {
  tabEntries.value = pipe(
    tabEntries.value,
    O.fromPredicate(not(A.exists(([id]) => id === tabID))),
    O.map(A.append([tabID, meta] as [string, TabMeta])),
    O.getOrElseW(() => throwError(`Tab with duplicate ID created: '${tabID}'`))
  )
}
const updateTabEntry = (tabID: string, newMeta: TabMeta) => {
  tabEntries.value = pipe(
    tabEntries.value,
    A.findIndex(([id]) => id === tabID),
    O.chain((index) =>
      pipe(
        tabEntries.value,
        A.updateAt(index, [tabID, newMeta] as [string, TabMeta])
      )
    ),
    O.getOrElseW(() => throwError(`Failed to update tab entry: ${tabID}`))
  )
}
const removeTabEntry = (tabID: string) => {
  tabEntries.value = pipe(
    tabEntries.value,
    A.findIndex(([id]) => id === tabID),
    O.chain((index) => pipe(tabEntries.value, A.deleteAt(index))),
    O.getOrElseW(() => throwError(`Failed to remove tab entry: ${tabID}`))
  )
  // If we tried to remove the active tabEntries, switch to first tab entry
  if (props.modelValue === tabID)
    if (tabEntries.value.length > 0) selectTab(tabEntries.value[0][0])
}
const sortTabs = (e: {
  oldDraggableIndex: number
  newDraggableIndex: number
}) => {
  emit("sort", {
    oldIndex: e.oldDraggableIndex,
    newIndex: e.newDraggableIndex,
  })
}
provide<TabProvider>("tabs-system", {
  activeTabID: computed(() => props.modelValue),
  addTabEntry,
  updateTabEntry,
  removeTabEntry,
})
const selectTab = (id: string) => {
  emit("update:modelValue", id)
}
const addTab = () => {
  emit("addTab")
}
</script>

<style scoped lang="scss">
.tabs {
  @apply flex;
  @apply whitespace-nowrap;
  @apply overflow-auto;
  @apply flex-shrink-0;

  &::after {
    @apply absolute;
    @apply inset-x-0;
    @apply bottom-0;
    @apply bg-dividerLight;
    @apply z-10;
    @apply h-0.25;
    content: "";
  }

  .tab {
    @apply relative;
    @apply flex;
    @apply py-2;
    @apply font-semibold;
    @apply w-46;
    @apply transition;
    @apply flex-1;
    @apply items-center;
    @apply justify-between;
    @apply text-secondaryLight;
    @apply hover:bg-primaryDark;
    @apply hover:text-secondary;
    @apply focus-visible:text-secondaryDark;

    &::before {
      @apply absolute;
      @apply left-0;
      @apply right-0;
      @apply top-0;
      @apply bg-transparent;
      @apply z-2;
      @apply h-0.5;
      content: "";
    }

    // &::after {
    //   @apply absolute;
    //   @apply left-0;
    //   @apply right-0;
    //   @apply bottom-0;
    //   @apply bg-divider;
    //   @apply z-2;
    //   @apply h-0.25;
    //   content: "";
    // }

    &:focus::before {
      @apply bg-divider;
    }

    &.active {
      @apply text-secondaryDark;
      @apply bg-primary;

      &::before {
        @apply bg-accent;
      }

      // &::after {
      //   @apply bg-transparent;
      // }
    }

    .close {
      @apply opacity-50;

      &.active {
        @apply opacity-80;
      }
    }
  }
}
</style>
