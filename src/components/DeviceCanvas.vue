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
let zoomDebounceTimer = null

const gridSpacing = 100
const gridColors = ['#f6f7fb', '#ffffff']

const deviceNodes = new Map()

// 视口裁剪余量（设备宽高的余量比例）
const VIEWPORT_MARGIN_RATIO = 0.3

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

/**
 * P1 视口裁剪：只返回可视区域内的设备
 */
const getVisibleDevices = () => {
  if (!stage) return props.devices

  const scale = stage.scaleX()
  const stageX = stage.x()
  const stageY = stage.y()
  const stageW = stage.width()
  const stageH = stage.height()

  // 计算可视区域在世界坐标系中的范围
  const viewLeft = -stageX / scale
  const viewTop = -stageY / scale
  const viewRight = (-stageX + stageW) / scale
  const viewBottom = (-stageY + stageH) / scale

  // 增加余量，避免滚动时闪烁
  const margin = 300 / scale

  const result = []
  for (let i = 0; i < props.devices.length; i++) {
    const d = props.devices[i]
    if (
      d.x + d.width > viewLeft - margin &&
      d.x < viewRight + margin &&
      d.y + d.height > viewTop - margin &&
      d.y < viewBottom + margin
    ) {
      result.push(d)
    }
  }
  return result
}

const renderDevices = () => {
  if (!layer) {
    return
  }

  const overlapSet = new Set(props.overlapIds)

  // P1: 使用视口裁剪
  const devicesToRender = getVisibleDevices()
  const visibleIds = new Set(devicesToRender.map(d => d.id))

  devicesToRender.forEach((device) => {
    const isSelected = device.id === props.selectedId
    const isOverlap = overlapSet.has(device.id)
    const fill = isOverlap ? '#ffb3b3' : isSelected ? '#ffd4a5' : '#9ec4ff'
    const stroke = isOverlap ? '#c81e1e' : isSelected ? '#e46c0a' : '#2a4b8d'
    const strokeWidth = isSelected ? 3 : 2

    let group = deviceNodes.get(device.id)
    if (!group) {
      group = new Konva.Group()

      // P0: 移除 shadow，改用双层矩形模拟立体感
      const outerRect = new Konva.Rect({
        name: 'outerRect',
        cornerRadius: 6,
        fill: 'rgba(0, 0, 0, 0.1)',
        offset: { x: 0, y: 0 }
      })

      const rect = new Konva.Rect({
        name: 'rect',
        cornerRadius: 6,
        // 不再使用 shadow，通过渐变和描边实现立体感
      })

      const label = new Konva.Text({
        name: 'label',
        fontSize: 16,
        fontFamily: 'Space Grotesk, system-ui, sans-serif',
        fontStyle: '600',
        fill: '#0f1c3f',
        align: 'center',
        listening: false  // P5: 禁用文本节点的 hit detection
      })

      // 点击事件只绑定在外层矩形上
      outerRect.on('click', () => emit('select', device.id))

      group.add(outerRect)
      group.add(rect)
      group.add(label)
      layer.add(group)
      deviceNodes.set(device.id, group)
    }

    group.position({ x: device.x, y: device.y })

    const outerRect = group.findOne('.outerRect')
    const rect = group.findOne('.rect')
    const label = group.findOne('.label')

    // 外层阴影矩形：偏移 2px
    outerRect.setAttrs({
      x: 2,
      y: 2,
      width: device.width,
      height: device.height,
      fill: 'rgba(0, 0, 0, 0.1)',
    })

    // 主矩形
    rect.setAttrs({
      x: 0,
      y: 0,
      width: device.width,
      height: device.height,
      fill,
      stroke,
      strokeWidth
    })

    label.setAttrs({
      y: device.height / 2 - 10,
      width: device.width,
      text: device.id
    })
  })

  // 清理不在可视范围内的设备节点（从 stage 中移除但保留缓存引用）
  for (const [id, group] of deviceNodes.entries()) {
    if (!visibleIds.has(id)) {
      // 从 layer 移除但保留在 deviceNodes 中，以便复用
      if (group.parent) {
        group.remove()
      }
    } else {
      // 确保可视范围内的设备在 layer 中
      if (!group.parent) {
        layer.add(group)
      }
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

// P2: 将 zoom 变化分离出来，避免触发全量重绘
watch(() => props.zoom, (newZoom) => {
  if (stage) {
    stage.scale({ x: newZoom, y: newZoom })
    clampStagePosition()
    // 不调用 renderDevices()，只重新计算视口内可见设备
    renderDevices()
  }
})

// P2: 只在设备数据、选中状态、重叠状态变化时重绘
watch(
  () => [props.devices, props.selectedId, props.width, props.height, props.overlapIds],
  () => {
    if (stage) {
      applyStageSize()
      clampStagePosition()
    }
    renderGrid()
    renderDevices()
  }
)

onBeforeUnmount(() => {
  // 清理 zoom debounce timer
  if (zoomDebounceTimer) {
    clearTimeout(zoomDebounceTimer)
    zoomDebounceTimer = null
  }

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
