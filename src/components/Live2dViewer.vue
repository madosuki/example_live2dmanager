<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import { Live2dManager, Live2dModel, Live2dViewer } from 'live2dmanager'

type ViewerStatus = 'idle' | 'loading' | 'ready' | 'error'
type CubismRuntimeModel = {
  getCanvasWidth: () => number
}
type CubismRuntimeModelMatrix = {
  setWidth: (width: number) => void
}
type DrawableLive2dModel = Live2dModel & {
  getModel: () => CubismRuntimeModel | null
  getModelMatrix: () => CubismRuntimeModelMatrix
}

declare global {
  interface Window {
    Live2DCubismCore?: unknown
  }
}

const canvasRef = ref<HTMLCanvasElement | null>(null)
const status = ref<ViewerStatus>('idle')
const message = ref('Live2D Cubism Coreをpublic/、モデルディレクトリを/public/live2d_assetsディレクトリに配置してLoad modelを押してください。このサンプルでは公式サンプルに同梱されているMaoを想定しています。')
const live2dAssetsDir = '/live2d_assets/'
const modelName = ref('Mao/')
const modelJsonFileName = ref('Mao.model3.json')
const preloadMotions = ref(false)
const motionFiles = ref<string[]>([])
const expressionIds = ref<string[]>([])
const modelReady = ref(false)

let viewer: Live2dViewer | null = null
let manager: Live2dManager | null = null
let currentModel: Live2dModel | null = null
let animationFrameId = 0

const isBusy = computed(() => status.value === 'loading')
const canUseModel = computed(() => status.value === 'ready' && modelReady.value)

const readLive2dAsset = async (filePath: string): Promise<ArrayBuffer> => {
  const response = await fetch(filePath, { cache: 'no-store' })

  if (!response.ok) {
    throw new Error(`${filePath} を読み込めませんでした (${response.status})`)
  }

  return response.arrayBuffer()
}

const loadCubismCore = async (): Promise<void> => {
  if (window.Live2DCubismCore != null) {
    return
  }

  await new Promise<void>((resolve, reject) => {
    const script = document.createElement('script')
    script.src = '/live2dcubismcore.min.js'
    script.async = true
    script.onload = () => resolve()
    script.onerror = () => reject(new Error('/live2dcubismcore.min.js が見つかりません。'))
    document.head.append(script)
  })

  if (window.Live2DCubismCore == null) {
    throw new Error('Live2D Cubism Coreの読み込み後にLive2DCubismCoreが見つかりません。')
  }
}

const fitCanvasToDisplay = () => {
  if (canvasRef.value == null || viewer == null) {
    return
  }

  const rect = canvasRef.value.getBoundingClientRect()
  const pixelRatio = Math.min(window.devicePixelRatio || 1, 2)
  const width = Math.max(1, Math.round(rect.width * pixelRatio))
  const height = Math.max(1, Math.round(rect.height * pixelRatio))

  if (canvasRef.value.width !== width || canvasRef.value.height !== height) {
    viewer.resize(width, height)
    manager?.setOffScreenSize(width, height)
  }
}

const stopLoop = () => {
  if (animationFrameId !== 0) {
    cancelAnimationFrame(animationFrameId)
    animationFrameId = 0
  }
}

const drawFrame = () => {
  if (viewer == null || currentModel == null || viewer.gl == null) {
    return
  }

  fitCanvasToDisplay()

  const gl = viewer.gl
  viewer.updateTime()

  gl.clearColor(0.0, 0.0, 0.0, 0.0)
  gl.enable(gl.DEPTH_TEST)
  gl.depthFunc(gl.LEQUAL)
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT)
  gl.clearDepth(1.0)

  gl.enable(gl.BLEND)
  gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)

  if (viewer._programId != null) {
    gl.useProgram(viewer._programId)
  }

  gl.flush()

  const { width, height } = viewer.canvas
  const projection = viewer.getNewMatrix44()

  const drawableModel = currentModel as DrawableLive2dModel
  const cubismModel = drawableModel.getModel()

  if (currentModel.isCompleteSetup && cubismModel != null) {
    modelReady.value = true

    if (cubismModel.getCanvasWidth() > 1.0 && width < height) {
      drawableModel.getModelMatrix().setWidth(2.0)
      projection.scale(1.0, width / height)
    } else {
      projection.scale(height / width, 1.0)
    }

    projection.multiplyByMatrix(viewer._viewMatrix)

    currentModel.update()
    currentModel.draw(projection, 0, 0, width, height, viewer.frameBuffer)
  }

  animationFrameId = requestAnimationFrame(drawFrame)
}

const releaseLive2d = () => {
  stopLoop()
  currentModel = null
  modelReady.value = false
  motionFiles.value = []
  expressionIds.value = []

  if (manager != null) {
    manager.release()
    manager = null
    viewer = null
  }
}

const loadModel = async () => {
  const canvas = canvasRef.value
  if (canvas == null) {
    return
  }

  status.value = 'loading'
  message.value = 'Live2Dを初期化しています。'

  try {
    releaseLive2d()
    await loadCubismCore()
    await nextTick()

    viewer = new Live2dViewer(canvas, canvas.clientWidth, canvas.clientHeight)
    manager = new Live2dManager(viewer)
    manager.initialize()

    const modelHomeDir = live2dAssetsDir + modelName.value

    currentModel = new Live2dModel(
      modelHomeDir,
      modelJsonFileName.value,
      viewer,
      readLive2dAsset,
    )

    await currentModel.loadAssets(preloadMotions.value)
    viewer.addModel('sample', currentModel)
    viewer.setCurrentModelkey('sample')

    motionFiles.value = currentModel.getMotionFileNameList()
    expressionIds.value = currentModel.getExpressionIdList()
    modelReady.value = currentModel.isCompleteSetup
    status.value = 'ready'
    message.value = `${modelHomeDir}${modelJsonFileName.value} を読み込みました。`
    drawFrame()
  } catch (error) {
    releaseLive2d()
    status.value = 'error'
    message.value = error instanceof Error ? error.message : String(error)
  }
}

const playMotion = async (fileName: string) => {
  if (currentModel == null) {
    return
  }
  
  console.log(`motion filename: ${fileName}`);

  const groupAndIndex = currentModel.getMotionGroupAndIndex(fileName)
  if (groupAndIndex == null) {
    return
  }

  await currentModel.startMotion(groupAndIndex[0], groupAndIndex[1])
}

const setExpression = (expressionId: string) => {
  currentModel?.setExpression(expressionId)
}

const clearExpression = () => {
  currentModel?.stopExpression()
}

const onPointerDown = (event: PointerEvent) => {
  canvasRef.value?.setPointerCapture(event.pointerId)
  viewer?.onTouchesBegin(event.clientX, event.clientY)
}

const onPointerMove = (event: PointerEvent) => {
  if (event.buttons === 0) {
    return
  }

  viewer?.onTouchesMoved(event.clientX, event.clientY)
}

const onPointerUp = (event: PointerEvent) => {
  canvasRef.value?.releasePointerCapture(event.pointerId)
  viewer?.onTouchesEnded()
}

onMounted(() => {
  window.addEventListener('resize', fitCanvasToDisplay)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', fitCanvasToDisplay)
  releaseLive2d()
})
</script>

<template>
  <main class="app-shell">
    <section class="stage">
      <canvas
        ref="canvasRef"
        class="viewer-canvas"
        aria-label="Live2D viewer"
        @pointerdown="onPointerDown"
        @pointermove="onPointerMove"
        @pointerup="onPointerUp"
        @pointercancel="onPointerUp"
      />
      <div v-if="status !== 'ready'" class="empty-state">
        <p>{{ message }}</p>
      </div>
    </section>

    <aside class="control-panel">
      <div class="panel-header">
        <p class="eyebrow">live2dmanager sample</p>
        <h1>Vue 3 SFC viewer</h1>
      </div>

      <label class="field">
        <span>Model Name</span>
        <input v-model="modelName" :disabled="isBusy" type="text" />
      </label>

      <label class="field">
        <span>model3.json</span>
        <input v-model="modelJsonFileName" :disabled="isBusy" type="text" />
      </label>

      <label class="checkbox-field">
        <input v-model="preloadMotions" :disabled="isBusy" type="checkbox" />
        <span>Load motions before first draw</span>
      </label>

      <button class="primary-action" :disabled="isBusy" type="button" @click="loadModel">
        {{ isBusy ? 'Loading...' : 'Load model' }}
      </button>

      <p class="status-line" :class="status">{{ message }}</p>

      <div class="requirements">
        <h2>Required files</h2>
        <code>public/live2dcubismcore.min.js</code>
        <code>public{{ live2dAssetsDir + modelName }}{{ modelJsonFileName }}</code>
      </div>

      <div v-if="motionFiles.length > 0" class="action-group">
        <h2>Motions</h2>
        <button
          v-for="motionFile in motionFiles"
          :key="motionFile"
          :disabled="!canUseModel"
          type="button"
          @click="playMotion(motionFile)"
        >
          {{ motionFile }}
        </button>
      </div>

      <div v-if="expressionIds.length > 0" class="action-group">
        <h2>Expressions</h2>
        <button
          v-for="expressionId in expressionIds"
          :key="expressionId"
          :disabled="!canUseModel"
          type="button"
          @click="setExpression(expressionId)"
        >
          {{ expressionId }}
        </button>
        <button :disabled="!canUseModel" type="button" @click="clearExpression">
          Clear expression
        </button>
      </div>
    </aside>
  </main>
</template>
