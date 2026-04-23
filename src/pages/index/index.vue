<template>
  <view class="desk-page">
    <view class="desk-scene">
      <view class="desk-calendar">
        <view class="ring-row">
          <view v-for="n in 8" :key="n" class="ring" />
        </view>

        <view class="calendar-paper-stack">
          <view class="paper current" :class="{ 'is-flipping': isFlipping }">
            <view class="paper-face front">
              <image
                v-if="currentEntry.images[0]"
                class="bg-thumb"
                :src="currentEntry.images[0]"
                mode="aspectFill"
              />
              <view class="paper-content">
                <text class="date-number">{{ currentDay }}</text>
                <text class="date-line">{{ currentDateLabel }}</text>
                <text class="essay">{{ currentEntry.text || '今天写点什么吧。' }}</text>
              </view>
              <view class="summary-chip">
                <text class="chip-label">AI Summary</text>
              </view>
            </view>

            <view class="paper-face back">
              <view class="paper-content ghost">
                <text class="date-number">{{ currentDay }}</text>
              </view>
            </view>
          </view>

          <view class="paper next">
            <view class="paper-face front">
              <image
                v-if="nextEntry.images[0]"
                class="bg-thumb"
                :src="nextEntry.images[0]"
                mode="aspectFill"
              />
              <view class="paper-content">
                <text class="date-number">{{ nextDay }}</text>
                <text class="date-line">{{ nextDateLabel }}</text>
                <text class="essay">{{ nextEntry.text || '下一页已准备好。' }}</text>
              </view>
              <view class="summary-chip">
                <text class="chip-label">AI Summary</text>
              </view>
            </view>
          </view>
        </view>

        <view class="desk-base" />
      </view>

      <view class="action-row">
        <button class="flip-btn" :disabled="isFlipping" @tap="flipPrev">上一天</button>
        <button class="flip-btn" :disabled="isFlipping" @tap="flipNext">下一天</button>
      </view>
    </view>
  </view>
</template>

<script setup>
import { computed, reactive, ref } from 'vue'

const FLIP_MS = 760

const state = reactive({
  year: 2026,
  month: 4,
  day: 23,
  entries: {
    '2026-04-23': {
      text: '今天把随笔应用改造成拟物的立台日历，翻页更有仪式感。',
      images: [],
      aiSummary: '以 3D 透视和顶部轴翻页重构主界面。'
    },
    '2026-04-24': {
      text: '翻页动画完成后再更新日期数据，避免内容在动画中突变。',
      images: [],
      aiSummary: '用 isFlipping 锁定交互，提升稳定性。'
    }
  }
})

const isFlipping = ref(false)
const pendingDay = ref(null)

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

const currentDay = computed(() => state.day)
const currentDateLabel = computed(() => `${state.year}.${String(state.month).padStart(2, '0')}.${String(state.day).padStart(2, '0')}`)
const currentEntry = computed(() => getEntry(state.day))

const nextDay = computed(() => {
  if (pendingDay.value !== null) return pendingDay.value
  const d = state.day + 1
  return d > daysInMonth.value ? 1 : d
})
const nextDateLabel = computed(() => `${state.year}.${String(state.month).padStart(2, '0')}.${String(nextDay.value).padStart(2, '0')}`)
const nextEntry = computed(() => getEntry(nextDay.value))

function runFlip(targetDay) {
  if (isFlipping.value) return
  pendingDay.value = targetDay
  isFlipping.value = true
  setTimeout(() => {
    state.day = targetDay
    isFlipping.value = false
    pendingDay.value = null
  }, FLIP_MS)
}

function flipNext() {
  const d = state.day + 1
  runFlip(d > daysInMonth.value ? 1 : d)
}

function flipPrev() {
  const d = state.day - 1
  runFlip(d < 1 ? daysInMonth.value : d)
}
</script>

<style scoped lang="scss">
.desk-page {
  min-height: 100vh;
  padding: 40rpx 24rpx 56rpx;
  background: radial-gradient(circle at 20% 15%, #f9fafc 0%, #e6eaef 55%, #d8dce1 100%);
}

.desk-scene {
  width: 100%;
  max-width: 720rpx;
  margin: 0 auto;
  perspective: 1800rpx;
  perspective-origin: center 30%;
}

.desk-calendar {
  transform-style: preserve-3d;
}

.ring-row {
  display: flex;
  justify-content: center;
  gap: 18rpx;
  margin-bottom: -12rpx;
  position: relative;
  z-index: 4;
}

.ring {
  width: 18rpx;
  height: 18rpx;
  border-radius: 50%;
  border: 3rpx solid #8a8f98;
  background: linear-gradient(145deg, #dbdee4, #9da3ad);
  box-shadow: inset 0 2rpx 3rpx rgba(255, 255, 255, 0.55), 0 2rpx 4rpx rgba(0, 0, 0, 0.22);
}

.calendar-paper-stack {
  position: relative;
  height: 880rpx;
  transform-style: preserve-3d;
}

.paper {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  border-radius: 18rpx;
  transform-style: preserve-3d;
}

.paper-face {
  position: absolute;
  inset: 0;
  border-radius: 18rpx;
  overflow: hidden;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  background: linear-gradient(180deg, #fbfcfd 0%, #f5f6f8 100%);
}

.paper.current {
  z-index: 3;
  transform-origin: top center;
  transition: transform 760ms cubic-bezier(0.2, 0.72, 0.2, 1), opacity 760ms ease;
  box-shadow: 0 24rpx 38rpx rgba(0, 0, 0, 0.18), 0 6rpx 12rpx rgba(0, 0, 0, 0.12);
}

.paper.current.is-flipping {
  transform: rotateX(-180deg);
  opacity: 0.22;
}

.paper.current .back {
  transform: rotateX(180deg);
  background: linear-gradient(180deg, #eceff3 0%, #dde2e9 100%);
}

.paper.next {
  z-index: 1;
  transform: translateZ(-6rpx) translateY(8rpx) scale(0.992);
  box-shadow: 0 14rpx 24rpx rgba(0, 0, 0, 0.12);
}

.bg-thumb {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  opacity: 0.15;
}

.paper-content {
  position: relative;
  z-index: 2;
  height: 100%;
  display: flex;
  flex-direction: column;
  padding: 84rpx 56rpx 120rpx;
  box-sizing: border-box;
}

.paper-content.ghost {
  align-items: center;
  justify-content: center;
  opacity: 0.35;
}

.date-number {
  font-size: 168rpx;
  line-height: 1;
  font-weight: 800;
  color: #1f242c;
  letter-spacing: 3rpx;
}

.date-line {
  margin-top: 12rpx;
  font-size: 28rpx;
  color: #667081;
}

.essay {
  margin-top: 46rpx;
  font-size: 34rpx;
  line-height: 1.7;
  color: #2e3440;
  white-space: pre-wrap;
}

.summary-chip {
  position: absolute;
  left: 56rpx;
  right: 56rpx;
  bottom: 38rpx;
  height: 70rpx;
  border-radius: 12rpx;
  background: linear-gradient(90deg, rgba(33, 38, 48, 0.94), rgba(61, 69, 84, 0.94));
  display: flex;
  align-items: center;
  padding: 0 22rpx;
  box-sizing: border-box;
}

.chip-label {
  color: #f4f6fa;
  font-size: 24rpx;
  letter-spacing: 1rpx;
}

.desk-base {
  margin: 20rpx auto 0;
  width: 90%;
  height: 84rpx;
  border-radius: 20rpx;
  background: linear-gradient(135deg, #6f5038 0%, #8a684b 28%, #4f3828 100%);
  box-shadow: inset 0 6rpx 10rpx rgba(255, 255, 255, 0.22), inset 0 -8rpx 14rpx rgba(0, 0, 0, 0.28),
    0 18rpx 26rpx rgba(0, 0, 0, 0.25);
  transform: translateZ(-22rpx) rotateX(68deg);
  transform-origin: top;
}

.action-row {
  margin-top: 40rpx;
  display: flex;
  justify-content: center;
  gap: 24rpx;
}

.flip-btn {
  width: 220rpx;
  height: 78rpx;
  line-height: 78rpx;
  border: none;
  border-radius: 40rpx;
  color: #fff;
  background: linear-gradient(135deg, #4a5568, #2d3748);
  font-size: 28rpx;
}

.flip-btn[disabled] {
  opacity: 0.45;
}
</style>
