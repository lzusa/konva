<script setup>
import { computed, ref } from 'vue'
import DeviceCanvas from './components/DeviceCanvas.vue'
import DeviceEditModal from './components/DeviceEditModal.vue'
import DeviceForm from './components/DeviceForm.vue'

const canvasSize = { width: 12000, height: 8000 }
const zoom = ref(0.4)
const zoomStep = 0.1
const minZoom = 0.1
const maxZoom = Math.min(2, 16000 / canvasSize.width, 16000 / canvasSize.height)

const devices = ref([
  { id: 'EQ-1001', x: 80, y: 60, width: 180, height: 120 },
  { id: 'EQ-1002', x: 80, y: 260, width: 220, height: 140 },
  { id: 'EQ-1003', x: 80, y: 500, width: 200, height: 160 },
  { id: 'EQ-2001', x: 400, y: 60, width: 160, height: 160 },
  { id: 'EQ-2002', x: 400, y: 260, width: 240, height: 120 },
  { id: 'EQ-2003', x: 400, y: 500, width: 180, height: 140 },

  { id: 'EQ-3001', x: 720, y: 60, width: 200, height: 100 },
  { id: 'EQ-3002', x: 720, y: 200, width: 180, height: 180 },
])

const selectedId = ref(null)
const isEditOpen = ref(false)

const selectedDevice = computed(() =>
  devices.value.find((device) => device.id === selectedId.value) || null
)

const existingIds = computed(() => devices.value.map((device) => device.id))

const overlapIds = computed(() => {
  const overlaps = new Set()
  const list = devices.value

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
            <p class="hint">Origin is top-left. Click a rectangle to edit.</p>
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
