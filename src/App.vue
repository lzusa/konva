<script setup>
import { computed, ref } from 'vue'
import DeviceCanvas from './components/DeviceCanvas.vue'
import DeviceEditModal from './components/DeviceEditModal.vue'
import DeviceForm from './components/DeviceForm.vue'

const canvasSize = { width: 800000, height: 200000 }
const zoom = ref(0.01)
const zoomStep = 0.01
const minZoom = 0.001
const maxZoom = 1

const selectedId = ref(null)
const isEditOpen = ref(false)

const selectedDevice = computed(() =>
  devices.value.find((device) => device.id === selectedId.value) || null
)

const existingIds = computed(() => devices.value.map((device) => device.id))

const overlapIds = computed(() => {
  const list = devices.value
  // P4: 当设备数量较少时直接用 O(N²)，数量多时用空间哈希
  if (list.length < 50) {
    const overlaps = new Set()
    for (let i = 0; i < list.length; i += 1) {
      const a = list[i]
      for (let j = i + 1; j < list.length; j += 1) {
        const b = list[j]
        const separated =
          a.x + a.width <= b.x ||
          b.x + b.width <= a.x ||
          a.y + a.height <= b.y ||
          b.y + b.height <= a.y
        if (!separated) {
          overlaps.add(a.id)
          overlaps.add(b.id)
        }
      }
    }
    return Array.from(overlaps)
  }

  // 空间哈希：O(N) 平均复杂度
  const cellSize = 500
  const grid = new Map()
  for (let i = 0; i < list.length; i++) {
    const d = list[i]
    const cx0 = Math.floor(d.x / cellSize)
    const cy0 = Math.floor(d.y / cellSize)
    const cx1 = Math.floor((d.x + d.width) / cellSize)
    const cy1 = Math.floor((d.y + d.height) / cellSize)
    for (let cx = cx0; cx <= cx1; cx++) {
      for (let cy = cy0; cy <= cy1; cy++) {
        const key = `${cx},${cy}`
        if (!grid.has(key)) grid.set(key, [])
        grid.get(key).push(i)
      }
    }
  }

  const overlaps = new Set()
  const checked = new Set()
  for (const [key, indices] of grid.entries()) {
    for (let i = 0; i < indices.length; i++) {
      for (let j = i + 1; j < indices.length; j++) {
        const pairKey = indices[i] < indices[j] ? `${indices[i]}-${indices[j]}` : `${indices[j]}-${indices[i]}`
        if (checked.has(pairKey)) continue
        checked.add(pairKey)

        const a = list[indices[i]]
        const b = list[indices[j]]
        const separated =
          a.x + a.width <= b.x ||
          b.x + b.width <= a.x ||
          a.y + a.height <= b.y ||
          b.y + b.height <= a.y
        if (!separated) {
          overlaps.add(a.id)
          overlaps.add(b.id)
        }
      }
    }
  }

  return Array.from(overlaps)
})

const hasOverlap = computed(() => overlapIds.value.length > 0)

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

const resetZoom = () => {
  setZoom(0.4)
}
</script>

<template>
  <div class="page">
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
          <span class="meta__value">{{ canvasSize.width }} x {{ canvasSize.height }}</span>
        </div>
        <div class="meta__item">
          <span class="meta__label">Devices</span>
          <span class="meta__value">{{ devices.length }}</span>
        </div>
      </div>
    </header>

    <main class="page__main">
      <section class="panel panel--form">
        <h2>Add Equipment</h2>
        <DeviceForm :existing-ids="existingIds" @add="addDevice" />
      </section>

      <section class="panel panel--canvas">
        <div class="panel__header">
          <h2>Layout Canvas</h2>
          <div class="panel__controls">
            <p class="hint">Origin is bottom-left. X → right, Y ↑ up. Click a rectangle to edit.</p>
            <p v-if="hasOverlap" class="warning-banner">
              Overlap detected: {{ overlapIds.join(', ') }}
            </p>
            <div class="zoom">
              <button class="icon-button" type="button" @click="zoomOut">-</button>
              <input
                v-model.number="zoom"
                class="zoom__range"
                type="range"
                min="0.1"
                :max="maxZoom"
                step="0.05"
              />
              <button class="icon-button" type="button" @click="zoomIn">+</button>
              <button class="icon-button" type="button" @click="resetZoom">Reset</button>
              <span class="zoom__value">{{ Math.round(zoom * 100) }}%</span>
            </div>
          </div>
        </div>
        <DeviceCanvas
          :devices="devices"
          :selected-id="selectedId"
          :width="canvasSize.width"
          :height="canvasSize.height"
          :zoom="zoom"
          :overlap-ids="overlapIds"
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
