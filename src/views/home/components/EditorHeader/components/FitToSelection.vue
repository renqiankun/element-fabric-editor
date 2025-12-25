<template>
  <div class="inline-block">
    <el-tooltip :content="$t('editor.header.fitToSelection')">
      <el-button @click="openFitModal" link size="small">
        <el-icon extClass="text-20px" color="#333">
          <AddLocation />
        </el-icon>
      </el-button>
    </el-tooltip>

    <!-- 模态框 -->
    <el-dialog
      v-model="showModal"
      :title="$t('editor.header.fitToSelectionModal.title')"
    >
      <div class="fit-selection-content">
        <p>{{ $t('editor.header.fitToSelectionModal.description') }}</p>
        <div class="input-group">
          <el-input
            v-model="firstId"
            :placeholder="
              $t('editor.header.fitToSelectionModal.firstIdPlaceholder')
            "
            clearable
          />
        </div>
        <div class="input-group">
          <el-input
            v-model="secondId"
            :placeholder="
              $t('editor.header.fitToSelectionModal.secondIdPlaceholder')
            "
            clearable
          />
        </div>
      </div>
      <template #footer>
        <el-button @click="handleClose"> 取消 </el-button>
        <el-button type="primary" @click="handleFitToSelection">
          id适配
        </el-button>
        <el-button type="primary" @click="handleFitToAllObjects">
          自动适配
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { AddLocation } from '@element-plus/icons-vue'
import { useEditorStore } from '@/store/modules/editor'

const editorStore = useEditorStore()

// 模态框状态
const showModal = ref(false)
const firstId = ref('')
const secondId = ref('')

// 打开模态框
const openFitModal = () => {
  showModal.value = true
  firstId.value = ''
  secondId.value = ''
}

// 带动画的缩放和平移动画
const animateToViewport = (targetVpt: number[], duration: number = 300) => {
  const canvas = editorStore.canvas
  if (!canvas) return

  const startTime = Date.now()
  const startVpt = [...canvas.viewportTransform!]

  const animate = () => {
    const elapsed = Date.now() - startTime
    const progress = Math.min(elapsed / duration, 1)

    // 使用缓动函数，这里使用easeOutCubic
    const easeProgress = 1 - Math.pow(1 - progress, 3)

    const currentVpt = startVpt.map(
      (start, idx) => start + (targetVpt[idx] - start) * easeProgress
    )

    canvas.setViewportTransform(currentVpt)
    canvas.requestRenderAll()

    if (progress < 1) {
      requestAnimationFrame(animate)
    }
  }

  animate()
}

// 计算对象边界
const calculateBounds = (objects: fabric.Object[]) => {
  if (!objects || objects.length === 0) return null

  let left = Infinity
  let top = Infinity
  let right = -Infinity
  let bottom = -Infinity

  objects.forEach((obj) => {
    const boundingRect = obj.getBoundingRect(true, true)
    left = Math.min(left, boundingRect.left)
    top = Math.min(top, boundingRect.top)
    right = Math.max(right, boundingRect.left + boundingRect.width)
    bottom = Math.max(bottom, boundingRect.top + boundingRect.height)
  })

  if (left === Infinity) return null

  return {
    left,
    top,
    width: right - left,
    height: bottom - top
  }
}

// 自动适配所有对象
const handleFitToAllObjects = () => {
  const canvas = editorStore.canvas
  if (!canvas) return

  // 获取画布中除workspace外的所有对象
  const objects = canvas.getObjects().filter((obj) => obj.id !== 'workspace')

  if (!objects || objects.length === 0) {
    ElMessage.warning('画布中没有可适配的对象')
    return
  }

  // 计算所有对象的边界
  const bounds = calculateBounds(objects)
  if (!bounds) {
    ElMessage.warning('无法计算对象边界')
    return
  }

  // 获取画布容器尺寸
  const container = document.querySelector('#workspace') as HTMLElement
  if (!container) {
    ElMessage.error('未找到画布容器')
    return
  }

  // 计算缩放比例
  const containerWidth = container.offsetWidth
  const containerHeight = container.offsetHeight

  // 添加一些内边距，避免对象紧贴边缘
  const padding = 20
  const scaleX = (containerWidth - padding * 2) / bounds.width
  const scaleY = (containerHeight - padding * 2) / bounds.height
  const scale = Math.min(scaleX, scaleY) * 0.9 // 0.9是为了确保对象完全显示

  // 计算缩放后的中心点
  const centerPoint = new fabric.Point(
    bounds.left + bounds.width / 2,
    bounds.top + bounds.height / 2
  )

  // 获取当前视口变换矩阵
  const currentVpt = canvas.viewportTransform
  if (!currentVpt) return

  // 计算目标视口变换矩阵
  const targetVpt = [...currentVpt]
  // 设置新的缩放
  targetVpt[0] = scale // scaleX
  targetVpt[3] = scale // scaleY

  // 计算目标位置，使所有对象的中心点位于画布中心
  const viewportCenter = canvas.getCenter()
  const offsetX = viewportCenter.left - centerPoint.x * scale
  const offsetY = viewportCenter.top - centerPoint.y * scale

  targetVpt[4] = offsetX // offsetX
  targetVpt[5] = offsetY // offsetY

  // 执行带动画的变换
  animateToViewport(targetVpt, 500) // 500ms 动画时间

  ElMessage.success('已自动缩放到所有对象')
  showModal.value = false
}

// 处理缩放适配
const handleFitToSelection = () => {
  if (!firstId.value || !secondId.value) {
    ElMessage.warning('请提供两个ID')
    return
  }

  const canvas = editorStore.canvas
  if (!canvas) return

  // 查找两个对象
  const firstObj = canvas.getObjects().find((obj) => obj.id === firstId.value)
  const secondObj = canvas.getObjects().find((obj) => obj.id === secondId.value)

  if (!firstObj || !secondObj) {
    ElMessage.warning('找不到指定ID的对象')
    return
  }

  // 计算两个对象的边界框
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

  // 获取画布容器尺寸
  const container = document.querySelector('#workspace') as HTMLElement
  if (!container) {
    ElMessage.error('未找到画布容器')
    return
  }

  // 计算缩放比例
  const containerWidth = container.offsetWidth
  const containerHeight = container.offsetHeight

  // 添加一些内边距，避免对象紧贴边缘
  const padding = 20
  const scaleX = (containerWidth - padding * 2) / combinedBounds.width
  const scaleY = (containerHeight - padding * 2) / combinedBounds.height
  const scale = Math.min(scaleX, scaleY) * 0.9 // 0.9是为了确保对象完全显示

  // 计算缩放后的中心点
  const scaledWidth = combinedBounds.width * scale
  const scaledHeight = combinedBounds.height * scale
  const centerPoint = new fabric.Point(
    combinedBounds.left + combinedBounds.width / 2,
    combinedBounds.top + combinedBounds.height / 2
  )

  // 获取当前视口变换矩阵
  const currentVpt = canvas.viewportTransform
  if (!currentVpt) return

  // 计算目标视口变换矩阵
  const targetVpt = [...currentVpt]
  // 设置新的缩放
  targetVpt[0] = scale // scaleX
  targetVpt[3] = scale // scaleY

  // 计算目标位置，使两个对象的中心点位于画布中心
  const viewportCenter = canvas.getCenter()
  const offsetX = viewportCenter.left - centerPoint.x * scale
  const offsetY = viewportCenter.top - centerPoint.y * scale

  targetVpt[4] = offsetX // offsetX
  targetVpt[5] = offsetY // offsetY

  // 执行带动画的变换
  animateToViewport(targetVpt, 500) // 500ms 动画时间

  ElMessage.success('已缩放适配到选中区域')
  showModal.value = false
}

// 关闭模态框
const handleClose = () => {
  showModal.value = false
  firstId.value = ''
  secondId.value = ''
}
</script>

<style scoped>
.fit-selection-content {
  padding: 10px 0;
}

.fit-selection-content p {
  margin-bottom: 15px;
  line-height: 1.5;
}

.input-group {
  margin-bottom: 15px;
}

.input-group:last-child {
  margin-bottom: 0;
}

.auto-fit-btn {
  margin-top: 10px;
}
</style>
