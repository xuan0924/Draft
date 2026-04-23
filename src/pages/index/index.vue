<template>
  <view class="jot-page">
    <view class="jot-scene">
      <view class="jot-rings">
        <view v-for="n in 8" :key="n" class="jot-ring" />
      </view>

      <view
        class="jot-stage"
        :class="{ 'is-transitioning': animState !== 'drag' }"
        :style="stageStyle"
        @touchstart="onTouchStart"
        @touchmove="onTouchMove"
        @touchend="onTouchEnd"
      >
        <view class="jot-page-next">
          <view class="jot-half jot-half--top">
            <text class="jot-day">{{ incomingDay }}</text>
          </view>
          <view class="jot-half jot-half--bottom">
            <text class="jot-date">{{ incomingLabel }}</text>
            <text class="jot-text">{{ incomingEntry.text || '下一页准备完成。' }}</text>
            <view class="jot-summary">AI Summary</view>
          </view>
        </view>

        <view
          class="jot-flip"
          :class="{
            'is-commit': animState === 'commit',
            'is-rebound': animState === 'rebound'
          }"
          :style="flipStyle"
        >
          <view class="jot-half jot-half--top jot-fold-shell" :style="upperStyle">
            <view class="jot-half-face front">
              <image v-if="currentEntry.images[0]" class="jot-bg" :src="currentEntry.images[0]" mode="aspectFill" />
              <text class="jot-day">{{ currentDay }}</text>
            </view>
            <view class="jot-half-face back">
              <text class="jot-day mirror">{{ currentDay }}</text>
            </view>
          </view>

          <view class="jot-half jot-half--bottom">
            <image v-if="currentEntry.images[0]" class="jot-bg" :src="currentEntry.images[0]" mode="aspectFill" />
            <text class="jot-date">{{ currentLabel }}</text>
            <text class="jot-text">{{ currentEntry.text || '今天写点什么吧。' }}</text>
            <view class="jot-summary">AI Summary</view>
          </view>
        </view>
      </view>

      <view class="jot-base" />
    </view>
  </view>
</template>

<script setup>
import { computed, reactive, ref } from 'vue'

const FLIP_MS = 760
const COMMIT_THRESHOLD = 90
const DRAG_MAX = 240

const state = reactive({
  year: 2026,
  month: 4,
  day: 23,
  entries: {
    '2026-04-23': {
      text: '今天把翻页手感从硬切换升级成了跟手弹性翻页，观感更像真实纸页。',
      images: [],
      aiSummary: '拆分上下半页，支持拖拽角度与阻尼回弹。'
    },
    '2026-04-24': {
      text: '现在手指停在哪里，纸张就停在哪里，松手后按阈值决定翻页或回弹。',
      images: [],
      aiSummary: 'touchmove 实时驱动 rotateX，阈值触发完整翻页。'
    }
  }
})

const animState = ref('idle') // idle | drag | commit | rebound
const touchStartY = ref(0)
const dragDeltaY = ref(0)
const angleX = ref(0)
const incomingDayRef = ref(0)
const targetAngle = ref(-180)

const daysInMonth = computed(() => new Date(state.year, state.month, 0).getDate())

function dateKey(day) {
  return `${state.year}-${String(state.month).padStart(2, '0')}-${String(day).padStart(2, '0')}`
}

function getEntry(day) {
  return (
    state.entries[dateKey(day)] || {
      text: '',
      images: [],
      aiSummary: ''
    }
  )
}

function normalizeDay(day) {
  if (day < 1) return daysInMonth.value
  if (day > daysInMonth.value) return 1
  return day
}

function getTouchY(e) {
  const touch = e.changedTouches?.[0] || e.touches?.[0]
  return touch?.pageY ?? touch?.clientY ?? 0
}

const progress = computed(() => Math.min(Math.abs(angleX.value) / 180, 1))
const bendY = computed(() => progress.value < 0.5 ? progress.value * 2 : (1 - progress.value) * 2)

const currentDay = computed(() => state.day)
const currentLabel = computed(
  () => `${state.year}.${String(state.month).padStart(2, '0')}.${String(state.day).padStart(2, '0')}`
)
const currentEntry = computed(() => getEntry(state.day))

const incomingDay = computed(() => {
  if (!incomingDayRef.value) {
    return normalizeDay(state.day + 1)
  }
  return incomingDayRef.value
})
const incomingLabel = computed(
  () => `${state.year}.${String(state.month).padStart(2, '0')}.${String(incomingDay.value).padStart(2, '0')}`
)
const incomingEntry = computed(() => getEntry(incomingDay.value))

const stageStyle = computed(() => ({
  '--flip-progress': progress.value.toFixed(3),
  '--flip-angle': `${angleX.value.toFixed(2)}deg`,
  '--target-angle': `${targetAngle.value}deg`
}))

const flipStyle = computed(() => ({
  transform: `rotateX(${angleX.value}deg)`
}))

const upperStyle = computed(() => {
  const skew = bendY.value * 5
  const scale = 1 + bendY.value * 0.04
  return {
    transform: `perspective(1600rpx) rotateX(${Math.sign(angleX.value || -1) * bendY.value * -18}deg) skewY(${skew}deg) scaleY(${scale})`
  }
})

function commitFlip(targetDay, sign) {
  animState.value = 'commit'
  incomingDayRef.value = targetDay
  targetAngle.value = sign * 180
  angleX.value = sign * -180
  setTimeout(() => {
    state.day = targetDay
    animState.value = 'idle'
    angleX.value = 0
    dragDeltaY.value = 0
    incomingDayRef.value = 0
  }, FLIP_MS)
}

function reboundFlip() {
  animState.value = 'rebound'
  angleX.value = 0
  dragDeltaY.value = 0
  setTimeout(() => {
    animState.value = 'idle'
    incomingDayRef.value = 0
    targetAngle.value = -180
  }, FLIP_MS - 120)
}

function onTouchStart(e) {
  if (animState.value === 'commit') return
  animState.value = 'drag'
  touchStartY.value = getTouchY(e)
  dragDeltaY.value = 0
  incomingDayRef.value = normalizeDay(state.day + 1)
}

function onTouchMove(e) {
  if (animState.value !== 'drag') return
  const y = getTouchY(e)
  const delta = y - touchStartY.value
  dragDeltaY.value = delta

  const clamped = Math.max(-DRAG_MAX, Math.min(DRAG_MAX, delta))
  const ratio = clamped / DRAG_MAX
  angleX.value = Math.max(-170, Math.min(85, ratio * 180))

  incomingDayRef.value = clamped <= 0 ? normalizeDay(state.day + 1) : normalizeDay(state.day - 1)
}

function onTouchEnd() {
  if (animState.value !== 'drag') return

  const absDelta = Math.abs(dragDeltaY.value)
  const isPrev = dragDeltaY.value > 0
  const targetDay = isPrev ? normalizeDay(state.day - 1) : normalizeDay(state.day + 1)

  if (absDelta >= COMMIT_THRESHOLD) {
    const sign = isPrev ? 1 : -1
    commitFlip(targetDay, sign)
    return
  }
  reboundFlip()
}
</script>

<style scoped lang="scss">
.jot-page {
  min-height: 100vh;
  padding: 40rpx 22rpx 60rpx;
  background: radial-gradient(circle at 18% 12%, #f8fafc 0%, #e8ecf1 56%, #d9dde3 100%);
}

.jot-scene {
  width: 100%;
  max-width: 720rpx;
  margin: 0 auto;
  perspective: 1800rpx;
  perspective-origin: center 26%;
}

.jot-rings {
  display: flex;
  justify-content: center;
  gap: 18rpx;
  margin-bottom: -14rpx;
  position: relative;
  z-index: 5;
}

.jot-ring {
  width: 18rpx;
  height: 18rpx;
  border-radius: 50%;
  border: 3rpx solid #8c929b;
  background: linear-gradient(140deg, #dfe3e8 0%, #9ca3ae 100%);
  box-shadow: inset 0 2rpx 4rpx rgba(255, 255, 255, 0.6), 0 2rpx 4rpx rgba(0, 0, 0, 0.2);
}

.jot-stage {
  position: relative;
  height: 860rpx;
  transform-style: preserve-3d;
  will-change: transform;
}

.jot-stage::after {
  content: '';
  position: absolute;
  left: 6%;
  right: 6%;
  bottom: 6rpx;
  height: 90rpx;
  pointer-events: none;
  background: radial-gradient(
    ellipse at center,
    rgba(20, 26, 35, calc(0.12 + var(--flip-progress) * 0.34)) 0%,
    rgba(20, 26, 35, 0) 72%
  );
  transform: translateZ(-24rpx) scaleY(calc(1 + var(--flip-progress) * 0.25));
}

.jot-page-next,
.jot-flip {
  position: absolute;
  inset: 0;
  border-radius: 18rpx;
  overflow: hidden;
}

.jot-page-next {
  z-index: 1;
  background: linear-gradient(180deg, #f7f9fc 0%, #eceff5 100%);
  transform: translateZ(-10rpx) translateY(8rpx);
  box-shadow: 0 16rpx 24rpx rgba(0, 0, 0, 0.11);
}

.jot-flip {
  z-index: 3;
  transform-origin: top center;
  transform-style: preserve-3d;
  will-change: transform;
  box-shadow: 0 24rpx 36rpx rgba(0, 0, 0, 0.18), 0 8rpx 12rpx rgba(0, 0, 0, 0.12);
}

.jot-stage.is-transitioning .jot-flip {
  transition: transform 760ms cubic-bezier(0.22, 0.92, 0.26, 1);
}

.jot-flip.is-commit {
  animation: jot-paper-flip 760ms cubic-bezier(0.22, 0.92, 0.26, 1);
}

.jot-flip.is-rebound {
  transition: transform 620ms cubic-bezier(0.16, 1.15, 0.25, 1);
}

@keyframes jot-paper-flip {
  0% {
    transform: rotateX(var(--flip-angle));
  }
  50% {
    transform: rotateX(calc(var(--flip-angle) * 0.7 - 72deg)) skewY(3.8deg);
  }
  100% {
    transform: rotateX(var(--target-angle));
  }
}

.jot-half {
  position: relative;
  width: 100%;
  overflow: hidden;
  background: linear-gradient(180deg, #fbfcfd 0%, #f2f5f9 100%);
}

.jot-half--top {
  height: 50%;
  border-radius: 18rpx 18rpx 0 0;
}

.jot-half--bottom {
  height: 50%;
  border-radius: 0 0 18rpx 18rpx;
  border-top: 1px solid rgba(125, 133, 146, 0.18);
}

.jot-fold-shell {
  transform-origin: bottom center;
  transform-style: preserve-3d;
  will-change: transform;
}

.jot-half-face {
  position: absolute;
  inset: 0;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
}

.jot-half-face.back {
  transform: rotateX(180deg);
  background: linear-gradient(180deg, #e7ebf1 0%, #dbe1ea 100%);
}

.jot-bg {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  opacity: 0.14;
}

.jot-day {
  position: absolute;
  left: 50rpx;
  top: 64rpx;
  font-size: 168rpx;
  line-height: 1;
  font-weight: 800;
  color: #20252d;
  letter-spacing: 2rpx;
}

.jot-day.mirror {
  transform: rotateX(180deg);
  opacity: 0.35;
}

.jot-date {
  display: block;
  margin: 34rpx 52rpx 0;
  font-size: 28rpx;
  color: #687283;
}

.jot-text {
  display: block;
  margin: 18rpx 52rpx 0;
  font-size: 34rpx;
  line-height: 1.68;
  color: #2c323d;
  white-space: pre-wrap;
}

.jot-summary {
  position: absolute;
  left: 52rpx;
  right: 52rpx;
  bottom: 34rpx;
  height: 70rpx;
  border-radius: 12rpx;
  display: flex;
  align-items: center;
  padding-left: 20rpx;
  box-sizing: border-box;
  color: #f3f6fb;
  font-size: 24rpx;
  background: linear-gradient(90deg, rgba(34, 40, 52, 0.94), rgba(61, 70, 84, 0.94));
}

.jot-base {
  margin: 22rpx auto 0;
  width: 90%;
  height: 88rpx;
  border-radius: 20rpx;
  background: linear-gradient(135deg, #6f5038 0%, #89674a 30%, #4e3828 100%);
  box-shadow: inset 0 6rpx 10rpx rgba(255, 255, 255, 0.22), inset 0 -8rpx 14rpx rgba(0, 0, 0, 0.3),
    0 16rpx 24rpx rgba(0, 0, 0, 0.24);
  transform: translateZ(-22rpx) rotateX(66deg);
  transform-origin: top;
}
</style>
