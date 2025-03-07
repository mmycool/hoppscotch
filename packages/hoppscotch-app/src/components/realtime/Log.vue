<template>
  <div class="flex flex-col flex-1 overflow-auto whitespace-nowrap">
    <div
      class="sticky top-0 z-10 flex items-center justify-between pl-4 border-b bg-primary border-dividerLight"
    >
      <label for="log" class="font-semibold text-secondaryLight">
        {{ title }}
      </label>
      <div class="flex">
        <ButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.delete')"
          :icon="IconTrash"
          @click="emit('delete')"
        />
        <ButtonSecondary
          id="bottompage"
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.scroll_to_top')"
          :icon="IconArrowUp"
          @click="scrollTo('top')"
        />
        <ButtonSecondary
          id="bottompage"
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.scroll_to_bottom')"
          :icon="IconArrowDown"
          @click="scrollTo('bottom')"
        />
        <ButtonSecondary
          id="bottompage"
          v-tippy="{ theme: 'tooltip' }"
          :title="t('action.autoscroll')"
          :icon="IconChevronsDown"
          :class="toggleAutoscrollColor"
          @click="toggleAutoscroll()"
        />
      </div>
    </div>
    <div
      v-if="log.length !== 0"
      ref="logs"
      class="flex flex-col overflow-y-auto border-b border-dividerLight"
    >
      <div class="flex flex-col h-full divide-y divide-dividerLight">
        <RealtimeLogEntry
          v-for="(entry, index) in log"
          :key="`entry-${index}`"
          :entry="entry"
        />
      </div>
    </div>
    <div
      v-else
      class="flex flex-col items-center justify-center p-4 text-secondaryLight"
    >
      <div class="flex pb-4 my-4 space-x-2">
        <div class="flex flex-col items-end space-y-4 text-right">
          <span class="flex items-center flex-1">
            {{ t("shortcut.request.send_request") }}
          </span>
          <span class="flex items-center flex-1">
            {{ t("shortcut.general.show_all") }}
          </span>
          <span class="flex items-center flex-1">
            {{ t("shortcut.general.command_menu") }}
          </span>
          <span class="flex items-center flex-1">
            {{ t("shortcut.general.help_menu") }}
          </span>
        </div>
        <div class="flex flex-col space-y-4">
          <div class="flex">
            <kbd class="shortcut-key">{{ getSpecialKey() }}</kbd>
            <kbd class="shortcut-key">↩</kbd>
          </div>
          <div class="flex">
            <kbd class="shortcut-key">{{ getSpecialKey() }}</kbd>
            <kbd class="shortcut-key">K</kbd>
          </div>
          <div class="flex">
            <kbd class="shortcut-key">/</kbd>
          </div>
          <div class="flex">
            <kbd class="shortcut-key">?</kbd>
          </div>
        </div>
      </div>
      <ButtonSecondary
        :label="`${t('app.documentation')}`"
        to="https://docs.hoppscotch.io/realtime"
        :icon="IconExternalLink"
        blank
        outline
        reverse
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, PropType, computed, watch } from "vue"
import IconTrash from "~icons/lucide/trash"
import IconArrowUp from "~icons/lucide/arrow-up"
import IconArrowDown from "~icons/lucide/arrow-down"
import IconChevronsDown from "~icons/lucide/chevron-down"
import IconExternalLink from "~icons/lucide/external-link"
import { useThrottleFn, useScroll } from "@vueuse/core"
import { useI18n } from "@composables/i18n"
import { getPlatformSpecialKey as getSpecialKey } from "~/helpers/platformutils"

export type LogEntryData = {
  prefix?: string
  ts: number | undefined
  source: "info" | "client" | "server" | "disconnected"
  payload: string
  event: "connecting" | "connected" | "disconnected" | "error"
}

const props = defineProps({
  log: { type: Array as PropType<LogEntryData[]>, default: () => [] },
  title: {
    type: String,
    default: "",
  },
})

const emit = defineEmits<{
  (e: "delete"): void
}>()

const t = useI18n()

const logs = ref<HTMLElement>()

const autoScrollEnabled = ref(true)

const logListScroll = useScroll(logs)

// Disable autoscroll when scrolling to top
watch(logListScroll.isScrolling, (isScrolling) => {
  if (isScrolling && logListScroll.directions.top)
    autoScrollEnabled.value = false
})

const scrollTo = (position: "top" | "bottom") => {
  if (position === "top") {
    logs.value?.scroll({
      behavior: "smooth",
      top: 0,
    })
  } else if (position === "bottom") {
    logs.value?.scroll({
      behavior: "smooth",
      top: logs.value?.scrollHeight,
    })
  }
}

watch(
  () => props.log,
  useThrottleFn(() => {
    if (autoScrollEnabled.value) scrollTo("bottom")
  }, 200),
  { flush: "post" }
)

const toggleAutoscroll = () => {
  autoScrollEnabled.value = !autoScrollEnabled.value
}

const toggleAutoscrollColor = computed(() =>
  autoScrollEnabled.value ? "text-green-500" : "text-red-500"
)
</script>
