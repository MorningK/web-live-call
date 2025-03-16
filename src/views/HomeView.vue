<script setup lang="ts">
import { nextTick, onBeforeMount, onMounted, ref } from 'vue'
import { useToast } from 'vue-toast-notification'
import { getWaveBlob } from 'webm-to-wav-converter'

const mode = ref<'audio' | 'video'>('audio')
// 创建一个判断是否前置摄像头的ref
const isFrontCamera = ref<boolean>(false)
const videoRef = ref<HTMLVideoElement | null>(null)
const imageRef = ref<HTMLImageElement | null>(null)
// mediaStream的ref
const mediaStream = ref<MediaStream | null>(null)
const cleanupRegister = ref<Array<() => void>>([])
const isRecording = ref(false)
const toast = useToast()

// 在组件挂载时初始化媒体流
onMounted(() => {
  initMediaStream()
})

onBeforeMount(() => {
  cleanup()
})

// 实现切换mode的方法
const toggleMode = async () => {
  mode.value = mode.value === 'audio' ? 'video' : 'audio'
  cleanup()
  await nextTick()
  await initMediaStream()
}

const initMediaStream = async () => {
  const msg = toast.info('Initializing media stream...')
  const stream = await navigator.mediaDevices.getUserMedia({
    audio: {
      noiseSuppression: true,
    },
    video:
      mode.value === 'video'
        ? {
            facingMode: isFrontCamera.value ? 'user' : 'environment',
          }
        : false,
  })
  mediaStream.value = stream
  if (mode.value === 'video' && videoRef.value) {
    videoRef.value.srcObject = stream
    videoRef.value.onloadeddata = () => {
      msg.dismiss()
    }
  } else {
    msg.dismiss()
  }
}
// 切换前置摄像头的方法
const toggleFrontCamera = async () => {
  isFrontCamera.value = !isFrontCamera.value
  cleanup()
  await nextTick()
  initMediaStream()
}
// 捕捉用户的麦克风数据流并转换为wav格式的音频
const captureAudio = () => {
  if (!mediaStream.value) {
    console.warn('No media stream available')
    return
  }
  console.log('capture audio')
  let msg = toast.info('Recording audio...', { duration: 0 })
  const audioContext = new AudioContext()
  const analyser = audioContext.createAnalyser()
  const audioSource = audioContext.createMediaStreamSource(mediaStream.value)
  audioSource.connect(analyser)
  analyser.connect(audioContext.destination)
  let executor: number | null = null
  let audioCallbackId: number | null = null
  const detactVolume = () => {
    const dataArray = new Uint8Array(analyser.frequencyBinCount)
    analyser.getByteFrequencyData(dataArray)
    const volume = dataArray.reduce((acc, cur) => acc + cur, 0) / dataArray.length
    console.debug('Volume:', volume)
    if (volume > 16) {
      if (executor) {
        clearTimeout(executor)
        executor = null
      }
      if (recorder.state === 'inactive') {
        console.log('start recording')
        recorder.start()
        msg = toast.info('Recording audio...', { duration: 0 })
      }
    } else if (recorder.state === 'recording' && executor === null) {
      executor = setTimeout(() => {
        console.log('stop recording')
        recorder.stop()
        msg.dismiss()
        executor = null
      }, 1000)
    }
    audioCallbackId = requestAnimationFrame(detactVolume)
  }
  // 获取音频数据并转化为wav格式的文件并播放
  const recorder = new MediaRecorder(mediaStream.value)
  recorder.start()
  recorder.ondataavailable = async (event) => {
    if (event.data.size === 0) {
      return
    }
    const audioBlob = await getWaveBlob(event.data, false)
    const audioUrl = URL.createObjectURL(audioBlob)
    const audio = new Audio(audioUrl)
    console.log('audio play', event.data, audioBlob, audioUrl)
    audio.play()
    audio.onended = () => {
      URL.revokeObjectURL(audioUrl)
    }
    cleanupRegister.value.push(() => {
      audio.pause()
      audio.onended = null
      URL.revokeObjectURL(audioUrl)
    })
  }
  audioCallbackId = requestAnimationFrame(detactVolume)
  cleanupRegister.value.push(() => {
    if (audioCallbackId) {
      cancelAnimationFrame(audioCallbackId)
      audioCallbackId = null
    }
    if (executor) {
      clearTimeout(executor)
      executor = null
    }
    recorder.stop()
    recorder.ondataavailable = null
    analyser.disconnect()
    audioSource.disconnect()
    audioContext.close()
  })
}

// 捕捉用户的摄像头数据流并转换为图片
const captureVideo = () => {
  if (!mediaStream.value || !videoRef.value) {
    console.warn('No media stream available')
    return
  }
  const canvas = document.createElement('canvas')
  canvas.width = videoRef.value.videoWidth
  canvas.height = videoRef.value.videoHeight
  const ctx = canvas.getContext('2d')
  let videoCallbackId: number | null = null
  const drawImage: VideoFrameRequestCallback = (now, metadata) => {
    if (ctx && imageRef.value && videoRef.value) {
      console.debug('draw image', now, metadata)
      ctx.drawImage(videoRef.value, 0, 0, canvas.width, canvas.height)
      const imageUrl = canvas.toDataURL('image/jpg')
      imageRef.value.src = imageUrl
      videoCallbackId = videoRef.value.requestVideoFrameCallback(drawImage)
    }
  }
  videoCallbackId = videoRef.value.requestVideoFrameCallback(drawImage)
  cleanupRegister.value.push(() => {
    if (videoCallbackId) {
      videoRef.value?.cancelVideoFrameCallback(videoCallbackId)
      videoCallbackId = null
    }
  })
}
// 点击触发捕捉事件
const capture = () => {
  if (isRecording.value) {
    cleanup()
    return
  }
  isRecording.value = true
  captureAudio()
  if (mode.value === 'audio') {
  } else {
    captureVideo()
  }
}

const cleanup = () => {
  if (mediaStream.value) {
    mediaStream.value.getTracks().forEach((track) => {
      track.stop()
    })
    mediaStream.value = null
  }
  cleanupRegister.value.forEach((fn) => fn())
  cleanupRegister.value = []
  isRecording.value = false
}
</script>

<template>
  <main>
    <div>
      <h1>Mode: {{ mode }}</h1>
      <!-- 如何mode等于audio则显示文本段落，否则显示video元素 -->
      <p v-if="mode === 'audio'">Audio mode</p>
      <video v-else ref="videoRef" autoplay muted></video>
    </div>
    <button @click="toggleMode">Toggle Mode</button>
    <!-- 当mode等于video时显示切换前置摄像头的按钮 -->
    <button v-if="mode === 'video'" @click="toggleFrontCamera">Toggle Front Camera</button>
    <button @click="capture">{{ isRecording ? 'Stop' : 'Capture' }}</button>
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
