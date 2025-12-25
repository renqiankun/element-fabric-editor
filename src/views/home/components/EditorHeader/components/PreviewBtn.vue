<template>
  <div class="inline-block">
    <el-tooltip :content="$t('editor.header.previewBtn')">
      <el-button @click="openPreview" link size="small">
        <el-icon extClass="text-20px" color="#333">
          <View />
        </el-icon>
      </el-button>
    </el-tooltip>

    <!-- 预览模态框 -->
    <el-dialog
      v-model="showPreview"
      :title="$t('editor.header.previewModal.title')"
      width="50%"
    >
      <div class="preview-wrapper">
        <Preview
          ref="previewRef"
          :visible="showPreview"
          :width="previewWidth"
          :height="previewHeight"
          :padding="20"
          @click="handlePreviewClick"
        />

        <div class="preview-controls">
          <div class="size-controls">
            <el-form :model="sizeForm" label-width="auto" class="size-form">
              <el-form-item :label="$t('editor.header.previewModal.width')">
                <el-input-number
                  v-model="sizeForm.width"
                  :min="200"
                  :max="800"
                  :step="50"
                />
              </el-form-item>
              <el-form-item :label="$t('editor.header.previewModal.height')">
                <el-input-number
                  v-model="sizeForm.height"
                  :min="150"
                  :max="600"
                  :step="50"
                />
              </el-form-item>
            </el-form>
          </div>

          <div class="action-controls mt-10px">
            <el-button @click="fitToAll" type="primary" plain>
              {{ $t('editor.header.previewModal.fitToAll') }}
            </el-button>
            <el-button
              @click="openFitToSelection"
              type="primary"
              plain
              class="ml-10px"
            >
              {{ $t('editor.header.previewModal.fitToSelection') }}
            </el-button>
          </div>
        </div>
      </div>
    </el-dialog>

    <!-- 适配指定区域的模态框 -->
    <el-dialog
      v-model="showFitModal"
      :title="$t('editor.header.fitToSelectionModal.title')"
      width="30%"
      :show-close="true"
      @close="closeFitModal"
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
        <el-button @click="closeFitModal">取消</el-button>
        <el-button type="primary" @click="handleFitToSelection">
          确认
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { View } from '@element-plus/icons-vue'
import { useEditorStore } from '@/store/modules/editor'
import Preview from '@/components/Preview.vue'

const editorStore = useEditorStore()

// 模态框状态
const showPreview = ref(false)
const previewRef = ref<InstanceType<typeof Preview>>()
const previewWidth = ref(400)
const previewHeight = ref(300)

// 尺寸表单
const sizeForm = reactive({
  width: 400,
  height: 300
})

// 适配指定区域的模态框
const showFitModal = ref(false)
const firstId = ref('')
const secondId = ref('')

// 监听尺寸变化
watch(
  () => sizeForm.width,
  (val) => {
    previewWidth.value = val
  }
)

watch(
  () => sizeForm.height,
  (val) => {
    previewHeight.value = val
  }
)

// 打开预览
const openPreview = () => {
  showPreview.value = true
  sizeForm.width = 400
  sizeForm.height = 300
}

// 处理预览点击
const handlePreviewClick = (target: fabric.Object | null, isInsideShape: boolean = true) => {
  if (!target) return

  console.log('点击的对象:', target)
  console.log('对象属性:', {
    type: target.type,
    left: target.left,
    top: target.top,
    width: target.width,
    height: target.height,
    scaleX: target.scaleX,
    scaleY: target.scaleY,
    id: target.id || 'no ID'
  })
  
  console.log('是否点击在形状内部:', isInsideShape)

  // 如果点击在不规则图形内部，显示提示
  if (isInsideShape && ['path', 'polygon', 'polyline', 'circle', 'ellipse'].includes(target.type || '')) {
    ElMessage.info(`点击了${target.type}对象内部: ${target.id || 'no ID'}`)
  } else {
    ElMessage.info(`点击了对象: ${target.type || 'unknown'} (ID: ${target.id || 'no ID'})`)
  }
  
  // 这里可以添加更多的交互逻辑
}

// 关闭预览
const handleClose = () => {
  showPreview.value = false
}

// 适配所有对象
const fitToAll = () => {
  if (previewRef.value) {
    previewRef.value.fitToAll()
  }
}

// 打开适配指定区域模态框
const openFitToSelection = () => {
  showFitModal.value = true
  firstId.value = ''
  secondId.value = ''
}

// 适配指定区域
const handleFitToSelection = () => {
  if (!firstId.value || !secondId.value) {
    ElMessage.warning('请提供两个ID')
    return
  }

  if (previewRef.value) {
    previewRef.value.fitToSelection(firstId.value, secondId.value)
    showFitModal.value = false
  }
}

// 关闭适配模态框
const closeFitModal = () => {
  showFitModal.value = false
  firstId.value = ''
  secondId.value = ''
}
</script>

<style scoped>
.preview-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.preview-controls {
  width: 100%;
  padding: 15px;
  background: #f5f5f5;
  border-radius: 4px;
}

.size-controls {
  display: flex;
  gap: 20px;
  align-items: center;
}

.size-form {
  display: flex;
  gap: 15px;
}

.size-form :deep(.el-form-item) {
  margin-bottom: 0;
}

.action-controls {
  display: flex;
  justify-content: center;
}
</style>
