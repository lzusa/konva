<script setup>
import { onBeforeUnmount, onMounted, ref, watch } from 'vue'
import Konva from 'konva'

const props = defineProps({
  devices: {
    type: Array,
    default: () => []
  },
  selectedId: {
    type: String,
    default: null
  },
  width: {
    type: Number,
    default: 1200
  },
  height: {
    type: Number,
    default: 800
  },
  zoom: {
    type: Number,
    default: 1
  },
  overlapIds: {
    type: Array,
    default: () => []
  }
})

const emit = defineEmits(['select', 'zoom'])

const containerRef = ref(null)
let stage = null
let layer = null
let gridLayer = null
let resizeObserver = null
let bgRect = null

const gridSpacing = 100
const gridColors = ['#f6f7fb', '#ffffff']

const deviceNodes = new Map()

const createGridPattern = () => {
  const canvas = document.createElement('canvas')
  canvas.width = gridSpacing * 2
  canvas.height = gridSpacing * 2
  const ctx = canvas.getContext('2d')
  ctx.fillStyle = gridColors[0]
  ctx.fillRect(0, 0, gridSpacing, gridSpacing)
  ctx.fillRect(gridSpacing, gridSpacing, gridSpacing, gridSpacing)
  ctx.fillStyle = gridColors[1]
  ctx.fillRect(gridSpacing, 0, gridSpacing, gridSpacing)
  ctx.fillRect(0, gridSpacing, gridSpacing, gridSpacing)
  return canvas
}

const getContainerSize = () => {
  if (!containerRef.value) {
    return { width: 0, height: 0 }
  }

  const rect = containerRef.value.getBoundingClientRect()
  return {
    width: Math.max(1, Math.floor(rect.width)),
    height: Math.max(1, Math.floor(rect.height))
  }
}

const applyStageSize = () => {
  if (!stage) {
    return
  }
  const { width, height } = getContainerSize()
  if (width > 0 && height > 0) {
    stage.size({ width, height })
  }
}

const clampStagePosition = () => {
  if (!stage) {
    return
  }

  const scale = stage.scaleX()
  const viewWidth = stage.width()
  const viewHeight = stage.height()
  const worldWidth = props.width * scale
  const worldHeight = props.height * scale

  const minX = Math.min(0, viewWidth - worldWidth)
  const minY = Math.min(0, viewHeight - worldHeight)

  const clampedX = Math.min(0, Math.max(minX, stage.x()))
  const clampedY = Math.min(0, Math.max(minY, stage.y()))

  if (clampedX !== stage.x() || clampedY !== stage.y()) {
    stage.position({ x: clampedX, y: clampedY })
  }
}

const createStage = () => {
  const { width, height } = getContainerSize()
  stage = new Konva.Stage({
    container: containerRef.value,
    width,
    height
  })

  stage.scale({ x: props.zoom, y: props.zoom })
  stage.draggable(true)

  gridLayer = new Konva.FastLayer({ listening: false })
  stage.add(gridLayer)

  layer = new Konva.Layer()
  stage.add(layer)

  stage.on('click', (event) => {
    if (event.target === stage) {
      emit('select', null)
    }
  })

  stage.on('wheel', (event) => {
    event.evt.preventDefault()
    const delta = event.evt.deltaY
    const oldScale = stage.scaleX()
    const scaleBy = 1.05
    const nextZoom = delta > 0 ? oldScale / scaleBy : oldScale * scaleBy
    const pointer = stage.getPointerPosition()

    if (pointer) {
      const mousePointTo = {
        x: (pointer.x - stage.x()) / oldScale,
        y: (pointer.y - stage.y()) / oldScale
      }
      const newPos = {
        x: pointer.x - mousePointTo.x * nextZoom,
        y: pointer.y - mousePointTo.y * nextZoom
      }
      stage.position(newPos)
    }

    emit('zoom', nextZoom)
    clampStagePosition()
    stage.batchDraw()
  })

  stage.on('dragmove', () => {
    clampStagePosition()
  })

  stage.on('dragend', () => {
    clampStagePosition()
  })
}

const renderDevices = () => {
  if (!layer) {
    return
  }

  const overlapSet = new Set(props.overlapIds)
  const currentIds = new Set()

  props.devices.forEach((device) => {
    currentIds.add(device.id)
    const isSelected = device.id === props.selectedId
    const isOverlap = overlapSet.has(device.id)
    const fill = isOverlap ? '#ffb3b3' : isSelected ? '#ffd4a5' : '#9ec4ff'
    const stroke = isOverlap ? '#c81e1e' : isSelected ? '#e46c0a' : '#2a4b8d'
    const strokeWidth = isSelected ? 3 : 2

    let group = deviceNodes.get(device.id)
    if (!group) {
      group = new Konva.Group()

      const rect = new Konva.Rect({
        name: 'rect',
        cornerRadius: 6,
        shadowColor: 'rgba(0, 0, 0, 0.15)',
        shadowBlur: 8,
        shadowOffset: { x: 2, y: 2 }
      })

      const label = new Konva.Text({
        name: 'label',
        fontSize: 16,
        fontFamily: 'Space Grotesk, system-ui, sans-serif',
        fontStyle: '600',
        fill: '#0f1c3f',
        align: 'center'
      })

      rect.on('click', () => emit('select', device.id))
      label.on('click', () => emit('select', device.id))

      group.add(rect)
      group.add(label)
      layer.add(group)
      deviceNodes.set(device.id, group)
    }

    group.position({ x: device.x, y: device.y })

    const rect = group.findOne('.rect')
    rect.setAttrs({
      width: device.width,
      height: device.height,
      fill,
      stroke,
      strokeWidth
    })

    const label = group.findOne('.label')
    label.setAttrs({
      y: device.height / 2 - 10,
      width: device.width,
      text: device.id
    })
  })

  for (const [id, group] of deviceNodes.entries()) {
    if (!currentIds.has(id)) {
      group.destroy()
      deviceNodes.delete(id)
    }
  }

  layer.batchDraw()
}

const renderGrid = () => {
  if (!gridLayer) {
    return
  }

  if (!bgRect) {
    bgRect = new Konva.Rect({
      x: 0,
      y: 0,
      width: props.width,
      height: props.height,
      fillPatternImage: createGridPattern(),
      fillPatternRepeat: 'repeat'
    })
    gridLayer.add(bgRect)
  } else {
    bgRect.width(props.width)
    bgRect.height(props.height)
  }

  gridLayer.batchDraw()
}

onMounted(() => {
  if (!containerRef.value) {
    return
  }
  createStage()
  renderGrid()
  // Wait to let component mount
  setTimeout(() => renderDevices(), 0)

  resizeObserver = new ResizeObserver(() => {
    applyStageSize()
  })
  resizeObserver.observe(containerRef.value)
})

watch(
  () => [props.devices, props.selectedId, props.width, props.height, props.zoom, props.overlapIds],
  () => {
    if (stage) {
      stage.scale({ x: props.zoom, y: props.zoom })
      applyStageSize()
      clampStagePosition()
    }
    renderGrid()
    renderDevices()
  }
)

onBeforeUnmount(() => {
  if (resizeObserver && containerRef.value) {
    resizeObserver.unobserve(containerRef.value)
    resizeObserver.disconnect()
    resizeObserver = null
  }
  if (stage) {
    stage.destroy()
    stage = null
    layer = null
    gridLayer = null
  }
})
</script>

<template>
  <div class="canvas">
    <div ref="containerRef" class="canvas__stage"></div>
  </div>
</template>