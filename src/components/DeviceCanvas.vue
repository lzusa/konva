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
let gridRaf = 0

const gridSpacing = 100
const gridColors = ['#f6f7fb', '#ffffff']
const gridOverscan = 2

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
    requestGridRender()
    stage.batchDraw()
  })

  stage.on('dragmove', () => {
    clampStagePosition()
    requestGridRender()
  })

  stage.on('dragend', () => {
    clampStagePosition()
    requestGridRender()
  })
}

const renderDevices = () => {
  if (!layer) {
    return
  }

  layer.destroyChildren()
  const overlapSet = new Set(props.overlapIds)

  props.devices.forEach((device) => {
    const isSelected = device.id === props.selectedId
    const isOverlap = overlapSet.has(device.id)
    const fill = isOverlap ? '#ffb3b3' : isSelected ? '#ffd4a5' : '#9ec4ff'
    const stroke = isOverlap ? '#c81e1e' : isSelected ? '#e46c0a' : '#2a4b8d'

    const rect = new Konva.Rect({
      x: device.x,
      y: device.y,
      width: device.width,
      height: device.height,
      fill,
      stroke,
      strokeWidth: isSelected ? 3 : 2,
      cornerRadius: 6,
      shadowColor: 'rgba(0, 0, 0, 0.15)',
      shadowBlur: 8,
      shadowOffset: { x: 2, y: 2 }
    })

    const label = new Konva.Text({
      x: device.x,
      y: device.y + device.height / 2 - 10,
      width: device.width,
      text: device.id,
      fontSize: 16,
      fontFamily: 'Space Grotesk, system-ui, sans-serif',
      fontStyle: '600',
      fill: '#0f1c3f',
      align: 'center'
    })

    rect.on('click', () => emit('select', device.id))
    label.on('click', () => emit('select', device.id))

    layer.add(rect)
    layer.add(label)
  })

  layer.draw()
}

const renderGrid = () => {
  if (!gridLayer) {
    return
  }

  gridLayer.destroyChildren()

  const scale = stage ? stage.scaleX() : 1
  const stagePos = stage ? stage.position() : { x: 0, y: 0 }
  const { width, height } = getContainerSize()

  const viewLeft = (-stagePos.x / scale) - gridSpacing * gridOverscan
  const viewTop = (-stagePos.y / scale) - gridSpacing * gridOverscan
  const viewRight = viewLeft + width / scale + gridSpacing * gridOverscan * 2
  const viewBottom = viewTop + height / scale + gridSpacing * gridOverscan * 2

  const colStart = Math.max(0, Math.floor(viewLeft / gridSpacing))
  const rowStart = Math.max(0, Math.floor(viewTop / gridSpacing))
  const colEnd = Math.min(
    Math.ceil(props.width / gridSpacing),
    Math.ceil(viewRight / gridSpacing)
  )
  const rowEnd = Math.min(
    Math.ceil(props.height / gridSpacing),
    Math.ceil(viewBottom / gridSpacing)
  )

  for (let row = rowStart; row < rowEnd; row += 1) {
    for (let col = colStart; col < colEnd; col += 1) {
      const fill = gridColors[(row + col) % 2]
      gridLayer.add(
        new Konva.Rect({
          x: col * gridSpacing,
          y: row * gridSpacing,
          width: gridSpacing,
          height: gridSpacing,
          fill
        })
      )
    }
  }

  gridLayer.batchDraw()
}

const requestGridRender = () => {
  if (gridRaf) {
    return
  }
  gridRaf = window.requestAnimationFrame(() => {
    gridRaf = 0
    renderGrid()
  })
}

onMounted(() => {
  if (!containerRef.value) {
    return
  }
  createStage()
  renderGrid()
  renderDevices()

  resizeObserver = new ResizeObserver(() => {
    applyStageSize()
    requestGridRender()
  })
  resizeObserver.observe(containerRef.value)
})

watch(
  () => [props.devices, props.selectedId, props.width, props.height, props.zoom],
  () => {
    if (stage) {
      stage.scale({ x: props.zoom, y: props.zoom })
      applyStageSize()
      clampStagePosition()
    }
    requestGridRender()
    renderDevices()
  },
  { deep: true }
)

onBeforeUnmount(() => {
  if (resizeObserver && containerRef.value) {
    resizeObserver.unobserve(containerRef.value)
    resizeObserver.disconnect()
    resizeObserver = null
  }
  if (gridRaf) {
    window.cancelAnimationFrame(gridRaf)
    gridRaf = 0
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