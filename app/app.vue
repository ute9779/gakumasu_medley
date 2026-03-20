<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, computed } from 'vue'

// 再生リストの型定義
interface Song {
  id: string
  start: number
  duration: number
  title: string
}

const playlist: Song[] = [
  { id: '4ALFjALIc5Q', start: 45, duration: 15, title: 'Fighting My Way' },
  { id: 'NBL9sMFXj1g', start: 54, duration: 15, title: 'Luna say maybe' },
  { id: '6D0n7hzfZiY', start: 52, duration: 15, title: '世界一可愛い私' },
  { id: '4TchmSegna0', start: 77, duration: 15, title: 'Fluorite' },
  { id: '9qCDnZY3_G0', start: 59, duration: 15, title: '白線' },
  { id: 'eD3jimowimI', start: 56, duration: 15, title: 'Wonder Scale' },
  { id: 'tW_Em8uObwA', start: 58, duration: 15, title: 'Tame-Lie-One-Step' },
  { id: 'jYqdVV-uFgg', start: 50, duration: 15, title: '光景' },
  { id: 'P7Dg1RPe8uA', start: 56, duration: 15, title: 'clumsy trick' },
  { id: '568aZaWSTbk', start: 64, duration: 15, title: 'The Rolling Riceball' },
  { id: 'fGU4kGofuyg', start: 56, duration: 15, title: '小さな野望' },
  { id: '6TZKFFRzwzs', start: 195, duration: 15, title: 'ツキノカメ' },

  

  { id: '7ANTx8MZp68', start: 56, duration: 15, title: '極光' },
  { id: 'Ie_btWuTW6M', start: 64, duration: 15, title: 'Superlative' },

  { id: 'XqIhCSiRIO4', start: 50, duration: 10, title: '冠菊' },
  { id: 'sFq_Fc33IrA', start: 59, duration: 10, title: 'ENDLESS DANCE (花海咲季・月村手毬・藤田ことね ver.)' },
  { id: 'VTtL0-ougbI', start: 54, duration: 10, title: 'ハッピーミルフィーユ' },
  { id: 'ISoaL8q9TgI', start: 45, duration: 10, title: 'White Night! White Wish!' },
  { id: 'prlcJJxDcG4', start: 72, duration: 10, title: 'ミラクルナナウ (ﾟ∀ﾟ) ！ (有村麻央・紫雲清夏・篠澤広 ver.)' },
  { id: 'F-JRSzq4zuo', start: 54, duration: 10, title: 'ナイワ' },
  
  
  { id: 'Fhwbl82WdcA', start: 69, duration: 15, title: '自己肯定感爆上げ↑↑しゅきしゅきソング' },
  { id: 'uKG6NrHbEQ8', start: 190, duration: 15, title: 'Atmosphere' },
  { id: 'cMHUxNhw_3s', start: 50, duration: 15, title: '36℃ U･B･U' },
  { id: 'G8vzMH9c5Xs', start: 59, duration: 15, title: '空と約束' },
]

const isStarted = ref(false)
const isPaused = ref(false)
const isReady = ref(false)
const currentIndex = ref(0)
const isProcessing = ref(false)

let player: any = null
let timer: ReturnType<typeof setTimeout> | null = null
let fadeInterval: ReturnType<typeof setInterval> | null = null
let startTime = 0
let remainingTime = 0

// 背景用のサムネイルURL
const currentThumb = computed(() => `https://i.ytimg.com/vi/${playlist[currentIndex.value]?.id}/maxresdefault.jpg`)

const clearAllTimers = () => {
  if (timer) clearTimeout(timer)
  if (fadeInterval) clearInterval(fadeInterval)
}

const initPlayer = () => {
  player = new (window as any).YT.Player('main-player', {
    height: '100%',
    width: '100%',
    playerVars: { 
      'autoplay': 0, 
      'controls': 1, // 規約遵守のためコントロールを表示
      'rel': 0, 
      'playsinline': 1,
      'modestbranding': 1 
    },
    events: {
      'onReady': () => { isReady.value = true },
      'onStateChange': onPlayerStateChange
    }
  })
}

const switchNext = () => {
  if (isPaused.value) return
  currentIndex.value = (currentIndex.value + 1) % playlist.length
  playSong()
}

const fade = (target: number, ms: number, callback?: () => void) => {
  if (fadeInterval) clearInterval(fadeInterval)
  const startVol = player?.getVolume ? player.getVolume() : 0
  const step = (target - startVol) / 10
  let count = 0
  
  fadeInterval = setInterval(() => {
    count++
    if (player?.setVolume) {
      player.setVolume(Math.max(0, Math.min(100, startVol + (step * count))))
    }
    if (count >= 10) {
      if (fadeInterval) clearInterval(fadeInterval)
      if (callback) callback()
    }
  }, ms / 10)
}

const startTimers = () => {
  startTime = Date.now()
  if (timer) clearTimeout(timer)
  timer = setTimeout(() => {
    fade(0, 1000, () => switchNext())
  }, remainingTime - 1000)
}

const onPlayerStateChange = (event: any) => {
  if (event.data === (window as any).YT.PlayerState.PLAYING) {
    if (!isPaused.value) {
      player.setVolume(0)
      fade(100, 1000)
      startTimers()
    }
    isProcessing.value = false
  }
}

const playSong = () => {
  clearAllTimers()
  const song = playlist[currentIndex.value]
  if (!song) return
  remainingTime = song.duration * 1000
  if (player && player.loadVideoById) {
    player.loadVideoById({ 
      videoId: song.id, 
      startSeconds: song.start 
    })
  }
}

const startMedley = () => {
  if (isProcessing.value) return
  isStarted.value = true
  playSong()
}

const togglePause = () => {
  if (isProcessing.value) return
  isProcessing.value = true
  if (isPaused.value) {
    isPaused.value = false
    player.playVideo()
    fade(100, 500)
  } else {
    isPaused.value = true
    clearAllTimers()
    remainingTime -= (Date.now() - startTime)
    fade(0, 500, () => {
      player.pauseVideo()
      isProcessing.value = false
    })
  }
}

onMounted(() => {
  if (!(window as any).YT) {
    const tag = document.createElement('script')
    tag.src = "https://www.youtube.com/iframe_api"
    document.head.appendChild(tag)
  }
  (window as any).onYouTubeIframeAPIReady = () => initPlayer()
})

onBeforeUnmount(() => {
  clearAllTimers()
  if (player) player.destroy()
})
</script>

<template>
  <div class="medley-wrapper">
    <div class="bg-blur" :style="{ backgroundImage: `url(${currentThumb})` }"></div>

    <div class="monitor">
      <div class="card">
        <div class="video-container">
          <div id="main-player"></div>
          
          <div v-if="!isStarted" class="start-overlay">
            <button @click="startMedley" :disabled="!isReady || isProcessing" class="play-btn">
              {{ isReady ? 'PRODUCE START' : 'LOADING...' }}
            </button>
          </div>
        </div>

        <div class="info-area" v-if="isStarted">
          <p class="now-playing">NOW PLAYING</p>
          <h2 class="title-text">{{ playlist[currentIndex]?.title }}</h2>
          
          <div class="progress-bar-bg">
            <div 
              class="progress-bar-fill" 
              :style="{ animationDuration: playlist[currentIndex]?.duration + 's' }" 
              :class="{ paused: isPaused }" 
              :key="currentIndex"
            ></div>
          </div>

          <div class="controls">
            <button @click="togglePause" :disabled="isProcessing" class="ctrl-btn">
              {{ isPaused ? 'RESUME' : 'PAUSE' }}
            </button>
            <a :href="`https://www.youtube.com/watch?v=${playlist[currentIndex]?.id}`" target="_blank" class="link-btn">
              Youtubeで見る
            </a>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.medley-wrapper {
  background: #000;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: 'Avenir', sans-serif;
  position: relative;
  overflow: hidden;
}

.bg-blur {
  position: absolute;
  inset: -20px;
  background-size: cover;
  background-position: center;
  filter: blur(30px) brightness(0.3);
  z-index: 0;
  transition: background-image 1s ease;
}

.monitor {
  width: 90%;
  max-width: 500px;
  z-index: 1;
}

.card {
  background: rgba(30, 30, 30, 0.9);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  overflow: hidden;
  box-shadow: 0 30px 60px rgba(0,0,0,0.8);
  border: 1px solid rgba(255,255,255,0.1);
}

.video-container {
  position: relative;
  width: 100%;
  aspect-ratio: 16 / 9;
  background: #000;
}

.start-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(0,0,0,0.7);
  z-index: 10;
}

.play-btn {
  background: #f06;
  color: #fff;
  border: none;
  padding: 18px 36px;
  border-radius: 40px;
  font-weight: bold;
  cursor: pointer;
  box-shadow: 0 0 20px rgba(255, 0, 102, 0.5);
  transition: transform 0.2s;
}

.play-btn:active { transform: scale(0.95); }

.info-area { padding: 25px; text-align: center; }
.now-playing { font-size: 11px; color: #f06; letter-spacing: 3px; margin: 0; font-weight: 900; }
.title-text { 
  font-size: 20px; 
  margin: 12px 0 20px; 
  color: #fff; 
  white-space: nowrap; 
  overflow: hidden; 
  text-overflow: ellipsis; 
}

.progress-bar-bg { width: 100%; height: 4px; background: rgba(255,255,255,0.1); border-radius: 2px; margin-bottom: 25px; overflow: hidden; }
.progress-bar-fill { height: 100%; background: #f06; width: 0; animation: fill linear forwards; }
.progress-bar-fill.paused { animation-play-state: paused; }

.controls { display: flex; gap: 10px; justify-content: center; }
.ctrl-btn, .link-btn {
  background: transparent;
  border: 1px solid rgba(255,255,255,0.2);
  color: #fff;
  padding: 10px 20px;
  border-radius: 25px;
  cursor: pointer;
  font-size: 12px;
  text-decoration: none;
  display: flex;
  align-items: center;
  transition: 0.3s;
}
.ctrl-btn:hover, .link-btn:hover { background: rgba(255,255,255,0.1); border-color: #fff; }

@keyframes fill { from { width: 0%; } to { width: 100%; } }
</style>