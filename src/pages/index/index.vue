<template>
  <view class="page" :class="{ 'is-editor-open': expandedKey }">
    <view class="calendar-wrap">
      <view class="week-row">
        <text v-for="w in weekLabels" :key="w" class="week-cell">{{ w }}</text>
      </view>

      <view class="month-grid">
        <view
          v-for="cell in paddedCells"
          :key="cell.key"
          class="day-cell"
          :class="{ 'is-empty': cell.type === 'pad', 'has-entry': cell.entry }"
          :id="'day-' + cell.key"
          @tap="cell.type === 'day' && onDayTap(cell)"
        >
          <template v-if="cell.type === 'day'">
            <text class="day-num">{{ cell.day }}</text>

            <view v-if="cell.entry" class="polaroid">
              <image
                v-if="cell.entry.images[0]"
                class="polaroid-img"
                :src="cell.entry.images[0]"
                mode="aspectFill"
              />
              <view class="polaroid-text">
                <text class="line">{{ cell.entry.previewLines[0] || '' }}</text>
                <text class="line">{{ cell.entry.previewLines[1] || '' }}</text>
              </view>
            </view>
          </template>
        </view>
      </view>
    </view>

    <view class="backdrop" :class="{ 'is-dim': expandedKey }" />

    <view
      v-if="expandedCell"
      class="editor-layer"
      :class="{ 'is-expanded': editorExpanded }"
      :style="layerStyle"
    >
      <view class="editor-inner">
        <view class="editor-head">
          <text class="back" @tap.stop="closeEditor">返回</text>
          <text class="big-date">{{ expandedCell.dateLabel }}</text>
        </view>

        <textarea
          class="note-input"
          :value="expandedCell.entry?.text"
          placeholder="写点什么…"
          placeholder-style="color: rgba(0,0,0,0.35)"
          @input="onNoteInput"
        />

        <view class="editor-foot">
          <scroll-view scroll-x class="thumb-row" :show-scrollbar="false">
            <image
              v-for="(src, i) in expandedCell.entry?.images || []"
              :key="i"
              class="thumb"
              :src="src"
              mode="aspectFill"
            />
          </scroll-view>
          <view class="ai-card">
            <text class="ai-title">AI 总结</text>
            <text class="ai-body">{{ expandedCell.entry?.aiSummary || '暂无总结' }}</text>
          </view>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup>
import { reactive, computed, ref, nextTick } from 'vue'

const ANIM_MS = 320

const state = reactive({
  year: 2026,
  month: 4,
  entries: {
    '2026-04-23': {
      text: '第一行\n第二行会有预览',
      images: [],
      previewLines: [],
      aiSummary: '今日要点摘录 …'
    }
  }
})

const weekLabels = ['日', '一', '二', '三', '四', '五', '六']

function splitPreview(text = '') {
  const lines = text.replace(/\r/g, '').split('\n').filter(Boolean)
  return [lines[0] || '', lines[1] || '']
}

function hydrateEntries() {
  Object.keys(state.entries).forEach((k) => {
    const e = state.entries[k]
    e.previewLines = splitPreview(e.text)
  })
}
hydrateEntries()

const daysInMonth = computed(() => new Date(state.year, state.month, 0).getDate())

const startWeekday = computed(() => new Date(state.year, state.month - 1, 1).getDay())

const paddedCells = computed(() => {
  const cells = []
  const { year, month } = state
  const pad = startWeekday.value
  for (let i = 0; i < pad; i++) {
    cells.push({ type: 'pad', key: `pad-${i}` })
  }
  for (let d = 1; d <= daysInMonth.value; d++) {
    const key = `${year}-${String(month).padStart(2, '0')}-${String(d).padStart(2, '0')}`
    cells.push({
      type: 'day',
      key,
      day: d,
      dateLabel: `${month}月${d}日`,
      entry: state.entries[key] || null
    })
  }
  return cells
})

const expandedKey = ref('')
const expandedCell = computed(() =>
  paddedCells.value.find((c) => c.key === expandedKey.value && c.type === 'day')
)

const editorExpanded = ref(false)
const layerStyle = ref({})

let lastRect = null

function getViewportSize() {
  const sys = uni.getSystemInfoSync()
  return { vw: sys.windowWidth, vh: sys.windowHeight }
}

/**
 * 格子 rect → 全屏 layer 的初始 transform（中心对齐 + 等比缩放到格子大小）
 */
function flipToFullscreen(rect) {
  const { vw, vh } = getViewportSize()
  const cx = rect.left + rect.width / 2
  const cy = rect.top + rect.height / 2
  const vx = vw / 2
  const vy = vh / 2
  const sx = rect.width / vw
  const sy = rect.height / vh
  return {
    transform: `translate(${cx - vx}px, ${cy - vy}px) scale(${sx}, ${sy})`,
    opacity: '1'
  }
}

async function onDayTap(cell) {
  expandedKey.value = cell.key
  editorExpanded.value = false

  await nextTick()

  const applyFlip = (rect) => {
    lastRect = rect
    layerStyle.value = flipToFullscreen(rect)
    requestAnimationFrame(() => {
      requestAnimationFrame(() => {
        editorExpanded.value = true
        layerStyle.value = {
          transform: 'translate(0px, 0px) scale(1, 1)',
          opacity: '1'
        }
      })
    })
  }

  uni.createSelectorQuery()
    .select(`#day-${cell.key}`)
    .boundingClientRect((rect) => {
      if (rect && rect.width) {
        applyFlip(rect)
        return
      }
      setTimeout(() => {
        uni.createSelectorQuery()
          .select(`#day-${cell.key}`)
          .boundingClientRect((r2) => {
            if (r2 && r2.width) applyFlip(r2)
          })
          .exec()
      }, 50)
    })
    .exec()
}

function closeEditor() {
  if (!lastRect) {
    expandedKey.value = ''
    layerStyle.value = {}
    return
  }
  editorExpanded.value = false
  layerStyle.value = flipToFullscreen(lastRect)
  setTimeout(() => {
    expandedKey.value = ''
    layerStyle.value = {}
    lastRect = null
  }, ANIM_MS)
}

function onNoteInput(e) {
  const key = expandedKey.value
  if (!state.entries[key]) {
    state.entries[key] = {
      text: '',
      images: [],
      previewLines: ['', ''],
      aiSummary: ''
    }
  }
  state.entries[key].text = e.detail.value
  state.entries[key].previewLines = splitPreview(e.detail.value)
}
</script>

<style scoped lang="scss">
.page {
  min-height: 100vh;
  position: relative;
  background: #fff;
}

.calendar-wrap {
  padding: 24rpx;
  transition: opacity 0.35s ease;
}

.page.is-editor-open .calendar-wrap {
  opacity: 0;
  pointer-events: none;
}

.week-row,
.month-grid {
  display: flex;
  flex-wrap: wrap;
}

.week-cell,
.day-cell {
  width: 14.2857%;
  box-sizing: border-box;
}

.week-cell {
  text-align: center;
  font-size: 22rpx;
  color: #888;
  padding: 8rpx 0;
}

.day-cell {
  aspect-ratio: 3 / 4;
  border: 1px solid #eee;
  padding: 8rpx;
  display: flex;
  flex-direction: column;
  gap: 6rpx;
}

.day-cell.is-empty {
  border-color: transparent;
}

.day-num {
  font-size: 22rpx;
  color: #333;
}

.polaroid {
  flex: 1;
  background: #faf7f2;
  border-radius: 10rpx;
  box-shadow: 0 6rpx 16rpx rgba(0, 0, 0, 0.08);
  padding: 8rpx;
  overflow: hidden;
}

.polaroid-img {
  width: 100%;
  height: 72rpx;
  border-radius: 6rpx;
}

.polaroid-text .line {
  display: block;
  font-size: 18rpx;
  color: #444;
  line-height: 1.35;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}

.backdrop {
  pointer-events: none;
  transition: opacity 0.35s ease;
}

.backdrop.is-dim {
  opacity: 0;
}

.editor-layer {
  position: fixed;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  z-index: 999;
  background: rgba(255, 255, 255, 0.92);
  transform-origin: center center;
  will-change: transform, opacity;
  transition: transform 320ms cubic-bezier(0.22, 1, 0.36, 1), opacity 320ms ease;
}

.editor-inner {
  height: 100%;
  display: flex;
  flex-direction: column;
  padding: calc(env(safe-area-inset-top) + 24rpx) 24rpx calc(env(safe-area-inset-bottom) + 24rpx);
  box-sizing: border-box;
}

.editor-head {
  margin-bottom: 16rpx;
}

.back {
  font-size: 28rpx;
  color: #007aff;
}

.big-date {
  display: block;
  margin-top: 12rpx;
  font-size: 44rpx;
  font-weight: 700;
  letter-spacing: 0.06em;
}

.note-input {
  flex: 1;
  width: 100%;
  min-height: 200rpx;
  background: transparent;
  font-size: 30rpx;
  line-height: 1.6;
}

.editor-foot {
  margin-top: 16rpx;
}

.thumb-row {
  white-space: nowrap;
  margin-bottom: 12rpx;
}

.thumb {
  width: 120rpx;
  height: 120rpx;
  border-radius: 12rpx;
  margin-right: 12rpx;
  display: inline-block;
  vertical-align: top;
}

.ai-card {
  background: #f6f7fb;
  border-radius: 16rpx;
  padding: 16rpx;
}

.ai-title {
  font-size: 24rpx;
  color: #666;
}

.ai-body {
  display: block;
  margin-top: 8rpx;
  font-size: 26rpx;
  color: #222;
}
</style>
