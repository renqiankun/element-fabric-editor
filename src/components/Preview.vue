<template>
  <div class="preview-container" ref="previewContainerRef">
    <canvas ref="previewCanvasRef" class="preview-canvas"></canvas>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, nextTick, watch, computed } from 'vue'
import { useEditorStore } from '@/store/modules/editor'
import { fabric } from 'fabric'

const props = defineProps({
  visible: {
    type: Boolean,
    default: false
  },
  width: {
    type: Number,
    default: 300
  },
  height: {
    type: Number,
    default: 200
  },
  padding: {
    type: Number,
    default: 20
  }
})

const emit = defineEmits<{
  click: [target: fabric.Object, isInsideShape: boolean]
}>()

const editorStore = useEditorStore()
const previewContainerRef = ref<HTMLDivElement>()
const previewCanvasRef = ref<HTMLCanvasElement>()
const previewCanvas = ref<fabric.Canvas>()

// 初始化预览canvas
const initPreviewCanvas = async () => {
  if (!previewCanvasRef.value) return

  // 创建预览canvas实例
  previewCanvas.value = new fabric.Canvas(previewCanvasRef.value, {
    width: props.width,
    height: props.height,
    selection: false,
    hoverCursor: 'grab',
    preserveObjectStacking: true,
    // 启用画布拖拽和缩放
    allowTouchScrolling: true,
    // 禁用选择框
    selection: false,
    // 禁用组选
    selectionKey: null,
    // 启用逐像素目标查找
    perPixelTargetFind: true
  })

  // 设置画布可拖拽
  previewCanvas.value.defaultCursor = 'grab'
  previewCanvas.value.moveCursor = 'grabbing'

  // 绑定拖拽和缩放事件
  bindPreviewEvents()
}

// 绑定预览canvas事件
const bindPreviewEvents = () => {
  if (!previewCanvas.value) return

  let startX = 0
  let startY = 0
  let startTime = 0
  let isDragging = false
  const dragThreshold = 5 // 拖拽阈值

  // 鼠标按下事件
  previewCanvas.value.on('mouse:down', (opt) => {
    // 记录按下时的位置和时间
    startX = opt.e.clientX
    startY = opt.e.clientY
    startTime = new Date().getTime()
    isDragging = false

    previewCanvas.value!.defaultCursor = 'grabbing'
  })

  // 鼠标移动事件
  previewCanvas.value.on('mouse:move', (opt) => {
    if (opt.e.buttons === 1) {
      // 左键按下
      const deltaX = Math.abs(opt.e.clientX - startX)
      const deltaY = Math.abs(opt.e.clientY - startY)
      const distance = Math.sqrt(deltaX * deltaX + deltaY * deltaY)

      // 如果移动距离超过阈值，则认为是拖拽
      if (distance > dragThreshold && !isDragging) {
        isDragging = true
      }

      if (isDragging) {
        // 拖拽模式
        const vpt = previewCanvas.value!.viewportTransform
        if (!vpt) return
        vpt[4] += opt.e.clientX - startX
        vpt[5] += opt.e.clientY - startY
        startX = opt.e.clientX
        startY = opt.e.clientY
        previewCanvas.value!.requestRenderAll()
      }
    }
  })

  // 鼠标抬起事件
  previewCanvas.value.on('mouse:up', (opt) => {
    const endTime = new Date().getTime()
    const tapDuration = endTime - startTime

    // 只有在非拖拽状态且点击时间较短时才触发点击
    if (!isDragging && tapDuration < 300) {
      // 检查是否有目标对象被点击
      const target = previewCanvas.value!.findTarget(opt)
      if (target) {
        // 检查是否点击在对象的实际区域内（对于不规则图形）
        const pointer = previewCanvas.value!.getPointer(opt.e)
        const isInsideShape = target.containsPoint(pointer)
        emit('click', target, isInsideShape)
      }
    }

    // 重置状态
    isDragging = false
    previewCanvas.value!.defaultCursor = 'grab'

    // 确保在拖拽后更新对象坐标
    previewCanvas.value?.getObjects().forEach((obj) => {
      obj.setCoords()
    })
  })

  // 鼠标移出画布时重置光标
  previewCanvas.value.on('mouse:out', () => {
    previewCanvas.value!.defaultCursor = 'grab'
    isDragging = false
  })

  // 鼠标滚轮缩放
  previewCanvas.value.on('mouse:wheel', (opt) => {
    const delta = opt.e.deltaY
    let zoom = previewCanvas.value!.getZoom()
    zoom *= 0.999 ** delta
    if (zoom > 20) zoom = 20
    if (zoom < 0.01) zoom = 0.01
    
    // 使用Fabric.js内置的zoomToPoint方法，以鼠标位置为中心进行缩放
    const deltaPoint = new fabric.Point(opt.e.offsetX, opt.e.offsetY);
    previewCanvas.value!.zoomToPoint(deltaPoint, zoom)
    
    opt.e.preventDefault()
    opt.e.stopPropagation()
  })

  // 重要：在视口变换后更新对象坐标
  previewCanvas.value.on('after:render', () => {
    previewCanvas.value?.getObjects().forEach((obj) => {
      obj.setCoords()
    })
  })
}

// 设置缩放
const setZoom = (zoom: number, point?: fabric.Point) => {
  if (!previewCanvas.value) return
  const canvas = previewCanvas.value
  
  const vpt = canvas.viewportTransform
  if (!vpt) return
  
  // 获取画布中心作为默认缩放中心
  const center = point ? new fabric.Point(point.x, point.y) : canvas.getCenter()
  
  // 计算缩放因子
  const currentZoom = vpt[0]  // 当前缩放比例
  const zoomFactor = zoom / currentZoom
  
  // 计算新的偏移量，确保缩放中心点位置不变
  // 公式: newOffset = center - (center - currentOffset) * zoomFactor
  const newOffsetX = center.x - (center.x - vpt[4]) * zoomFactor
  const newOffsetY = center.y - (center.y - vpt[5]) * zoomFactor
  
  // 正确设置视口变换矩阵的所有参数
  vpt[0] = zoom  // scaleX
  vpt[1] = 0     // skewX (保持为0)
  vpt[2] = 0     // skewY (保持为0)
  vpt[3] = zoom  // scaleY
  vpt[4] = newOffsetX  // offsetX
  vpt[5] = newOffsetY  // offsetY
  
  canvas.setViewportTransform(vpt)
  canvas.requestRenderAll()
}

// 更新预览内容
const updatePreview = () => {
  if (!previewCanvas.value || !editorStore.canvas) return

  // 清空预览canvas
  previewCanvas.value.clear()

  // 获取原始canvas对象
  const originalObjects = editorStore.canvas.getObjects()
  const workspace = originalObjects.find((obj) => obj.id === 'workspace')

  // 如果有workspace，复制其背景色
  if (workspace) {
    const ws = workspace as fabric.Rect
    previewCanvas.value.setBackgroundColor(
      ws.fill || '#ffffff',
      previewCanvas.value.renderAll.bind(previewCanvas.value)
    )
  }

  // 复制对象到预览canvas
  originalObjects.forEach((obj) => {
    if (obj.id !== 'workspace') {
      // 不复制workspace本身，但保留其背景色
      obj.clone((clonedObj: fabric.Object) => {
        // 调整对象位置以适应预览canvas尺寸
        const scaleX = props.width / (editorStore.canvas?.width || 1)
        const scaleY = props.height / (editorStore.canvas?.height || 1)
        const scale = Math.min(scaleX, scaleY) * 0.9 // 留一些边距

        clonedObj.set({
          left: (clonedObj.left || 0) * scale,
          top: (clonedObj.top || 0) * scale,
          scaleX: (clonedObj.scaleX || 1) * scale,
          scaleY: (clonedObj.scaleY || 1) * scale,
          // 禁用对象选择，但保持事件响应
          selectable: false,
          evented: true
        })

        // 确保对象在预览canvas范围内
        clonedObj.setCoords()
        previewCanvas.value?.add(clonedObj)
      })
    }
  })

  previewCanvas.value.renderAll()
}

// 自适应缩放适配到两个ID的区域
const fitToSelection = (firstId: string, secondId: string) => {
  if (!previewCanvas.value || !editorStore.canvas) return

  // 查找原始画布中的两个对象
  const firstObj = editorStore.canvas
    .getObjects()
    .find((obj) => obj.id === firstId)
  const secondObj = editorStore.canvas
    .getObjects()
    .find((obj) => obj.id === secondId)

  if (!firstObj || !secondObj) {
    console.warn('找不到指定ID的对象')
    return
  }

  // 计算两个对象在原始画布中的边界框
  const firstBounds = firstObj.getBoundingRect(true, true)
  const secondBounds = secondObj.getBoundingRect(true, true)

  // 计算两个对象的合并边界框
  const combinedBounds = {
    left: Math.min(firstBounds.left, secondBounds.left),
    top: Math.min(firstBounds.top, secondBounds.top),
    width:
      Math.max(
        firstBounds.left + firstBounds.width,
        secondBounds.left + secondBounds.width
      ) - Math.min(firstBounds.left, secondBounds.left),
    height:
      Math.max(
        firstBounds.top + firstBounds.height,
        secondBounds.top + secondBounds.height
      ) - Math.min(firstBounds.top, secondBounds.top)
  }

  // 将原始画布的边界框转换为预览画布的坐标
  const previewScaleX = props.width / (editorStore.canvas?.width || 1)
  const previewScaleY = props.height / (editorStore.canvas?.height || 1)
  const previewScale = Math.min(previewScaleX, previewScaleY) * 0.9 // 与updatePreview中相同的缩放比例

  const previewBounds = {
    left: combinedBounds.left * previewScale,
    top: combinedBounds.top * previewScale,
    width: combinedBounds.width * previewScale,
    height: combinedBounds.height * previewScale
  }

  // 计算缩放比例
  const containerWidth = props.width - props.padding * 2
  const containerHeight = props.height - props.padding * 2
  const scaleX = containerWidth / previewBounds.width
  const scaleY = containerHeight / previewBounds.height
  const scale = Math.min(scaleX, scaleY)

  // 计算缩放后的中心点
  const centerPoint = new fabric.Point(
    previewBounds.left + previewBounds.width / 2,
    previewBounds.top + previewBounds.height / 2
  )

  // 获取当前视口变换矩阵
  const currentVpt = previewCanvas.value.viewportTransform
  if (!currentVpt) return

  // 计算目标视口变换矩阵
  const targetVpt = [...currentVpt]
  // 设置新的缩放
  targetVpt[0] = scale // scaleX
  targetVpt[3] = scale // scaleY

  // 计算目标位置，使两个对象的中心点位于画布中心
  const viewportCenter = previewCanvas.value.getCenter()
  const offsetX = viewportCenter.left - centerPoint.x * scale
  const offsetY = viewportCenter.top - centerPoint.y * scale

  targetVpt[4] = offsetX // offsetX
  targetVpt[5] = offsetY // offsetY

  // 执行带动画的变换
  animateToViewport(targetVpt, 500) // 500ms 动画时间
}

// 自动适配所有对象
const fitToAll = () => {
  if (!previewCanvas.value || !editorStore.canvas) return

  // 获取原始画布中除workspace外的所有对象
  const objects = editorStore.canvas
    .getObjects()
    .filter((obj) => obj.id !== 'workspace')

  if (!objects || objects.length === 0) {
    console.warn('画布中没有可适配的对象')
    return
  }

  // 计算所有对象在原始画布中的边界
  const bounds = calculateOriginalBounds(objects)
  if (!bounds) {
    console.warn('无法计算对象边界')
    return
  }

  // 将原始画布的边界转换为预览画布的坐标
  const previewScaleX = props.width / (editorStore.canvas?.width || 1)
  const previewScaleY = props.height / (editorStore.canvas?.height || 1)
  const previewScale = Math.min(previewScaleX, previewScaleY) * 0.9 // 与updatePreview中相同的缩放比例

  const previewBounds = {
    left: bounds.left * previewScale,
    top: bounds.top * previewScale,
    width: bounds.width * previewScale,
    height: bounds.height * previewScale
  }

  // 计算缩放比例
  const containerWidth = props.width - props.padding * 2
  const containerHeight = props.height - props.padding * 2
  const scaleX = containerWidth / previewBounds.width
  const scaleY = containerHeight / previewBounds.height
  const scale = Math.min(scaleX, scaleY)

  // 计算缩放后的中心点
  const centerPoint = new fabric.Point(
    previewBounds.left + previewBounds.width / 2,
    previewBounds.top + previewBounds.height / 2
  )

  // 获取当前视口变换矩阵
  const currentVpt = previewCanvas.value.viewportTransform
  if (!currentVpt) return

  // 计算目标视口变换矩阵
  const targetVpt = [...currentVpt]
  // 设置新的缩放
  targetVpt[0] = scale // scaleX
  targetVpt[3] = scale // scaleY

  // 计算目标位置，使所有对象的中心点位于画布中心
  const viewportCenter = previewCanvas.value.getCenter()
  const offsetX = viewportCenter.left - centerPoint.x * scale
  const offsetY = viewportCenter.top - centerPoint.y * scale

  targetVpt[4] = offsetX // offsetX
  targetVpt[5] = offsetY // offsetY

  // 执行带动画的变换
  animateToViewport(targetVpt, 500) // 500ms 动画时间
}

// 计算原始画布中多个对象的边界
const calculateOriginalBounds = (objects: fabric.Object[]) => {
  if (!objects || objects.length === 0) return null

  let left = Infinity
  let top = Infinity
  let right = -Infinity
  let bottom = -Infinity

  objects.forEach((obj) => {
    const bounds = obj.getBoundingRect(true, true)
    left = Math.min(left, bounds.left)
    top = Math.min(top, bounds.top)
    right = Math.max(right, bounds.left + bounds.width)
    bottom = Math.max(bottom, bounds.top + bounds.height)
  })

  if (left === Infinity) return null

  return {
    left,
    top,
    width: right - left,
    height: bottom - top
  }
}

// 带动画的视口变换
const animateToViewport = (targetVpt: number[], duration: number) => {
  if (!previewCanvas.value) return

  const startVpt = [...previewCanvas.value.viewportTransform]
  const startTime = Date.now()

  const animate = () => {
    const now = Date.now()
    const progress = Math.min(1, (now - startTime) / duration)

    // 使用缓动函数
    const easeProgress = easeOutCubic(progress)

    const currentVpt = startVpt.map(
      (start, i) => start + (targetVpt[i] - start) * easeProgress
    ) as fabric.TMat2D

    previewCanvas.value?.setViewportTransform(currentVpt)
    previewCanvas.value?.requestRenderAll()

    if (progress < 1) {
      requestAnimationFrame(animate)
    }
  }

  animate()
}

// 缓动函数
const easeOutCubic = (t: number) => --t * t * t + 1

// 处理画布尺寸变化时的自适应缩放
const handleResize = () => {
  if (props.visible) {
    nextTick(() => {
      fitToAll()
    })
  }
}

// 监听画布变化
watch(
  () => props.visible,
  (visible) => {
    if (visible && editorStore.canvas) {
      nextTick(() => {
        updatePreview()
        fitToAll()
      })
    }
  },
  { immediate: true }
)

// 暴露方法供外部调用
defineExpose({
  fitToSelection,
  fitToAll
})

// 监听画布对象变化
onMounted(() => {
  initPreviewCanvas()

  if (editorStore.canvas) {
    editorStore.canvas.on('object:added', updatePreview)
    editorStore.canvas.on('object:modified', updatePreview)
    editorStore.canvas.on('object:removed', updatePreview)
    editorStore.canvas.on('canvas:cleared', updatePreview)

    // 监听画布尺寸变化
    window.addEventListener('resize', handleResize)
  }
})

// 组件卸载时清理事件
onUnmounted(() => {
  if (editorStore.canvas) {
    editorStore.canvas.off('object:added', updatePreview)
    editorStore.canvas.off('object:modified', updatePreview)
    editorStore.canvas.off('object:removed', updatePreview)
    editorStore.canvas.off('canvas:cleared', updatePreview)

    // 移除事件监听
    window.removeEventListener('resize', handleResize)
  }

  previewCanvas.value?.dispose()
})
</script>

<style scoped>
.preview-container {
  position: relative;
  border: 1px solid #e0e0e0;
  background: #fff;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  overflow: hidden;
  border-radius: 4px;
  cursor: grab;
}

.preview-canvas {
  display: block;
}
</style>
