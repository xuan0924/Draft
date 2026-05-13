<template>
  <view class="jot-page">
    <view class="jot-header-glass">
      <view class="jot-header-inner">
        <view class="jot-header-left">
          <view class="jot-month-nav" @click="changeMonth(-1)">
            <text class="jot-month-nav-icon">‹</text>
          </view>
          <text class="jot-month-title">{{ monthName }}</text>
          <view class="jot-month-nav" @click="changeMonth(1)">
            <text class="jot-month-nav-icon">›</text>
          </view>
        </view>
        <view class="jot-header-right">
          <view class="jot-year-btn" @click="changeYear(-1)">
            <text class="jot-year-icon">↑</text>
          </view>
          <text class="jot-year-label">{{ state.year }}</text>
          <view class="jot-year-btn" @click="changeYear(1)">
            <text class="jot-year-icon">↓</text>
          </view>
        </view>
      </view>
    </view>

    <view class="jot-calendar">
      <view class="jot-week-row">
        <text v-for="w in weekdayLabels" :key="w" class="jot-week-cell">{{ w }}</text>
      </view>
      <view class="jot-day-grid">
        <view
          v-for="(cell, idx) in calendarCells"
          :key="'c-' + idx"
          class="jot-day-slot"
          @click="onCellClick(cell)"
        >
          <view v-if="cell.day" class="jot-day-cell">
            <view
              class="jot-day-num-wrap"
              :class="{ 'is-selected': cell.day === state.day }"
            >
              <text class="jot-day-num" :class="{ 'is-selected': cell.day === state.day }">
                {{ cell.day }}
              </text>
            </view>
            <view v-if="hasRecord(cell.day)" class="jot-day-dot" />
          </view>
        </view>
      </view>
    </view>

    <scroll-view class="jot-list-scroll" scroll-y :show-scrollbar="false">
      <view class="jot-list-anim" :animation="listAnim">
        <view v-if="currentEntry.text || currentEntry.aiSummary || (currentEntry.images && currentEntry.images.length)" class="jot-card">
          <text class="jot-card-date">{{ currentLabel }}</text>
          <text class="jot-card-body">{{ currentEntry.text || '（无正文）' }}</text>
        </view>

        <view
          v-if="currentEntry.aiSummary"
          class="jot-ai-sheet"
        >
          <text class="jot-ai-title">AI 摘要</text>
          <text class="jot-ai-body">{{ currentEntry.aiSummary }}</text>
        </view>

        <view
          v-if="!currentEntry.text && !currentEntry.aiSummary && !(currentEntry.images && currentEntry.images.length)"
          class="jot-card jot-card--empty"
        >
          <text class="jot-empty-text">这一天还没有随笔</text>
        </view>
      </view>
      <view class="jot-list-bottom-space" />
    </scroll-view>
  </view>
</template>

<script setup>
import { computed, nextTick, onMounted, reactive, ref, watch } from 'vue'

const weekdayLabels = ['日', '一', '二', '三', '四', '五', '六']

const MONTH_NAMES = [
  '一月',
  '二月',
  '三月',
  '四月',
  '五月',
  '六月',
  '七月',
  '八月',
  '九月',
  '十月',
  '十一月',
  '十二月'
]

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

const listAnim = ref({})
const skipListAnim = ref(true)

const daysInMonth = computed(() => new Date(state.year, state.month, 0).getDate())

const monthName = computed(() => MONTH_NAMES[state.month - 1] || '')

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

function hasRecord(day) {
  const e = getEntry(day)
  return !!(e.text || e.aiSummary || (e.images && e.images.length))
}

const firstWeekday = computed(() => new Date(state.year, state.month - 1, 1).getDay())

const calendarCells = computed(() => {
  const dim = daysInMonth.value
  const pad = firstWeekday.value
  const list = []
  for (let i = 0; i < pad; i++) list.push({ day: 0 })
  for (let d = 1; d <= dim; d++) list.push({ day: d })
  return list
})

const currentEntry = computed(() => getEntry(state.day))

const currentLabel = computed(
  () => `${state.year}.${String(state.month).padStart(2, '0')}.${String(state.day).padStart(2, '0')}`
)

function clampDayToMonth() {
  const dim = daysInMonth.value
  if (state.day > dim) state.day = dim
  if (state.day < 1) state.day = 1
}

watch([() => state.year, () => state.month], () => {
  clampDayToMonth()
})

function playListEnter() {
  const anim = uni.createAnimation({
    duration: 300,
    timingFunction: 'cubic-bezier(0.25, 0.1, 0.25, 1)'
  })
  anim.opacity(0).translateY(36).step({ duration: 20 })
  anim.opacity(1).translateY(0).step({ duration: 280 })
  listAnim.value = anim.export()
}

function selectDay(day) {
  if (!day || day === state.day) return
  state.day = day
  if (skipListAnim.value) return
  playListEnter()
}

function changeMonth(delta) {
  let m = state.month + delta
  let y = state.year
  if (m < 1) {
    m = 12
    y -= 1
  } else if (m > 12) {
    m = 1
    y += 1
  }
  state.year = y
  state.month = m
  clampDayToMonth()
}

function changeYear(delta) {
  state.year += delta
  clampDayToMonth()
}

function onCellClick(cell) {
  if (!cell.day) return
  selectDay(cell.day)
}

onMounted(() => {
  nextTick(() => {
    skipListAnim.value = false
  })
})
</script>

<style scoped lang="scss">
.jot-page {
  min-height: 100vh;
  height: 100vh;
  display: flex;
  flex-direction: column;
  padding-bottom: env(safe-area-inset-bottom);
  background: #f2f2f7;
  box-sizing: border-box;
}

.jot-header-glass {
  position: sticky;
  top: 0;
  z-index: 20;
  padding-top: calc(env(safe-area-inset-top) + 12rpx);
  padding-bottom: 16rpx;
  background: rgba(242, 242, 247, 0.72);
  -webkit-backdrop-filter: blur(20px);
  backdrop-filter: blur(20px);
  border-bottom: 1rpx solid rgba(60, 60, 67, 0.12);
}

.jot-header-inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 28rpx 0 24rpx;
}

.jot-header-left {
  display: flex;
  align-items: center;
  gap: 4rpx;
}

.jot-month-nav {
  width: 56rpx;
  height: 56rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
}

.jot-month-nav-icon {
  font-size: 40rpx;
  font-weight: 300;
  color: rgba(60, 60, 67, 0.65);
  line-height: 1;
}

.jot-month-title {
  font-size: 52rpx;
  font-weight: 700;
  letter-spacing: 1rpx;
  color: #000000;
  line-height: 1.1;
}

.jot-header-right {
  display: flex;
  align-items: center;
  gap: 8rpx;
}

.jot-year-btn {
  width: 48rpx;
  height: 48rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
}

.jot-year-icon {
  font-size: 26rpx;
  color: rgba(60, 60, 67, 0.55);
  line-height: 1;
}

.jot-year-label {
  font-size: 28rpx;
  font-weight: 600;
  color: rgba(60, 60, 67, 0.85);
  min-width: 72rpx;
  text-align: center;
}

.jot-calendar {
  padding: 8rpx 20rpx 24rpx;
}

.jot-week-row {
  display: flex;
  flex-direction: row;
  margin-bottom: 8rpx;
}

.jot-week-cell {
  flex: 1;
  text-align: center;
  font-size: 22rpx;
  font-weight: 500;
  color: rgba(60, 60, 67, 0.45);
  line-height: 1.2;
}

.jot-day-grid {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
}

.jot-day-slot {
  width: 14.2857%;
  padding: 10rpx 0 14rpx;
  box-sizing: border-box;
}

.jot-day-cell {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  gap: 8rpx;
}

.jot-day-num-wrap {
  width: 64rpx;
  height: 64rpx;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
}

.jot-day-num-wrap.is-selected {
  background: #ff3b30;
}

.jot-day-num {
  font-size: 30rpx;
  font-weight: 500;
  color: #000000;
  line-height: 1;
}

.jot-day-num.is-selected {
  color: #ffffff;
  font-weight: 600;
}

.jot-day-dot {
  width: 8rpx;
  height: 8rpx;
  border-radius: 50%;
  background: rgba(60, 60, 67, 0.35);
}

.jot-list-scroll {
  flex: 1;
  height: 0;
  min-height: 200rpx;
  padding: 0 24rpx;
  box-sizing: border-box;
}

.jot-list-anim {
  transform: translateZ(0);
}

.jot-card {
  background: #ffffff;
  border-radius: 24rpx;
  padding: 28rpx 28rpx 32rpx;
  margin-bottom: 20rpx;
  border: 1rpx solid rgba(60, 60, 67, 0.08);
  box-shadow: 0 4rpx 24rpx rgba(0, 0, 0, 0.04);
}

.jot-card--empty {
  padding: 48rpx 28rpx;
  align-items: center;
}

.jot-card-date {
  display: block;
  font-size: 24rpx;
  font-weight: 500;
  color: rgba(60, 60, 67, 0.5);
  margin-bottom: 16rpx;
}

.jot-card-body {
  display: block;
  font-size: 32rpx;
  line-height: 1.65;
  color: #1c1c1e;
  white-space: pre-wrap;
}

.jot-empty-text {
  display: block;
  text-align: center;
  font-size: 30rpx;
  color: rgba(60, 60, 67, 0.45);
}

.jot-ai-sheet {
  margin-top: 4rpx;
  margin-bottom: 20rpx;
  padding: 28rpx 28rpx 32rpx;
  border-radius: 28rpx;
  background: rgba(255, 255, 255, 0.96);
  border: 1rpx solid rgba(60, 60, 67, 0.1);
  box-shadow: 0 8rpx 40rpx rgba(0, 0, 0, 0.08);
}

.jot-ai-title {
  display: block;
  font-size: 26rpx;
  font-weight: 600;
  color: rgba(60, 60, 67, 0.55);
  margin-bottom: 12rpx;
  letter-spacing: 0.5rpx;
}

.jot-ai-body {
  display: block;
  font-size: 30rpx;
  line-height: 1.55;
  color: #1c1c1e;
}

.jot-list-bottom-space {
  height: 48rpx;
}
</style>
