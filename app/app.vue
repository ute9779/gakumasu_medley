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

  

  // { id: '7ANTx8MZp68', start: 56, duration: 15, title: '極光' },
  // { id: 'Ie_btWuTW6M', start: 64, duration: 15, title: 'Superlative' },

  // { id: 'XqIhCSiRIO4', start: 50, duration: 10, title: '冠菊' },
  // { id: 'sFq_Fc33IrA', start: 59, duration: 10, title: 'ENDLESS DANCE (花海咲季・月村手毬・藤田ことね ver.)' },
  // { id: 'VTtL0-ougbI', start: 54, duration: 10, title: 'ハッピーミルフィーユ' },
  // { id: 'ISoaL8q9TgI', start: 45, duration: 10, title: 'White Night! White Wish!' },
  // { id: 'prlcJJxDcG4', start: 72, duration: 10, title: 'ミラクルナナウ (ﾟ∀ﾟ) ！ (有村麻央・紫雲清夏・篠澤広 ver.)' },
  // { id: 'F-JRSzq4zuo', start: 54, duration: 10, title: 'ナイワ' },
  
  
  // { id: 'Fhwbl82WdcA', start: 69, duration: 15, title: '自己肯定感爆上げ↑↑しゅきしゅきソング' },
  // { id: 'uKG6NrHbEQ8', start: 190, duration: 15, title: 'Atmosphere' },
  // { id: 'cMHUxNhw_3s', start: 50, duration: 15, title: '36℃ U･B･U' },
  // { id: 'G8vzMH9c5Xs', start: 59, duration: 15, title: '空と約束' },
]

// --- 状態管理 ---
const isStarted = ref(false)      // メドレーが開始されているか
const isPaused = ref(false)       // 一時停止中か
const isReady = ref(false)        // YouTube APIの準備が完了したか
const currentIndex = ref(0)       // 現在再生中の曲のインデックス
const isProcessing = ref(false)   // 連打防止などの処理中フラグ

let player: any = null            // YouTubeプレイヤーのインスタンス
let timer: ReturnType<typeof setTimeout> | null = null        // 次の曲へ進むためのタイマー
let fadeInterval: ReturnType<typeof setInterval> | null = null // 音量フェード用のインターバル
let startTime = 0                 // 曲の再生開始時刻（ms）
let remainingTime = 0             // 現在の曲の残り再生時間（ms）

// 現在の曲のサムネイルURLを計算
const currentThumb = computed(() => `https://i.ytimg.com/vi/${playlist[currentIndex.value]?.id}/maxresdefault.jpg`)

/**
 * すべての実行中タイマー（切り替え用、フェード用）をクリアする
 */
const clearAllTimers = () => {
  if (timer) clearTimeout(timer)
  if (fadeInterval) clearInterval(fadeInterval)
}

/**
 * YouTube IFrame APIの初期化
 */
const initPlayer = () => {
  player = new (window as any).YT.Player('hidden-player', {
    height: '0',
    width: '0',
    playerVars: { 'autoplay': 0, 'controls': 0, 'rel': 0, 'playsinline': 1 },
    events: {
      'onReady': () => { isReady.value = true },
      'onStateChange': onPlayerStateChange
    }
  })
}

/**
 * 次の曲へインデックスを進めて再生する
 */
const switchNext = () => {
  if (isPaused.value) return
  currentIndex.value = (currentIndex.value + 1) % playlist.length
  playSong()
}

/**
 * 音量を滑らかに変更する（フェードイン/アウト）
 * @param target 目標音量 (0-100)
 * @param ms フェードにかける時間 (ミリ秒)
 * @param callback 完了時に実行する関数
 */
const fade = (target: number, ms: number, callback?: () => void) => {
  if (fadeInterval) clearInterval(fadeInterval)
  
  const startVol = player?.getVolume ? player.getVolume() : 0
  const step = (target - startVol) / 10 // 10分割して音量を変化させる
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

/**
 * 曲の終了タイミングを監視するタイマーを開始
 * 終了1秒前からフェードアウトを開始する
 */
const startTimers = () => {
  startTime = Date.now()
  if (timer) clearTimeout(timer)
  
  // 指定のdurationが経過する1秒前にフェードアウト処理を実行
  timer = setTimeout(() => {
    fade(0, 1000, () => switchNext())
  }, remainingTime - 1000)
}

/**
 * プレイヤーの状態が変化した時のイベントハンドラ
 * 再生開始(PLAYING)を検知してフェードインとタイマーを開始する
 */
const onPlayerStateChange = (event: any) => {
  if (event.data === (window as any).YT.PlayerState.PLAYING) {
    if (!isPaused.value) {
      fade(100, 1000) // 1秒かけてフェードイン
      startTimers()
    }
    isProcessing.value = false
  }
}

/**
 * 現在のインデックスに基づき動画を読み込んで再生する
 */
const playSong = () => {
  clearAllTimers()
  const song = playlist[currentIndex.value]
  if (!song) return

  remainingTime = song.duration * 1000
  
  if (player && player.loadVideoById) {
    player.setVolume(0) // 音量を0にしてからロード開始
    player.unMute()
    player.loadVideoById({ 
      videoId: song.id, 
      startSeconds: song.start 
    })
  }
}

/**
 * メドレーの最初の再生を開始する
 */
const startMedley = () => {
  if (isProcessing.value) return
  isStarted.value = true
  playSong()
}

/**
 * 再生と一時停止を切り替える
 * 一時停止時は経過時間を計算し、再開時にその残り時間からタイマーをセットする
 */
const togglePause = () => {
  if (isProcessing.value) return
  isProcessing.value = true

  if (isPaused.value) {
    // 再生再開
    isPaused.value = false
    player.playVideo()
    fade(100, 500)
  } else {
    // 一時停止
    isPaused.value = true
    clearAllTimers()
    // 停止までの経過時間を計算し、残り時間を更新
    remainingTime -= (Date.now() - startTime)
    
    fade(0, 500, () => {
      player.pauseVideo()
      isProcessing.value = false
    })
  }
}

// マウント時にYouTube APIスクリプトを読み込み
onMounted(() => {
  if (!(window as any).YT) {
    const tag = document.createElement('script')
    tag.src = "https://www.youtube.com/iframe_api"
    document.head.appendChild(tag)
  }
  // APIの準備ができたらプレイヤーを初期化
  (window as any).onYouTubeIframeAPIReady = () => initPlayer()
})

// コンポーネント破棄時にタイマーのクリアとプレイヤーの破棄を行う
onBeforeUnmount(() => {
  clearAllTimers()
  if (player) player.destroy()
})
</script>

<template>
  <div class="medley-wrapper">
    <div class="monitor">
      <div class="card">
        <div id="hidden-player"></div>
        <div class="jacket-area">
          <img :src="currentThumb" class="jacket-img" :class="{ 'is-playing': isStarted && !isPaused }" />
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
            <div class="progress-bar-fill" :style="{ animationDuration: playlist[currentIndex]?.duration + 's' }" :class="{ paused: isPaused }" :key="currentIndex"></div>
          </div>
          <button @click="togglePause" :disabled="isProcessing" class="ctrl-btn">
            {{ isPaused ? 'RESUME' : 'PAUSE' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.medley-wrapper {
  background: #121212; min-height: 100vh; display: flex; justify-content: center; align-items: center; font-family: 'Avenir', sans-serif;
}
.monitor { width: 400px; }
.card { background: #1e1e1e; border-radius: 16px; overflow: hidden; box-shadow: 0 20px 40px rgba(0,0,0,0.5); }
#hidden-player { position: absolute; left: -9999px; top: -9999px; }

.jacket-area { position: relative; width: 400px; height: 400px; background: #000; }
.jacket-img { width: 100%; height: 100%; object-fit: cover; opacity: 0.5; transition: 0.5s; }
.jacket-img.is-playing { opacity: 1; }

.start-overlay { position: absolute; inset: 0; display: flex; justify-content: center; align-items: center; }
.play-btn { background: #f06; color: #fff; border: none; padding: 15px 30px; border-radius: 30px; font-weight: bold; cursor: pointer; }
.play-btn:disabled { background: #444; opacity: 0.7; cursor: not-allowed; }

.info-area { padding: 30px; text-align: center; }
.now-playing { font-size: 12px; color: #f06; letter-spacing: 2px; margin: 0; }
.title-text { font-size: 24px; margin: 10px 0 20px; color: #fff; }

.progress-bar-bg { width: 100%; height: 4px; background: #333; border-radius: 2px; margin-bottom: 20px; overflow: hidden; }
.progress-bar-fill { height: 100%; background: #f06; width: 0; animation: fill linear forwards; }
.progress-bar-fill.paused { animation-play-state: paused; }

.ctrl-btn { background: transparent; border: 1px solid #444; color: #ccc; padding: 8px 20px; border-radius: 20px; cursor: pointer; }
.ctrl-btn:disabled { opacity: 0.5; cursor: wait; }

@keyframes fill { from { width: 0%; } to { width: 100%; } }
</style>