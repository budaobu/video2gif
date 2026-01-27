<script setup lang="ts">
import { ref, reactive, onMounted, nextTick } from 'vue'
import ThemeToggle from './components/ThemeToggle.vue'
import {
  FilmIcon,
  ArrowUpTrayIcon,
  VideoCameraIcon,
  CheckCircleIcon,
  ExclamationTriangleIcon,
  AdjustmentsHorizontalIcon,
  ArrowPathIcon,
  InformationCircleIcon,
  SparklesIcon,
  PhotoIcon,
  ArrowDownTrayIcon
} from '@heroicons/vue/24/outline'

// --- 状态管理 ---
const isLibLoaded = ref(false)
const videoFile = ref<File | null>(null)
const videoUrl = ref<string>('')
const workerBlobUrl = ref<string>('')
const isProcessing = ref(false)
const progress = ref(0)
const progressText = ref('')
const loadingText = ref('')
const resultUrl = ref('')
const resultSize = ref('')

// 设置项
const settings = reactive({
  loop: true, // 默认开启循环
  width: 480,
  fps: 10
})

// DOM 引用
const sourceVideoRef = ref<HTMLVideoElement | null>(null)
const fileInputRef = ref<HTMLInputElement | null>(null)

// --- 核心逻辑 ---

// 1. 初始化 Worker
const initWorker = async () => {
  if (workerBlobUrl.value) return true
  try {
    const response = await fetch('https://cdnjs.cloudflare.com/ajax/libs/gif.js/0.2.0/gif.worker.js')
    const workerScript = await response.text()
    const blob = new Blob([workerScript], { type: 'application/javascript' })
    workerBlobUrl.value = URL.createObjectURL(blob)
    return true
  } catch (error) {
    console.error("Failed to load gif worker:", error)
    alert("初始化组件失败，请检查网络连接")
    return false
  }
}

// 2. 处理文件选择
const handleFileChange = (event: Event) => {
  const input = event.target as HTMLInputElement
  if (input.files && input.files[0]) {
    processFile(input.files[0])
  }
}

const handleDrop = (event: DragEvent) => {
  event.preventDefault()
  if (event.dataTransfer?.files && event.dataTransfer.files[0]) {
    processFile(event.dataTransfer.files[0])
  }
}

const processFile = (file: File) => {
  if (!file.type.startsWith('video/')) {
    alert('请上传视频文件')
    return
  }
  
  // 清理旧资源
  if (videoUrl.value) URL.revokeObjectURL(videoUrl.value)
  if (resultUrl.value) {
    URL.revokeObjectURL(resultUrl.value)
    resultUrl.value = ''
  }

  videoFile.value = file
  videoUrl.value = URL.createObjectURL(file)
  
  // 重置状态
  progress.value = 0
  progressText.value = ''
  
  // 等待视频加载元数据
  nextTick(() => {
    if (sourceVideoRef.value) {
      sourceVideoRef.value.load()
    }
  })
}

// 3. 开始转换
const startConversion = async () => {
  if (!videoFile.value || !sourceVideoRef.value) return
  if (!workerBlobUrl.value) {
    const success = await initWorker()
    if (!success) return
  }

  // @ts-ignore
  if (typeof GIF === 'undefined') {
    alert('组件尚未加载完成，请稍候...')
    return
  }

  isProcessing.value = true
  loadingText.value = '正在初始化...'
  
  const width = settings.width
  const ratio = sourceVideoRef.value.videoHeight / sourceVideoRef.value.videoWidth
  const height = Math.round(width * ratio)
  
  const canvas = document.createElement('canvas')
  canvas.width = width
  canvas.height = height
  const ctx = canvas.getContext('2d', { willReadFrequently: true })
  if (!ctx) return

  // @ts-ignore
  const gif = new GIF({
    workers: 2,
    quality: 10,
    width: width,
    height: height,
    workerScript: workerBlobUrl.value,
    repeat: settings.loop ? 0 : -1, // 关键逻辑：0 为循环，-1 为不循环
    background: '#fff'
  })

  gif.on('progress', (p: number) => {
    progress.value = Math.round(p * 100)
    progressText.value = `${progress.value}%`
    loadingText.value = '正在编码 GIF...'
  })

  gif.on('finished', (blob: Blob) => {
    resultUrl.value = URL.createObjectURL(blob)
    resultSize.value = (blob.size / 1024 / 1024).toFixed(2)
    isProcessing.value = false
    loadingText.value = ''
  })

  // 捕获帧
  loadingText.value = '正在捕获视频帧...'
  const video = sourceVideoRef.value
  const duration = video.duration
  const interval = 1 / settings.fps
  let currentTime = 0
  let frameCount = 0
  const maxFrames = 300 

  const captureFrame = async () => {
    // Check if processing was cancelled or unmounted (optional safety)
    if (!isProcessing.value && frameCount > 0 && !resultUrl.value) return 

    if (frameCount >= maxFrames || currentTime >= duration) {
      loadingText.value = '开始渲染...'
      gif.render()
      return
    }

    video.currentTime = currentTime
    
    await new Promise<void>(resolve => {
      const handleSeek = () => {
        resolve()
      }
      video.addEventListener('seeked', handleSeek, { once: true })
    })

    ctx.drawImage(video, 0, 0, width, height)
    gif.addFrame(ctx, { copy: true, delay: interval * 1000 })
    
    frameCount++
    const captureProgress = Math.round((currentTime / duration) * 100)
    progressText.value = `捕获: ${captureProgress}% (${frameCount}帧)`
    
    currentTime += interval
    setTimeout(captureFrame, 0)
  }

  video.pause()
  captureFrame()
}

const triggerFileInput = () => {
  fileInputRef.value?.click()
}

const loadGifJs = () => {
    const script = document.createElement('script')
    script.src = 'https://cdnjs.cloudflare.com/ajax/libs/gif.js/0.2.0/gif.js'
    script.onload = () => { isLibLoaded.value = true }
    document.head.appendChild(script)
}

onMounted(() => {
  loadGifJs()
  initWorker()
  document.title = '视频转 GIF 工具'
})
</script>

<template>
  <div class="min-h-screen bg-gray-50 dark:bg-gray-950 text-gray-900 dark:text-gray-100 font-sans transition-colors duration-300">
    
    <!-- Navbar -->
    <nav class="border-b border-gray-200 dark:border-gray-800 bg-white dark:bg-gray-900">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
            <div class="flex justify-between h-16">
                <div class="flex items-center gap-2">
                    <div class="flex items-center justify-center w-8 h-8 rounded-full bg-primary-100 dark:bg-primary-900/30 text-primary-600 dark:text-primary-400">
                        <FilmIcon class="w-5 h-5" />
                    </div>
                    <span class="font-bold text-xl">Video2Gif</span>
                </div>
                <div class="flex items-center gap-4">
                    <ThemeToggle />
                    <a href="https://github.com/budaobu/video2gif" target="_blank" class="text-gray-500 hover:text-gray-900 dark:hover:text-white transition-colors">
                        <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 24 24" aria-hidden="true">
                            <path fill-rule="evenodd" d="M12 2C6.477 2 2 6.484 2 12.017c0 4.425 2.865 8.18 6.839 9.504.5.092.682-.217.682-.483 0-.237-.008-.868-.013-1.703-2.782.605-3.369-1.343-3.369-1.343-.454-1.158-1.11-1.466-1.11-1.466-.908-.62.069-.608.069-.608 1.003.07 1.531 1.032 1.531 1.032.892 1.53 2.341 1.088 2.91.832.092-.647.35-1.088.636-1.338-2.22-.253-4.555-1.113-4.555-4.951 0-1.093.39-1.988 1.029-2.688-.103-.253-.446-1.272.098-2.65 0 0 .84-.27 2.75 1.026A9.564 9.564 0 0112 6.844c.85.004 1.705.115 2.504.337 1.909-1.296 2.747-1.027 2.747-1.027.546 1.379.202 2.398.1 2.651.64.7 1.028 1.595 1.028 2.688 0 3.848-2.339 4.695-4.566 4.943.359.309.678.92.678 1.855 0 1.338-.012 2.419-.012 2.747 0 .268.18.58.688.482A10.019 10.019 0 0022 12.017C22 6.484 17.522 2 12 2z" clip-rule="evenodd" />
                        </svg>
                    </a>
                </div>
            </div>
        </div>
    </nav>

    <div class="py-12 px-4 sm:px-6">
    <!-- Header -->
    <div class="max-w-3xl mx-auto text-center mb-12">
      <h1 class="text-3xl font-bold mb-3 bg-clip-text text-transparent bg-gradient-to-r from-blue-500 to-purple-500">视频转 GIF 工具</h1>
      <p class="text-gray-500 dark:text-gray-400">
        纯前端处理，视频不上传服务器，保护隐私更安全。
      </p>
    </div>

    <div class="max-w-6xl mx-auto grid grid-cols-1 lg:grid-cols-2 gap-8">
      
      <!-- Left Column: Input & Settings -->
      <div class="flex flex-col gap-6">
        
        <!-- 1. Upload Area -->
        <div class="bg-white dark:bg-gray-800 shadow rounded-lg overflow-hidden">
          <div class="px-4 py-5 sm:px-6 border-b border-gray-200 dark:border-gray-700">
            <h2 class="text-lg font-semibold flex items-center gap-2">
              <ArrowUpTrayIcon class="w-5 h-5 text-blue-500" /> 
              1. 上传视频
            </h2>
          </div>

          <div class="p-6">
            <div 
                class="relative border-2 border-dashed border-gray-300 dark:border-gray-700 rounded-xl p-8 text-center hover:border-blue-500 dark:hover:border-blue-500 hover:bg-blue-50 dark:hover:bg-blue-900/10 transition-all cursor-pointer group"
                @click="triggerFileInput"
                @dragover.prevent
                @drop.prevent="handleDrop"
            >
                <input 
                type="file" 
                ref="fileInputRef" 
                accept="video/*" 
                class="hidden" 
                @change="handleFileChange"
                >
                
                <div v-if="!videoFile">
                <VideoCameraIcon class="w-10 h-10 mx-auto text-gray-400 group-hover:text-blue-500 mb-3 transition-colors" />
                <p class="text-sm font-medium text-gray-600 dark:text-gray-300">点击或拖拽视频文件到此处</p>
                <p class="text-xs text-gray-400 mt-1">支持 MP4, WebM, MOV</p>
                </div>

                <div v-else class="text-left flex items-center gap-4">
                <CheckCircleIcon class="w-8 h-8 text-green-500 shrink-0" />
                <div class="min-w-0 flex-1">
                    <p class="text-sm font-bold truncate">{{ videoFile.name }}</p>
                    <p class="text-xs text-gray-500">{{ (videoFile.size / 1024 / 1024).toFixed(1) }} MB</p>
                </div>
                <button class="text-xs px-2 py-1 rounded bg-gray-100 dark:bg-gray-700 hover:bg-gray-200 dark:hover:bg-gray-600 transition-colors" @click.stop="triggerFileInput">更换</button>
                </div>
            </div>

            <!-- Video Preview -->
            <video 
                ref="sourceVideoRef" 
                class="mt-4 w-full rounded-lg bg-black aspect-video object-contain" 
                :class="{ 'hidden': !videoUrl }"
                :src="videoUrl" 
                controls 
                playsinline 
                muted
            ></video>

            <!-- Large File Warning -->
            <div v-if="videoFile && videoFile.size > 50 * 1024 * 1024" class="mt-4 p-3 bg-red-50 dark:bg-red-900/20 text-red-600 dark:text-red-400 text-xs rounded-lg flex gap-2 items-start">
                <ExclamationTriangleIcon class="w-5 h-5 shrink-0" />
                <span>视频文件较大 (>50MB)，转换可能导致浏览器崩溃，建议先剪辑。</span>
            </div>
          </div>
        </div>

        <!-- 2. Settings Area -->
        <div class="bg-white dark:bg-gray-800 shadow rounded-lg overflow-hidden">
          <div class="px-4 py-5 sm:px-6 border-b border-gray-200 dark:border-gray-700">
            <h2 class="text-lg font-semibold flex items-center gap-2">
              <AdjustmentsHorizontalIcon class="w-5 h-5 text-blue-500" />
              2. 转换设置
            </h2>
          </div>

          <div class="p-6 space-y-6">
            <!-- Loop Toggle -->
            <div class="flex items-center justify-between p-3 bg-gray-50 dark:bg-gray-700/50 rounded-lg border border-gray-200 dark:border-gray-700 cursor-pointer hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors" @click="settings.loop = !settings.loop">
              <div class="flex items-center gap-3">
                <div class="flex items-center justify-center w-8 h-8 rounded bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-600 text-blue-500">
                  <ArrowPathIcon class="w-5 h-5" />
                </div>
                <div>
                  <span class="block font-medium text-sm">循环播放</span>
                  <span class="block text-xs text-gray-500 dark:text-gray-400">GIF 播放完毕后是否重新开始</span>
                </div>
              </div>
              
              <div 
                class="relative inline-flex h-6 w-11 flex-shrink-0 cursor-pointer rounded-full border-2 border-transparent transition-colors duration-200 ease-in-out focus:outline-none"
                :class="settings.loop ? 'bg-blue-500' : 'bg-gray-200 dark:bg-gray-600'"
              >
                <span 
                  aria-hidden="true" 
                  class="pointer-events-none inline-block h-5 w-5 transform rounded-full bg-white shadow ring-0 transition duration-200 ease-in-out"
                  :class="settings.loop ? 'translate-x-5' : 'translate-x-0'"
                />
              </div>
            </div>

            <div class="grid grid-cols-2 gap-4">
              <div class="space-y-1">
                <label class="text-xs font-semibold text-gray-500 uppercase">宽度 (px)</label>
                <input v-model="settings.width" type="number" placeholder="480" class="w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md bg-white dark:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-500" />
                <p class="text-[10px] text-gray-400">建议 300-600</p>
              </div>
              <div class="space-y-1">
                <label class="text-xs font-semibold text-gray-500 uppercase">帧率 (FPS)</label>
                <input v-model="settings.fps" type="number" max="24" min="1" placeholder="10" class="w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md bg-white dark:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-500" />
                <p class="text-[10px] text-gray-400">建议 5-15</p>
              </div>
            </div>

            <div class="p-3 bg-yellow-50 dark:bg-yellow-900/20 text-yellow-800 dark:text-yellow-200 text-xs rounded-lg border border-yellow-100 dark:border-yellow-900/30 flex gap-2">
              <InformationCircleIcon class="w-5 h-5 shrink-0" />
              <span>提示：如果视频较长，请减少帧率或宽度。浏览器内存有限，转换长视频可能失败。</span>
            </div>

            <button 
              class="w-full flex justify-center items-center gap-2 py-3 px-4 border border-transparent rounded-lg shadow-sm text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500 disabled:opacity-50 disabled:cursor-not-allowed transition-all"
              :disabled="!videoFile || isProcessing"
              @click="startConversion"
            >
              <SparklesIcon v-if="!isProcessing" class="w-5 h-5" />
              <ArrowPathIcon v-else class="w-5 h-5 animate-spin" />
              {{ isProcessing ? '转换中...' : '开始转换' }}
            </button>
          </div>
        </div>
      </div>

      <!-- Right Column: Result -->
      <div class="flex flex-col h-full">
        <div class="bg-white dark:bg-gray-800 shadow rounded-lg h-full flex flex-col overflow-hidden">
          <div class="px-4 py-5 sm:px-6 border-b border-gray-200 dark:border-gray-700">
            <h2 class="text-lg font-semibold flex items-center gap-2">
              <PhotoIcon class="w-5 h-5 text-blue-500" />
              3. 结果预览
            </h2>
          </div>

          <div class="flex-1 p-6 flex flex-col">
            <div class="flex-1 bg-gray-100 dark:bg-gray-900 rounded-xl border-2 border-dashed border-gray-200 dark:border-gray-700 flex flex-col items-center justify-center p-4 relative overflow-hidden min-h-[300px]">
                
                <!-- Empty State -->
                <div v-if="!isProcessing && !resultUrl" class="text-center">
                <FilmIcon class="w-16 h-16 text-gray-300 dark:text-gray-600 mb-3 mx-auto" />
                <p class="text-gray-400 dark:text-gray-500">生成的 GIF 将显示在这里</p>
                </div>

                <!-- Loading State -->
                <div v-if="isProcessing" class="absolute inset-0 bg-white/90 dark:bg-gray-800/90 z-20 flex flex-col items-center justify-center p-8">
                <ArrowPathIcon class="w-10 h-10 text-blue-500 animate-spin mb-4" />
                <h3 class="text-lg font-bold text-gray-700 dark:text-white mb-1">正在处理</h3>
                <p class="text-sm text-gray-500 mb-4">{{ loadingText }}</p>
                
                <!-- Progress Bar -->
                <div class="w-full max-w-xs h-2 bg-gray-200 dark:bg-gray-700 rounded-full overflow-hidden">
                    <div class="h-full bg-blue-500 transition-all duration-300" :style="{ width: `${progress}%` }"></div>
                </div>
                <p class="text-xs text-gray-400 mt-2">{{ progressText }}</p>
                </div>

                <!-- Result -->
                <img 
                v-if="resultUrl" 
                :src="resultUrl" 
                class="max-w-full max-h-[500px] object-contain shadow-lg rounded-md bg-white select-none pointer-events-none" 
                alt="Generated GIF" 
                />
            </div>

            <div v-if="resultUrl" class="mt-6 grid grid-cols-2 gap-4">
                <div class="flex flex-col justify-center">
                <span class="text-sm text-gray-500">大小: {{ resultSize }} MB</span>
                </div>
                <a
                :href="resultUrl"
                download="converted.gif"
                class="flex justify-center items-center gap-2 py-2 px-4 border border-transparent rounded-lg shadow-sm text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500 transition-all text-center no-underline"
                >
                <ArrowDownTrayIcon class="w-5 h-5" />
                下载 GIF
                </a>
            </div>
          </div>
        </div>
      </div>

    </div>
  </div>
  </div>
</template>
