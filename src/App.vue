<script setup>
import { computed, onMounted, onBeforeUnmount, ref } from 'vue'
import DeviceCanvas from './components/DeviceCanvas.vue'
import DeviceEditModal from './components/DeviceEditModal.vue'
import DeviceForm from './components/DeviceForm.vue'

const PADDING = 2000
const canvasBounds = ref({ minX: 0, minY: 0, width: 800000, height: 200000 })
const zoom = ref(0.01)
const zoomStep = 0.01
const minZoom = 0.001
const maxZoom = 1

const devices = ref([])
const devicesLoading = ref(true)
const bgSvgUrl = ref(null)
const bgSvgMeta = ref(null)
const showBg = ref(false)

/** 从数据计算画布边界（兼容负坐标），可选纳入背景 bbox */
const computeBounds = (data, bgBbox) => {
  if ((!data || data.length === 0) && !bgBbox) {
    return { minX: 0, minY: 0, width: 800000, height: 200000 }
  }
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity
  for (const d of (data || [])) {
    const x = Number(d.x)
    const y = Number(d.y)
    const w = Number(d.width) || 0
    const h = Number(d.height) || 0
    if (x < minX) minX = x
    if (y < minY) minY = y
    if (x + w > maxX) maxX = x + w
    if (y + h > maxY) maxY = y + h
  }
  // 纳入背景 bbox
  if (bgBbox) {
    if (bgBbox.minX < minX) minX = bgBbox.minX
    if (bgBbox.minY < minY) minY = bgBbox.minY
    if (bgBbox.maxX > maxX) maxX = bgBbox.maxX
    if (bgBbox.maxY > maxY) maxY = bgBbox.maxY
  }
  // 给边界留 padding
  return {
    minX: minX - PADDING,
    minY: minY - PADDING,
    width: (maxX - minX) + PADDING * 2,
    height: (maxY - minY) + PADDING * 2,
  }
}

/** 生成 id — direction_flag-文字-bay_occupation 连字符格式 */
const makeId = (d, idx) => {
  if (d.id && String(d.id).trim()) return String(d.id).trim()
  const info = d.info || {}
  const dir = info.direction_flag ? String(info.direction_flag).trim() : ''
  const label = info.文字 ? String(info.文字).trim() : ''
  const bay = info.bay_occupation ? String(info.bay_occupation).trim() : ''
  const parts = [dir, label, bay].filter(Boolean)
  return parts.length > 0 ? parts.join('-') : `M-${idx + 1}`
}

onMounted(async () => {
  try {
    const res = await fetch('/output_eq.json')
    if (res.ok) {
      const data = await res.json()
      if (Array.isArray(data) && data.length > 0) {
        // 先算边界（依赖原始坐标）
        canvasBounds.value = computeBounds(data)

        devices.value = data.map((d, i) => ({
          id: makeId(d, i),
          x: Number(d.x),
          y: Number(d.y),
          width: Number(d.width),
          height: Number(d.height),
          info: d.info || null,
          _isDetail: !!(d.id && String(d.id).trim()),
        }))
      }
    }
  } catch {
    // output_eq.json not available, start with empty devices
  } finally {
    devicesLoading.value = false
  }

  // 加载背景 SVG
  try {
    const [svgRes, metaRes] = await Promise.all([
      fetch('/raw_layer.svg'),
      fetch('/raw_layer_meta.json'),
    ])
    if (svgRes.ok && metaRes.ok) {
      const svgText = await svgRes.text()
      const blob = new Blob([svgText], { type: 'image/svg+xml' })
      bgSvgUrl.value = URL.createObjectURL(blob)
      bgSvgMeta.value = await metaRes.json()
      // 将背景 bbox 纳入画布边界，确保全量可见
      if (bgSvgMeta.value && bgSvgMeta.value.bbox) {
        canvasBounds.value = computeBounds(devices.value, bgSvgMeta.value.bbox)
        // 等 DOM 更新后触发自适应
        await new Promise(resolve => setTimeout(resolve, 100))
        if (canvasRef.value) {
          canvasRef.value.autoFit()
        }
      }
    }
  } catch {
    // 无背景 SVG 时正常降级
  }

  window.addEventListener('keydown', handleKeydown)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeydown)
})

const selectedId = ref(null)
const isEditOpen = ref(false)

const selectedDevice = computed(() =>
  devices.value.find((device) => device.id === selectedId.value) || null
)

const existingIds = computed(() => devices.value.map((device) => device.id))

const addDevice = (device) => {
  devices.value = [...devices.value, device]
}

const openEdit = (deviceId) => {
  if (!deviceId) {
    selectedId.value = null
    isEditOpen.value = false
    return
  }

  selectedId.value = deviceId
  isEditOpen.value = true
}

const closeEdit = () => {
  isEditOpen.value = false
}

const saveEdit = (updated) => {
  devices.value = devices.value.map((device) =>
    device.id === selectedId.value ? updated : device
  )
  selectedId.value = updated.id
  isEditOpen.value = false
}

const deleteDevice = () => {
  if (!selectedId.value) {
    return
  }
  const confirmed = window.confirm('Delete this device? This cannot be undone.')
  if (!confirmed) {
    return
  }
  devices.value = devices.value.filter((device) => device.id !== selectedId.value)
  selectedId.value = null
  isEditOpen.value = false
}

const clampZoom = (value) => Math.min(maxZoom, Math.max(minZoom, value))

const setZoom = (value) => {
  if (!Number.isFinite(value)) {
    return
  }
  zoom.value = clampZoom(value)
}

const zoomIn = () => {
  setZoom(zoom.value + zoomStep)
}

const zoomOut = () => {
  setZoom(zoom.value - zoomStep)
}

const isFullscreen = ref(false)

const toggleFullscreen = async () => {
  isFullscreen.value = !isFullscreen.value

  // 同步更新 body class（比 CSS :has() 更可靠）
  if (isFullscreen.value) {
    document.body.classList.add('is-fullscreen')
  } else {
    document.body.classList.remove('is-fullscreen')
  }

  // 等待 DOM 更新 + CSS 过渡完成 + 浏览器布局计算
  await new Promise(resolve => setTimeout(resolve, 150))

  // 触发 resize 事件让 DeviceCanvas 重新计算尺寸
  window.dispatchEvent(new Event('resize'))

  // 再等待一帧确保 ResizeObserver 已触发
  await new Promise(resolve => requestAnimationFrame(resolve))

  if (canvasRef.value) {
    canvasRef.value.autoFit()
  }
}

const handleKeydown = (e) => {
  if (e.key === 'Escape' && isFullscreen.value) {
    toggleFullscreen()
  }
}

const canvasRef = ref(null)

const searchId = ref('')
const searchError = ref('')

const searchDevice = () => {
  const query = searchId.value.trim()
  searchError.value = ''

  if (!query) {
    searchError.value = '请输入机台 ID'
    return
  }

  const device = devices.value.find(
    (item) => item._isDetail && item.id.toLocaleLowerCase() === query.toLocaleLowerCase()
  )

  if (!device) {
    selectedId.value = null
    searchError.value = `未找到 ID 为“${query}”的机台`
    return
  }

  searchId.value = device.id
  selectedId.value = device.id
  isEditOpen.value = false
  canvasRef.value?.focusDevice(device.id)
}

const fitView = () => {
  if (canvasRef.value && devices.value.length > 0) {
    canvasRef.value.autoFit()
  }
}

const resetZoom = () => {
  setZoom(0.01)
}
</script>

<template>
  <div class="page" :class="{ 'page--fullscreen': isFullscreen }">
    <header class="page__header">
      <div>
        <p class="eyebrow">Semiconductor Floor Tools</p>
        <h1>Equipment Layout Manager</h1>
        <p class="subtitle">
          Add equipment by ID and coordinates, then click any rectangle to edit
          its details.
        </p>
      </div>
      <div class="meta">
        <div class="meta__item">
          <span class="meta__label">Canvas</span>
          <span class="meta__value">{{ canvasBounds.width.toFixed(0) }} x {{ canvasBounds.height.toFixed(0) }}</span>
        </div>
        <div class="meta__item">
          <span class="meta__label">Devices</span>
          <span class="meta__value">{{ devicesLoading ? '...' : devices.length }}</span>
        </div>
      </div>
    </header>

    <main class="page__main" :class="{ 'page__main--fullscreen': isFullscreen }">
      <section v-show="!isFullscreen" class="panel panel--form">
        <h2>Add Equipment</h2>
        <DeviceForm :existing-ids="existingIds" @add="addDevice" />
      </section>

      <section class="panel panel--canvas">
        <div class="panel__header">
          <h2>Layout Canvas</h2>
          <div class="panel__controls">
            <p class="hint">Origin is bottom-left. X → right, Y ↑ up. Click a rectangle to edit.</p>
            <form class="device-search" role="search" @submit.prevent="searchDevice">
              <label class="device-search__label" for="device-id-search">机台 ID</label>
              <input
                id="device-id-search"
                v-model="searchId"
                class="device-search__input"
                type="search"
                placeholder="例如 EQ-1001"
                autocomplete="off"
                @input="searchError = ''"
              />
              <button class="icon-button device-search__button" type="submit">搜索</button>
              <span v-if="searchError" class="device-search__error" role="alert">{{ searchError }}</span>
            </form>
            <div class="zoom">
              <button class="icon-button" type="button" @click="zoomOut">-</button>
              <input
                v-model.number="zoom"
                class="zoom__range"
                type="range"
                :min="minZoom"
                :max="maxZoom"
                :step="zoomStep"
              />
              <button class="icon-button" type="button" @click="zoomIn">+</button>
              <button class="icon-button" type="button" @click="resetZoom">Reset</button>
              <button class="icon-button" type="button" @click="fitView">Fit</button>
              <button class="icon-button" type="button" @click="toggleFullscreen">{{ isFullscreen ? 'Exit Fullscreen' : 'Fullscreen' }}</button>
              <button
                v-if="bgSvgUrl"
                class="icon-button"
                :class="{ 'icon-button--active': showBg }"
                type="button"
                @click="showBg = !showBg"
              >{{ showBg ? 'Hide BG' : 'Show BG' }}</button>
              <span class="zoom__value">{{ Math.round(zoom * 100) }}%</span>
            </div>
          </div>
        </div>
        <DeviceCanvas
          ref="canvasRef"
          :devices="devices"
          :selected-id="selectedId"
          :min-x="canvasBounds.minX"
          :min-y="canvasBounds.minY"
          :width="canvasBounds.width"
          :height="canvasBounds.height"
          :zoom="zoom"
          :bg-svg-url="bgSvgUrl"
          :bg-svg-meta="bgSvgMeta"
          :show-bg="showBg"
          @select="openEdit"
          @zoom="setZoom"
        />
      </section>
    </main>

    <DeviceEditModal
      :open="isEditOpen"
      :device="selectedDevice"
      :existing-ids="existingIds"
      @close="closeEdit"
      @save="saveEdit"
      @delete="deleteDevice"
    />
  </div>
</template>
