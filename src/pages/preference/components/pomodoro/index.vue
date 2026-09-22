<script setup lang="ts">
import { Button, InputNumber, message, Progress } from 'ant-design-vue'
import { computed, onMounted, onUnmounted, reactive } from 'vue'
import { useI18n } from 'vue-i18n'

import ProList from '@/components/pro-list/index.vue'
import ProListItem from '@/components/pro-list-item/index.vue'
import { showWindow } from '@/plugins/window'

type Phase = 'focus' | 'break'

interface TimerState {
  focusMinutes: number
  breakMinutes: number
  phase: Phase
  remainingSeconds: number
  endsAt: number | null
}

const STORAGE_KEY = 'bongocat:pomodoro:v1'
const DEFAULT_STATE: TimerState = {
  focusMinutes: 25,
  breakMinutes: 5,
  phase: 'focus',
  remainingSeconds: 25 * 60,
  endsAt: null,
}

const { t } = useI18n()
const timer = reactive<TimerState>({ ...DEFAULT_STATE })
let intervalId: ReturnType<typeof setInterval> | undefined

const isRunning = computed(() => timer.endsAt !== null)
const totalSeconds = computed(() => getDuration(timer.phase))
const progress = computed(() => Math.round((1 - timer.remainingSeconds / totalSeconds.value) * 100))
const timeText = computed(() => {
  const minutes = Math.floor(timer.remainingSeconds / 60)
  const seconds = timer.remainingSeconds % 60

  return `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`
})

function getDuration(phase: Phase) {
  return (phase === 'focus' ? timer.focusMinutes : timer.breakMinutes) * 60
}

function validMinutes(value: unknown, fallback: number) {
  return typeof value === 'number' && Number.isInteger(value) && value >= 1 && value <= 120
    ? value
    : fallback
}

function save() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(timer))
}

function load() {
  try {
    const saved = JSON.parse(localStorage.getItem(STORAGE_KEY) || 'null')

    if (!saved || typeof saved !== 'object') return

    timer.focusMinutes = validMinutes(saved.focusMinutes, DEFAULT_STATE.focusMinutes)
    timer.breakMinutes = validMinutes(saved.breakMinutes, DEFAULT_STATE.breakMinutes)
    timer.phase = saved.phase === 'break' ? 'break' : 'focus'
    timer.remainingSeconds = typeof saved.remainingSeconds === 'number'
      && Number.isInteger(saved.remainingSeconds)
      && saved.remainingSeconds > 0
      && saved.remainingSeconds <= getDuration(timer.phase)
      ? saved.remainingSeconds
      : getDuration(timer.phase)
    timer.endsAt = typeof saved.endsAt === 'number' && Number.isFinite(saved.endsAt)
      ? saved.endsAt
      : null
  } catch {
    // Damaged saved data should not prevent the settings window from opening.
  }
}

function tick(notify = true) {
  if (timer.endsAt === null) return

  const remaining = Math.min(
    getDuration(timer.phase),
    Math.max(0, Math.ceil((timer.endsAt - Date.now()) / 1000)),
  )

  if (remaining > 0) {
    timer.remainingSeconds = remaining
    return
  }

  const completedPhase = timer.phase
  timer.phase = completedPhase === 'focus' ? 'break' : 'focus'
  timer.remainingSeconds = getDuration(timer.phase)
  timer.endsAt = null
  save()

  if (notify) {
    showWindow('preference')
    message.success(t(completedPhase === 'focus'
      ? 'pages.preference.pomodoro.focusComplete'
      : 'pages.preference.pomodoro.breakComplete'))
  }
}

function start() {
  if (isRunning.value) return

  timer.endsAt = Date.now() + timer.remainingSeconds * 1000
  save()
}

function pause() {
  tick(false)
  timer.endsAt = null
  save()
}

function reset() {
  timer.endsAt = null
  timer.remainingSeconds = getDuration(timer.phase)
  save()
}

function selectPhase(phase: Phase) {
  if (timer.phase === phase) return

  timer.phase = phase
  reset()
}

function setMinutes(phase: Phase, value: unknown) {
  const minutes = validMinutes(value, phase === 'focus' ? timer.focusMinutes : timer.breakMinutes)

  if (phase === 'focus') timer.focusMinutes = minutes
  else timer.breakMinutes = minutes

  if (timer.phase === phase) reset()
  else save()
}

onMounted(() => {
  load()
  tick(false)
  intervalId = setInterval(() => tick(), 500)
})

onUnmounted(() => {
  if (intervalId) clearInterval(intervalId)
})
</script>

<template>
  <ProList :title="$t('pages.preference.pomodoro.title')">
    <div class="b b-color-2 rounded-lg b-solid bg-color-3 p-8 text-center">
      <div class="mb-5 flex justify-center gap-2">
        <Button
          :type="timer.phase === 'focus' ? 'primary' : 'default'"
          @click="selectPhase('focus')"
        >
          {{ $t('pages.preference.pomodoro.focus') }}
        </Button>
        <Button
          :type="timer.phase === 'break' ? 'primary' : 'default'"
          @click="selectPhase('break')"
        >
          {{ $t('pages.preference.pomodoro.break') }}
        </Button>
      </div>

      <Progress
        :percent="progress"
        :size="220"
        :stroke-color="timer.phase === 'focus' ? '#f97373' : '#65bfa5'"
        type="circle"
      >
        <template #format>
          <div class="text-10 font-semibold font-mono tabular-nums">
            {{ timeText }}
          </div>
        </template>
      </Progress>

      <div class="mt-6 flex justify-center gap-2">
        <Button
          v-if="isRunning"
          type="primary"
          @click="pause"
        >
          {{ $t('pages.preference.pomodoro.pause') }}
        </Button>
        <Button
          v-else
          type="primary"
          @click="start"
        >
          {{ $t('pages.preference.pomodoro.start') }}
        </Button>
        <Button @click="reset">
          {{ $t('pages.preference.pomodoro.reset') }}
        </Button>
      </div>
      <div class="mt-4 text-xs text-color-3">
        {{ $t('pages.preference.pomodoro.nextHint') }}
      </div>
    </div>
  </ProList>

  <ProList :title="$t('pages.preference.pomodoro.durationTitle')">
    <ProListItem :title="$t('pages.preference.pomodoro.focusDuration')">
      <InputNumber
        :addon-after="$t('pages.preference.pomodoro.minutes')"
        :disabled="isRunning"
        :max="120"
        :min="1"
        :precision="0"
        :value="timer.focusMinutes"
        @change="value => setMinutes('focus', value)"
      />
    </ProListItem>
    <ProListItem :title="$t('pages.preference.pomodoro.breakDuration')">
      <InputNumber
        :addon-after="$t('pages.preference.pomodoro.minutes')"
        :disabled="isRunning"
        :max="120"
        :min="1"
        :precision="0"
        :value="timer.breakMinutes"
        @change="value => setMinutes('break', value)"
      />
    </ProListItem>
  </ProList>
</template>
