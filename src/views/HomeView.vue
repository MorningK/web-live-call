<script setup lang="ts">
import { nextTick, onMounted, ref } from 'vue'
import { useToast } from 'vue-toast-notification'

const mode = ref<'audio' | 'video'>('audio')
// 创建一个判断是否前置摄像头的ref
const isFrontCamera = ref<boolean>(false)
const videoRef = ref<HTMLVideoElement | null>(null)
// mediaStream的ref
const mediaStream = ref<MediaStream | null>(null)
const toast = useToast()

// 在组件挂载时初始化媒体流
onMounted(() => {
  initMediaStream()
})

// 实现切换mode的方法
const toggleMode = async () => {
  mode.value = mode.value === 'audio' ? 'video' : 'audio'
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
  }
  msg.dismiss()
}
// 切换前置摄像头的方法
const toggleFrontCamera = () => {
  isFrontCamera.value = !isFrontCamera.value
  initMediaStream()
}
// 捕捉用户的麦克风数据流并转换为wav格式的音频
const captureAudio = () => {
  if (!mediaStream.value) {
    console.warn('No media stream available')
    return
  }
  let msg = toast.info('Recording audio...', { duration: 0 })
  const audioContext = new AudioContext()
  const analyser = audioContext.createAnalyser()
  const gain = audioContext.createGain()
  const audioSource = audioContext.createMediaStreamSource(mediaStream.value)
  audioSource.connect(gain)
  gain.connect(analyser)
  analyser.connect(audioContext.destination)
  function detactVolume() {
    const dataArray = new Uint8Array(analyser.frequencyBinCount)
    analyser.getByteFrequencyData(dataArray)
    const volume = dataArray.reduce((acc, cur) => acc + cur, 0) / dataArray.length
    console.debug('Volume:', volume)
    if (volume > 10) {
      if (recorder.state === 'inactive') {
        recorder.start()
        msg = toast.info('Recording audio...', { duration: 0 })
      }
    } else {
      recorder.stop()
      msg.dismiss()
    }
    requestAnimationFrame(detactVolume)
  }
  // 获取音频数据并转化为wav格式的文件并播放
  const recorder = new MediaRecorder(mediaStream.value)
  recorder.stream.getAudioTracks().forEach((track) => {
    track.onunmute = () => {
      console.log('Audio track unmuted', track)
    }
    track.onmute = () => {
      console.log('Audio track muted', track)
    }
  })
  recorder.start()
  recorder.ondataavailable = (event) => {
    if (event.data.size === 0) {
      return
    }
    const audioBlob = new Blob([event.data])
    const audioUrl = URL.createObjectURL(audioBlob)
    const audio = new Audio(audioUrl)
    console.log('audio play', event.data, audioBlob, audioUrl)
    audio.play()
    audio.onended = () => {
      URL.revokeObjectURL(audioUrl)
    }
  }
  requestAnimationFrame(detactVolume)
}
// 捕捉用户的摄像头数据流并转换为图片
const captureVideo = () => {
  if (!mediaStream.value) {
    console.warn('No media stream available')
    return
  }
  const canvas = document.createElement('canvas')
  const videoTrack = mediaStream.value.getVideoTracks()[0]
  const imageCapture = new ImageCapture(videoTrack)
  imageCapture.grabFrame().then((imageBitmap) => {
    canvas.width = imageBitmap.width
    canvas.height = imageBitmap.height
    const ctx = canvas.getContext('2d')
    if (ctx) {
      ctx.drawImage(imageBitmap, 0, 0)
      const imageUrl = canvas.toDataURL('image/png')
      const image = new Image()
      image.src = imageUrl
      document.body.appendChild(image)
    }
  })
}
// 点击触发捕捉事件
const capture = () => {
  captureAudio()
  if (mode.value === 'audio') {
  } else {
    captureVideo()
  }
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
    <button @click="capture">Capture</button>
  </main>
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
