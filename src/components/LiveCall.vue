<script setup lang="ts">
import { nextTick, onBeforeMount, onMounted, ref } from 'vue'
import { getWaveBlob } from 'webm-to-wav-converter'

const emit = defineEmits<{
  initMediaStream: []
  mediaStreamComplete: []
  startRecordAudio: []
  stopRecordAudio: []
  audioUpdate: [blob: Blob]
  videoUpdate: [dataUrl: string]
}>()

const liveMode = defineModel<'audio' | 'video'>('live-mode')
const facingMode = defineModel<'user' | 'environment'>('facingMode')
const videoRef = ref<HTMLVideoElement | null>(null)
// mediaStream的ref
const mediaStream = ref<MediaStream | null>(null)
const cleanupRegister = ref<Array<() => void>>([])
const isRecording = ref(false)

// 在组件挂载时初始化媒体流
onMounted(() => {
  initMediaStream()
})

onBeforeMount(() => {
  cleanup()
})

// 实现切换mode的方法
const changeLiveMode = async (value: typeof liveMode.value) => {
  if (value === liveMode.value) {
    return
  }
  liveMode.value = value
  cleanup()
  await nextTick()
  await initMediaStream()
}

const initMediaStream = async () => {
  emit('initMediaStream')
  const stream = await navigator.mediaDevices.getUserMedia({
    audio: {
      noiseSuppression: true,
    },
    video:
      liveMode.value === 'video'
        ? {
            facingMode: facingMode.value,
          }
        : false,
  })
  mediaStream.value = stream
  if (liveMode.value === 'video' && videoRef.value) {
    videoRef.value.srcObject = stream
    videoRef.value.onloadeddata = () => {
      emit('mediaStreamComplete')
    }
  } else {
    emit('mediaStreamComplete')
  }
}
// 切换前置摄像头的方法
const changeFacingMode = async (value: typeof facingMode.value) => {
  facingMode.value = value
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
  emit('startRecordAudio')
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
        emit('startRecordAudio')
      }
    } else if (recorder.state === 'recording' && executor === null) {
      executor = setTimeout(() => {
        console.log('stop recording')
        recorder.stop()
        emit('stopRecordAudio')
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
    emit('audioUpdate', audioBlob)
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
    if (ctx && videoRef.value) {
      console.debug('draw image', now, metadata)
      ctx.drawImage(videoRef.value, 0, 0, canvas.width, canvas.height)
      const imageUrl = canvas.toDataURL('image/jpeg')
      emit('videoUpdate', imageUrl)
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

const startCapture = () => {
  if (isRecording.value) {
    return
  }
  isRecording.value = true
  captureAudio()
  if (liveMode.value === 'video') {
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

defineExpose({
  mode: liveMode,
  changeLiveMode,
  changeFacingMode,
  startCapture,
  cleanup,
})
</script>

<template>
  <template v-if="liveMode === 'audio'">
    <slot></slot>
  </template>
  <video v-else ref="videoRef" autoplay muted controls="false"></video>
</template>

<style scoped>
video {
  width: 100%;
  height: auto;
}
</style>
