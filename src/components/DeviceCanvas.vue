<script setup>
import { onBeforeUnmount, onMounted, ref, watch } from 'vue'
import Konva from 'konva'

const emit = defineEmits(['select', 'zoom', 'fit'])

const props = defineProps({
  devices: {
    type: Array,
    default: () => []
  },
  selectedId: {
    type: String,
    default: null
  },
  minX: {
    type: Number,
    default: 0
  },
  minY: {
    type: Number,
    default: 0
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
  bgSvgUrl: {
    type: String,
    default: null
  },
  bgSvgMeta: {
    type: Object,
    default: null
  },
  showBg: {
    type: Boolean,
    default: false
  }
})

const containerRef = ref(null)
let stage = null
let layer = null
let gridLayer = null
let bgSvgLayer = null
let bgSvgImage = null
let tooltipLayer = null
let tooltipRect = null
let tooltipText = null
let resizeObserver = null
let bgRect = null
let originLayer = null
const gridSpacing = 100
const gridColors = ['#f6f7fb', '#ffffff']
// 搜索聚焦时让设备保留足够的周边环境，避免小设备被放大到占满视口。
const focusViewportRatio = 0.45
const maxFocusZoom = 0.5

/**
 * 将用户坐标系（左下角原点，Y向上）转换为 Konva 屏幕坐标（左上角原点，Y向下）
 * 支持负坐标：整体偏移 minX/minY，使最小坐标落在画布左下角
 * @param {number} userY - 用户坐标的 Y 值（距离底部的距离）
 * @param {number} height - 设备高度
 * @returns {number} Konva 坐标系的 Y 值
 */
const toKonvaY = (userY, height) => (props.height + props.minY) - userY - height

const deviceNodes = new Map() // id -> { group, rect, label }

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

  // 左下角原点坐标系：原点在屏幕左下，Y向上
  // stage.y 表示原点在屏幕上的位置，负值表示在可视区域上方
  const minX = Math.min(0, viewWidth - worldWidth)
  const maxY = 0  // 原点在底部，不能下移
  const minY = Math.min(0, viewHeight - worldHeight)

  const clampedX = Math.min(0, Math.max(minX, stage.x()))
  const clampedY = Math.max(minY, Math.min(maxY, stage.y()))

  if (clampedX !== stage.x() || clampedY !== stage.y()) {
    stage.position({ x: clampedX, y: clampedY })
  }
}

/** 格式化 info 值（处理 LP 坐标等对象类型） */
const _formatInfoVal = (value) => {
  if (value === null || value === undefined) return ''
  if (typeof value === 'object') {
    if ('x' in value && 'y' in value) return `x=${value.x}, y=${value.y}`
    return JSON.stringify(value)
  }
  return String(value)
}

/** 显示设备信息工具提示 */
const _showTooltip = (device, rect) => {
  if (!tooltipRect || !tooltipText || !stage) return

  const info = device.info
  if (!info) return

  // 构建信息文本
  const lines = [device._isDetail ? '📋 Detail Info' : '📋 Macro Info']
  for (const [key, value] of Object.entries(info)) {
    if (value === null || value === undefined || value === '') continue
    let label = key
    if (key === 'area') label = 'Area'
    else if (key === 'bay_occupation') label = 'Bay'
    else if (key === 'direction_flag') label = 'Direction'
    lines.push(`${label}: ${_formatInfoVal(value)}`)
  }
  const text = lines.join('\n')

  tooltipText.text(text)
  const tw = tooltipText.width()
  const th = tooltipText.height()

  // 计算屏幕位置：在矩形上方居中
  const rectAbs = rect.getAbsolutePosition()
  const scale = stage.scaleX()
  const tooltipX = rectAbs.x + (rect.width() * scale - tw) / 2
  const tooltipY = rectAbs.y - th - 8 * scale

  tooltipText.position({ x: tooltipX, y: tooltipY })
  tooltipRect.position({ x: tooltipX, y: tooltipY })
  tooltipRect.size({ width: tw, height: th })

  tooltipRect.visible(true)
  tooltipText.visible(true)
  tooltipLayer.batchDraw()
}

/** 隐藏工具提示 */
const _hideTooltip = () => {
  if (!tooltipRect || !tooltipText) return
  tooltipRect.visible(false)
  tooltipText.visible(false)
  if (tooltipLayer) tooltipLayer.batchDraw()
}

/**
 * 自适应画布：根据设备范围计算最佳缩放和位置
 * 使所有设备可见，且画布原点（左下角）对准设备区域左下角
 */
const autoFit = () => {
  if (!stage || props.devices.length === 0) return

  // 强制浏览器重新计算布局
  if (containerRef.value) {
    void containerRef.value.offsetHeight
  }

  const containerSize = getContainerSize()
  if (containerSize.width === 0 || containerSize.height === 0) {
    console.warn('Container size is zero, skipping autoFit')
    return
  }

  // 计算所有设备在 Konva 空间中的包围盒（已含偏移）
  let kMinX = Infinity, kMinY = Infinity
  let kMaxX = -Infinity, kMaxY = -Infinity
  const offX = props.minX
  const baseY = props.height + props.minY

  for (const d of props.devices) {
    const kx = d.x - offX
    const ky = baseY - d.y - d.height  // rect 左上角 Konva Y
    if (kx < kMinX) kMinX = kx
    if (ky < kMinY) kMinY = ky
    if (kx + d.width > kMaxX) kMaxX = kx + d.width
    if (ky + d.height > kMaxY) kMaxY = ky + d.height
  }

  const padding = 200
  const boundsWidth = kMaxX - kMinX + padding * 2
  const boundsHeight = kMaxY - kMinY + padding * 2

  // 计算最佳缩放
  const fitZoom = Math.min(
    containerSize.width / Math.max(boundsWidth, 1),
    containerSize.height / Math.max(boundsHeight, 1),
    1
  )

  stage.scale({ x: fitZoom, y: fitZoom })

  // 居中显示
  const targetX = (containerSize.width - (kMinX + kMaxX) * fitZoom) / 2
  const targetY = (containerSize.height - (kMinY + kMaxY) * fitZoom) / 2
  stage.position({ x: targetX, y: targetY })

  emit('zoom', fitZoom)
  stage.batchDraw()
}

/** 将指定机台放大并居中到当前可视区域 */
const focusDevice = (deviceId) => {
  if (!stage) return false

  const device = props.devices.find((item) => item.id === deviceId)
  if (!device) return false

  const { width: viewWidth, height: viewHeight } = getContainerSize()
  if (viewWidth <= 0 || viewHeight <= 0) return false

  // 目标设备最多占视口宽高的 45%，同时限制绝对缩放倍率。
  // 相比按固定边距铺满视口，可以在聚焦后继续看到设备周边布局。
  const targetZoom = Math.min(
    (viewWidth * focusViewportRatio) / Math.max(device.width, 1),
    (viewHeight * focusViewportRatio) / Math.max(device.height, 1),
    maxFocusZoom
  )
  const nextZoom = Math.max(targetZoom, 0.001)
  const centerX = device.x - props.minX + device.width / 2
  const centerY = toKonvaY(device.y, device.height) + device.height / 2

  stage.scale({ x: nextZoom, y: nextZoom })
  stage.position({
    x: viewWidth / 2 - centerX * nextZoom,
    y: viewHeight / 2 - centerY * nextZoom
  })
  emit('zoom', nextZoom)
  stage.batchDraw()
  return true
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

  // 背景 SVG 图层（grid 之上，设备之下）
  bgSvgLayer = new Konva.Layer({ listening: false })
  bgSvgLayer.hide()
  stage.add(bgSvgLayer)

  layer = new Konva.Layer()
  stage.add(layer)

  // 工具提示层
  tooltipLayer = new Konva.Layer()
  tooltipRect = new Konva.Rect({
    visible: false,
    fill: '#0f1c3f',
    cornerRadius: 4,
    opacity: 0.9,
    listening: false
  })
  tooltipText = new Konva.Text({
    visible: false,
    fontFamily: 'Arial, sans-serif',
    fontSize: 13,
    fill: '#ffffff',
    padding: 10,
    lineHeight: 1.5,
    listening: false
  })
  tooltipLayer.add(tooltipRect)
  tooltipLayer.add(tooltipText)
  stage.add(tooltipLayer)

  // 原点标记层：DXF (0,0) 十字准线
  originLayer = new Konva.Layer()
  const ox = -props.minX
  const oy = props.height + props.minY
  const crossLen = 5000
  originLayer.add(new Konva.Line({
    points: [ox - crossLen, oy, ox + crossLen, oy],
    stroke: '#ff4444',
    strokeWidth: 4,
    listening: false,
  }))
  originLayer.add(new Konva.Line({
    points: [ox, oy - crossLen, ox, oy + crossLen],
    stroke: '#ff4444',
    strokeWidth: 4,
    listening: false,
  }))
  originLayer.add(new Konva.Circle({
    x: ox, y: oy, radius: 8,
    fill: '#ff4444',
    listening: false,
  }))
  originLayer.add(new Konva.Text({
    x: ox + 12, y: oy + 12,
    text: 'Origin (0,0)',
    fontSize: 14, fill: '#ff4444',
    fontFamily: 'Arial, sans-serif',
    listening: false,
  }))
  stage.add(originLayer)

  // 悬浮坐标显示（HTML 层，固定在画布左上角，不受 stage 缩放影响）
  const coordEl = document.createElement('div')
  coordEl.style.cssText = 'position:absolute;top:8px;left:8px;z-index:99;pointer-events:none;'
    + 'background:rgba(0,0,0,0.7);color:#fff;font:12px monospace;padding:4px 8px;border-radius:3px;'
    + 'white-space:nowrap;'
  containerRef.value.appendChild(coordEl)

  stage.on('mousemove', () => {
    const ptr = stage.getPointerPosition()
    if (!ptr) return
    const scale = stage.scaleX()
    const wx = (ptr.x - stage.x()) / scale + props.minX
    const wy = props.height + props.minY - (ptr.y - stage.y()) / scale
    coordEl.textContent = `DXF: (${wx.toFixed(0)}, ${wy.toFixed(0)})`
  })

  stage.on('click', (event) => {
    if (event.target === stage) {
      emit('select', null)
    }
  })

  stage.on('wheel', (event) => {
    event.evt.preventDefault()
    _hideTooltip()
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

  // 根据所有矩形的平均大小计算统一线宽
  const avgDiag = (() => {
    if (props.devices.length === 0) return 0
    let total = 0
    for (const d of props.devices) {
      total += Math.sqrt(d.width * d.width + d.height * d.height)
    }
    return total / props.devices.length
  })()
  const baseStrokeWidth = Math.max(3, avgDiag * 0.003)

  props.devices.forEach((device) => {
    const isSelected = device.id === props.selectedId
    const fill = isSelected ? '#ffd166' : '#9ec4ff'
    const stroke = isSelected ? '#e64500' : '#000000'
    const strokeWidth = isSelected ? baseStrokeWidth * 2 : baseStrokeWidth

    let cached = deviceNodes.get(device.id)
    if (!cached) {
      const group = new Konva.Group()

      const rect = new Konva.Rect()
      const label = new Konva.Text({
        fontFamily: 'Arial, sans-serif',
        fill: '#0f1c3f',
        align: 'center',
        listening: false
      })

      rect.on('click', () => emit('select', device.id))

      rect.on('mouseenter', () => {
        if (device.info) {
          _showTooltip(device, rect)
        }
      })
      rect.on('mouseleave', () => {
        _hideTooltip()
      })

      group.add(rect)
      group.add(label)
      layer.add(group)
      cached = { group, rect, label }
      deviceNodes.set(device.id, cached)
    }

    // 更新缓存中的 info，供 tooltip 使用
    cached._info = device.info

    cached.group.position({ x: device.x - props.minX, y: toKonvaY(device.y, device.height) })

    cached.rect.setAttrs({
      width: device.width,
      height: device.height,
      fill,
      stroke,
      strokeWidth,
      shadowColor: isSelected ? '#ff5a1f' : undefined,
      shadowBlur: isSelected ? Math.max(12, baseStrokeWidth * 4) : 0,
      shadowOpacity: isSelected ? 0.8 : 0
    })

    const fontSize = Math.max(10, device.width / 9)
    let displayText, lineCount
    if (device._isDetail) {
      // Detail 模式：显示设备编号
      displayText = device.id
      lineCount = 1
    } else {
      // Macro 模式：三行显示 direction_flag / 文字 / bay_occupation
      const info = device.info || {}
      const l1 = info.direction_flag || ''
      const l2 = info.文字 || ''
      const l3 = info.bay_occupation || ''
      displayText = [l1, l2, l3].filter(Boolean).join('\n')
      lineCount = [l1, l2, l3].filter(Boolean).length
    }

    cached.label.setAttrs({
      y: device.height / 2 - (fontSize * lineCount) / 2,
      width: device.width,
      height: fontSize * lineCount,
      fontSize,
      text: displayText
    })
  })

  // 清理已删除设备的 group（防止残留在 layer 中）
  const currentIds = new Set(props.devices.map(d => d.id))
  for (const [id, cached] of deviceNodes.entries()) {
    if (!currentIds.has(id)) {
      if (cached.group.parent) {
        cached.group.destroy()
      }
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

let resizeTimeout = null

const handleWindowResize = () => {
  // 防抖处理
  if (resizeTimeout) {
    clearTimeout(resizeTimeout)
  }
  resizeTimeout = setTimeout(() => {
    applyStageSize()
    if (stage) {
      stage.batchDraw()
    }
  }, 100)
}

onMounted(() => {
  if (!containerRef.value) {
    return
  }
  createStage()
  renderGrid()
  setTimeout(() => renderDevices(), 0)

  // ResizeObserver 只负责更新 stage 尺寸，不自动调用 autoFit
  // autoFit 由父组件在适当时机调用
  resizeObserver = new ResizeObserver(() => {
    applyStageSize()
    if (stage) {
      stage.batchDraw()
    }
  })
  resizeObserver.observe(containerRef.value)

  // 监听窗口 resize 事件
  window.addEventListener('resize', handleWindowResize)
})

// 设备数据变化时自适应画布
watch(
  () => props.devices,
  (newDevices) => {
    if (stage && newDevices && newDevices.length > 0) {
      setTimeout(() => autoFit(), 50)
    }
  },
  { deep: true }
)

defineExpose({ autoFit, focusDevice })

// 背景 SVG 加载/切换
const loadBgSvg = (url, meta) => {
  if (!bgSvgLayer || !url || !meta) {
    console.warn('[bgSvg] 跳过: layer/url/meta 缺失', { layer: !!bgSvgLayer, url, meta })
    return
  }

  const bbox = meta.bbox
  if (!bbox) {
    console.warn('[bgSvg] meta.bbox 缺失', meta)
    return
  }
  const imgW = meta.svgWidth ?? (bbox.maxX - bbox.minX)
  const imgH = meta.svgHeight ?? (bbox.maxY - bbox.minY)
  console.log('[bgSvg] 开始加载, meta 尺寸:', imgW, '×', imgH, 'bbox:', bbox)

  const img = new window.Image()
  img.onerror = (err) => {
    console.error('[bgSvg] SVG 加载失败', err)
  }
  img.onload = () => {
    const natW = img.naturalWidth
    const natH = img.naturalHeight
    console.log('[bgSvg] SVG 加载完成, 原始尺寸:', natW, '×', natH, '期望:', imgW, '×', imgH)

    // 销毁旧图
    if (bgSvgImage) {
      bgSvgImage.destroy()
      bgSvgImage = null
    }

    const konvaX = bbox.minX - props.minX
    const konvaY = (props.height + props.minY) - bbox.maxY
    console.log('[bgSvg] Konva 放置位置: x=', konvaX, 'y=', konvaY,
      'canvas props: minX=', props.minX, 'minY=', props.minY,
      'width=', props.width, 'height=', props.height)

    bgSvgImage = new Konva.Image({
      image: img,
      x: konvaX,
      y: konvaY,
      width: imgW,
      height: imgH,
      opacity: 0.3,
      listening: false,
    })
    bgSvgLayer.add(bgSvgImage)
    bgSvgLayer.visible(props.showBg)
    bgSvgLayer.batchDraw()
    console.log('[bgSvg] Konva.Image 已添加到图层, visible=', props.showBg)
  }
  img.src = url
}

watch(() => [props.bgSvgUrl, props.bgSvgMeta], ([url, meta]) => {
  loadBgSvg(url, meta)
})

watch(() => props.showBg, (show) => {
  if (bgSvgLayer) {
    bgSvgLayer.visible(show && !!bgSvgImage)
    if (stage) stage.batchDraw()
  }
})

// zoom 变化时只更新缩放
watch(() => props.zoom, (newZoom) => {
  if (stage) {
    stage.scale({ x: newZoom, y: newZoom })
    clampStagePosition()
  }
})

// P2: 只在设备数据、选中状态、重叠状态变化时重绘
watch(
  () => [props.devices, props.selectedId, props.width, props.height],
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
  if (resizeTimeout) {
    clearTimeout(resizeTimeout)
    resizeTimeout = null
  }
  window.removeEventListener('resize', handleWindowResize)

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
