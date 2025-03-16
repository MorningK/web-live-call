<script setup lang="ts">
import { ref } from 'vue'
import { useToast, type ActiveToast } from 'vue-toast-notification'
import LiveCall from '@/components/LiveCall.vue'

const mode = ref<'audio' | 'video'>('audio')
const facingMode = ref<'user' | 'environment'>('user')
const liveCallRef = ref<InstanceType<typeof LiveCall> | null>(null)
const imageRef = ref<HTMLImageElement | null>(null)
const toast = useToast()
let msg: ActiveToast | null = null

// 实现切换mode的方法
const toggleMode = async () => {
  mode.value = mode.value === 'audio' ? 'video' : 'audio'
}
const handleInitMediaStream = () => {
  toast.info('Initializing media stream...')
}
const handleMediaStreamComplete = () => {
  msg?.dismiss()
}

// 切换前置摄像头的方法
const toggleFrontCamera = async () => {
  facingMode.value = facingMode.value === 'user' ? 'environment' : 'user'
}
const handleStartRecordAudio = () => {
  msg = toast.info('Recording audio...', { duration: 0 })
}
const handleStopRecordAudio = () => {
  msg?.dismiss()
}
const handleAudioUpdate = (audioBlob: Blob) => {
  const audioUrl = URL.createObjectURL(audioBlob)
  const audio = new Audio(audioUrl)
  console.log('audio play', audioBlob, audioUrl)
  audio.play()
  audio.onended = () => {
    URL.revokeObjectURL(audioUrl)
  }
}
const handleVideoUpdate = (imageUrl: string) => {
  if (!imageRef.value) {
    console.warn('No image element available')
    return
  }
  imageRef.value.src = imageUrl
}
// 点击触发捕捉事件
const capture = () => {
  liveCallRef.value?.startCapture()
}
</script>

<template>
  <main>
    <div>
      <h1>Mode: {{ mode }}</h1>
      <!-- 如何mode等于audio则显示文本段落，否则显示video元素 -->
      <LiveCall
        ref="liveCallRef"
        v-model:liveMode="mode"
        v-model:facingMode="facingMode"
        @initMediaStream="handleInitMediaStream"
        @mediaStreamComplete="handleMediaStreamComplete"
        @startRecordAudio="handleStartRecordAudio"
        @stopRecordAudio="handleStopRecordAudio"
        @audioUpdate="handleAudioUpdate"
        @videoUpdate="handleVideoUpdate"
      >
        <p>Audio mode</p>
      </LiveCall>
    </div>
    <button @click="toggleMode">Toggle Mode</button>
    <!-- 当mode等于video时显示切换前置摄像头的按钮 -->
    <button v-if="mode === 'video'" @click="toggleFrontCamera">Toggle Front Camera</button>
    <button @click="capture">{{ 'Capture' }}</button>
  </main>
  <img v-if="mode === 'video'" ref="imageRef" />
</template>

<style scoped>
/* 设置main的背景为灰色 */
main {
  background-color: #f0f0f0;
  padding: 20px;
}
div {
  background-color: white;
  padding: 20px;
  border-radius: 5px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  margin-bottom: 20px;
}
/* 设置button的样式 */
button {
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}
/* 设置button的hover样式 */
button:hover {
  background-color: #0056b3;
}
button + button {
  margin-left: 10px;
}
/* 设置video的样式 */
video {
  width: 100%;
  height: auto;
}
</style>
