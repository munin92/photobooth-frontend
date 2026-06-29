<template>
  <q-page-sticky :position="position" class="q-ma-lg">
    <div
      id="print-queue-overlay"
      class="glass-effect row items-center no-wrap q-gutter-x-sm q-px-md q-py-sm"
      :style="{ '--overlay-opacity': opacity }"
    >
      <q-icon name="sym_o_print" size="sm" />
      <div class="column">
        <div class="text-weight-medium">{{ t('PRINT_QUEUE_OVERLAY_QUEUED', { count: queueCount }) }}</div>
        <div v-if="showPrinterStatus" class="text-caption" :class="statusColorClass">{{ statusText }}</div>
      </div>
    </div>
  </q-page-sticky>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useIntervalFn } from '@vueuse/core'
import { storeToRefs } from 'pinia'
import { useI18n } from 'vue-i18n'
import { _fetch } from '@/util/fetch_api.js'
import { useConfigurationStore } from '@/stores/configuration-store'
import type { components } from '@/dto/api'

type PrintQueueStatus = components['schemas']['PrintQueueStatus']
type PrinterStatus = components['schemas']['PrinterStatus']

const { t } = useI18n()
const configurationStore = useConfigurationStore()
const { configuration } = storeToRefs(configurationStore)

const position = computed(() => configuration.value.print_queue_overlay.position)
const opacity = computed(() => configuration.value.print_queue_overlay.opacity)
const pollIntervalMs = computed(() => configuration.value.print_queue_overlay.poll_interval_ms)
const showPrinterStatus = computed(() => configuration.value.print_queue_overlay.show_printer_status)

const status = ref<PrintQueueStatus | null>(null)
const queueCount = computed(() => status.value?.queue_depth ?? 0)
const printerStatus = computed<PrinterStatus>(() => status.value?.printer_status ?? 'unknown')

const statusText = computed(() => {
  switch (printerStatus.value) {
    case 'busy':
      return t('PRINT_QUEUE_OVERLAY_PRINTING')
    case 'ok':
      return t('PRINT_QUEUE_OVERLAY_IDLE')
    default:
      return t('PRINT_QUEUE_OVERLAY_CHECK')
  }
})

const statusColorClass = computed(() => {
  switch (printerStatus.value) {
    case 'busy':
      return 'text-info'
    case 'ok':
      return 'text-positive'
    default:
      return 'text-negative'
  }
})

const fetchQueueStatus = async () => {
  try {
    const response = await _fetch('/api/print/queue-status')
    if (!response.ok) {
      return
    }
    status.value = (await response.json()) as PrintQueueStatus
  } catch (error) {
    console.error('failed to fetch print queue status', error)
  }
}

// useIntervalFn polls immediately and restarts automatically when the interval changes; cleaned up on unmount.
const { pause, resume } = useIntervalFn(fetchQueueStatus, pollIntervalMs, { immediateCallback: true })
watch(pollIntervalMs, () => {
  pause()
  resume()
})
</script>

<style lang="scss" scoped>
#print-queue-overlay {
  pointer-events: none;
  user-select: none;
  border-radius: 8px;
  backdrop-filter: blur(10px);
  background: color-mix(in srgb, var(--q-dark-page) calc(var(--overlay-opacity) * 100%), transparent);
}
</style>
