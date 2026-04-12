<script setup lang="ts">
// --- 型定義 ---
interface Song {
  id: string
  tag: string // "7, 9" のようなカンマ区切りを想定
  start: number
  duration: number
  title: string
}

interface TagInfo {
  tag: string
  name: string
}

// --- ステート ---
const playlist = ref<Song[]>([])
const tagList = ref<TagInfo[]>([])
const userSelection = ref<Song[]>([])

const isEditMode = ref(true)
const isStarted = ref(false)
const isPaused = ref(false)
const isReady = ref(false)
const isProcessing = ref(false)

const currentIndex = ref(0)
const volume = ref(50) // 初期音量を50に設定
const activeTag = ref<string>('ALL') // フィルタリング用
const shareUrl = ref('')

// 内部変数
let player: any = null
let timer: ReturnType<typeof setTimeout> | null = null
let fadeInterval: ReturnType<typeof setInterval> | null = null
let startTime = 0
let remainingTime = 0

// タグIDから名前を引くマップ
const tagMap = computed(() => {
  return tagList.value.reduce((acc, t) => {
    acc[t.tag] = t.name
    return acc
  }, {} as Record<string, string>)
})

// フィルター用タグリスト
const availableTags = computed(() => [
  { tag: 'ALL', name: 'すべて' },
  ...tagList.value
])

// 表示用のフィルタリング済みリスト（複数タグ対応）
const filteredPlaylist = computed(() => {
  if (activeTag.value === 'ALL') return playlist.value
  return playlist.value.filter(s => 
    s.tag.split(',').map(t => t.trim()).includes(activeTag.value)
  )
})

// 再生に使用するリスト
const activePlaylist = computed(() => 
  userSelection.value.length > 0 ? userSelection.value : playlist.value
)

const currentThumb = computed(() => {
  const song = activePlaylist.value[currentIndex.value]
  return song ? `https://i.ytimg.com/vi/${song.id}/maxresdefault.jpg` : ''
})

// --- データ取得 ---
const fetchData = async () => {
  try {
    const [resPlaylist, resTags] = await Promise.all([
      fetch('./playlist.json'),
      fetch('./taglist.json')
    ])
    if (!resPlaylist.ok || !resTags.ok) throw new Error('Data load failed')
    playlist.value = await resPlaylist.json()
    tagList.value = await resTags.json()
  } catch (e) {
    console.error('Error loading JSON:', e)
  }
}

// --- プレイヤー制御 ---

const clearAllTimers = () => { 
  if (timer) { clearTimeout(timer); timer = null }
  if (fadeInterval) { clearInterval(fadeInterval); fadeInterval = null }
}

const fade = (target: number, ms: number, callback?: () => void) => {
  if (fadeInterval) clearInterval(fadeInterval)
  
  const startVol = player?.getVolume ? player.getVolume() : 0
  const finalVol = target === 0 ? 0 : Number(volume.value)
  const step = (finalVol - startVol) / 10
  let count = 0
  
  fadeInterval = setInterval(() => {
    count++
    if (player?.setVolume) {
      player.setVolume(Math.max(0, Math.min(100, startVol + (step * count))))
    }
    if (count >= 10) { 
      clearInterval(fadeInterval!)
      fadeInterval = null
      if (callback) callback() 
    }
  }, ms / 10)
}

const startTimers = () => {
  startTime = Date.now()
  if (timer) clearTimeout(timer)
  timer = setTimeout(() => { 
    fade(0, 1000, () => switchNext()) 
  }, Math.max(0, remainingTime - 1000))
}

const playSong = () => {
  clearAllTimers()
  const song = activePlaylist.value[currentIndex.value]
  if (!song || !player?.loadVideoById) return
  
  remainingTime = song.duration * 1000
  player.loadVideoById({ 
    videoId: song.id, 
    startSeconds: song.start 
  })
}

const switchNext = () => {
  if (isPaused.value || activePlaylist.value.length === 0) return
  currentIndex.value = (currentIndex.value + 1) % activePlaylist.value.length
  playSong()
}

const onPlayerStateChange = (event: any) => {
  const YT = (window as any).YT
  if (event.data === YT.PlayerState.PLAYING) {
    if (!isPaused.value) {
      player.setVolume(0)
      fade(volume.value, 800) 
      startTimers()
    }
    isProcessing.value = false
  }
  if (event.data === YT.PlayerState.ENDED) {
    switchNext()
  }
}

const initPlayer = () => {
  if (player || !(window as any).YT) return 
  player = new (window as any).YT.Player('main-player', {
    height: '100%', 
    width: '100%',
    playerVars: { 
      'autoplay': 0, 'controls': 1, 'rel': 0, 
      'playsinline': 1, 'modestbranding': 0 
    },
    events: {
      'onReady': () => { isReady.value = true },
      'onStateChange': onPlayerStateChange,
      'onError': () => { isProcessing.value = false; switchNext() }
    }
  })
}

// --- 外部公開用メソッド ---

const handleVolumeChange = () => {
  if (player?.setVolume) {
    // ユーザーが操作したら強制的にフェードタイマーを殺す
    if (fadeInterval) {
      clearInterval(fadeInterval)
      fadeInterval = null
    }
    player.setVolume(Number(volume.value)) // 確実に数値として渡す
  }
}

const togglePause = () => {
  if (isProcessing.value || !player) return
  isProcessing.value = true
  
  if (isPaused.value) {
    isPaused.value = false
    player.playVideo()
    fade(volume.value, 500)
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

const stopAll = () => {
  clearAllTimers()
  if (player?.pauseVideo) player.pauseVideo()
  isStarted.value = false
  isPaused.value = false
}

// --- UI操作 ---

const getSelectedIndex = (song: Song) => userSelection.value.findIndex(s => s.id === song.id)

const toggleSongSelection = (song: Song) => {
  const idx = getSelectedIndex(song)
  if (idx > -1) {
    userSelection.value.splice(idx, 1)
  } else if (userSelection.value.length < 10) {
    userSelection.value.push(song)
  }
}

const clearSelection = () => { 
  if (confirm('リストをクリアしますか？')) userSelection.value = [] 
}

const confirmSelection = () => {
  if (userSelection.value.length === 0) return
  stopAll()
  isEditMode.value = false
  currentIndex.value = 0
  const ids = userSelection.value.map(s => s.id)
  shareUrl.value = `${window.location.origin}${window.location.pathname}?m=${btoa(JSON.stringify(ids))}`
}

const copyLink = () => {
  navigator.clipboard.writeText(shareUrl.value)
  alert('コピーしました！')
}

const loadFromParams = () => {
  const params = new URLSearchParams(window.location.search)
  const data = params.get('m')
  if (!data) return
  
  try {
    const ids = JSON.parse(atob(data)) as string[]
    const restored = ids
      .map(id => playlist.value.find(s => s.id === id))
      .filter((s): s is Song => !!s)
      
    if (restored.length > 0) { 
      userSelection.value = restored
      isEditMode.value = false 
    }
  } catch (e) { 
    console.error('URL parameter parse error:', e) 
  }
}

// --- ライフサイクル ---

onMounted(async () => {
  await fetchData()
  loadFromParams()
  
  if ((window as any).YT?.Player) {
    initPlayer()
  } else {
    const tag = document.createElement('script')
    tag.src = "https://www.youtube.com/iframe_api"
    document.head.appendChild(tag)
    ;(window as any).onYouTubeIframeAPIReady = () => initPlayer()
  }
})

onBeforeUnmount(() => { 
  stopAll()
  if (player?.destroy) player.destroy() 
})
</script>

<template>
  <div class="app-container">
    
    <div class="medley-wrapper">
      <div class="bg-blur" :style="{ backgroundImage: `url(${currentThumb})` }"></div>

      <div class="monitor">
        <div v-show="isEditMode" class="card edit-card">
          <div class="header">
            <div class="header-top">
              <h3 class="main-title">最強のプラチナガシャ画面を作ろう</h3>
            </div>
            <div class="selection-status">
              <span>MY LINEUP ({{ userSelection.length }}/10)</span>
              <button v-show="userSelection.length > 0" @click="clearSelection" class="clear-btn">CLEAR</button>
            </div>
            <div class="tag-filter-container">
              <button 
                v-for="t in availableTags" 
                :key="t.tag" 
                @click="activeTag = t.tag"
                :class="{ 'active': activeTag === t.tag }"
                class="tag-btn"
              >
                {{ t.name }}
              </button>
            </div>
          </div>

          <div class="song-list-scroll">
            <div 
              v-for="song in filteredPlaylist" 
              :key="song.id" 
              class="song-item"
              :class="{ 'is-selected': getSelectedIndex(song) > -1 }"
              @click="toggleSongSelection(song)"
            >
              <div class="selection-order">
                <span v-if="getSelectedIndex(song) > -1">{{ getSelectedIndex(song) + 1 }}</span>
              </div>
              <img :src="`https://i.ytimg.com/vi/${song.id}/default.jpg`" class="mini-thumb" />
              <div class="song-info">
                <span class="song-title">{{ song.title }}</span>
                <div class="tag-badges">
                  <span 
                    v-for="tId in song.tag.split(',').map(s => s.trim())" 
                    :key="tId" 
                    class="song-tag-badge"
                  >
                    {{ tagMap[tId] || tId }}
                  </span>
                </div>
              </div>
            </div>
          </div>

          <div class="footer">
            <button @click="confirmSelection" class="confirm-btn" :disabled="userSelection.length === 0">
              ラインナップを確定させる
            </button>
          </div>
        </div>

        <div v-show="!isEditMode" class="card play-card">
          <div class="video-section">
            <div id="main-player"></div>
          </div>

          <div class="info-area">
            <div v-if="!isStarted" class="setup-panel">
              <button @click="isStarted = true; playSong()" :disabled="!isReady || isProcessing" class="play-btn">
                {{ isReady ? 'DEMO START' : 'LOADING...' }}
              </button>
            </div>

            <div v-else class="playing-panel">
              <p class="now-playing">NOW PLAYING ({{currentIndex + 1}}/{{activePlaylist.length}})</p>
              <h2 class="title-text">{{ activePlaylist[currentIndex]?.title }}</h2>
              
              <div class="progress-bar-bg">
                <div class="progress-bar-fill" :style="{ animationDuration: activePlaylist[currentIndex]?.duration + 's' }" :class="{ paused: isPaused }" :key="currentIndex"></div>
              </div>

              <div class="controls">
                <div class="volume-control">
                  <span class="volume-icon">Vol</span>
                  <input 
                    type="range" 
                    v-model="volume" 
                    min="0" 
                    max="100" 
                    @input="handleVolumeChange"
                    class="volume-slider"
                  />
                  <span class="volume-num">{{ volume }}</span>
                </div>
                <button @click="togglePause" :disabled="isProcessing" class="ctrl-btn">
                  {{ isPaused ? 'RESUME' : 'PAUSE' }}
                </button>
                <button @click="isEditMode = true; stopAll()" class="ctrl-btn">EDIT</button>
                
                <a :href="`https://www.youtube.com/watch?v=${activePlaylist[currentIndex]?.id}`" target="_blank" class="ctrl-btn original-link">
                  ORIGINAL
                </a>

                <button @click="copyLink" class="ctrl-btn share">SHARE</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <footer class="disclaimer-outer">
      <p>本サイトは個人による非公式ファンサイトです。</p>
      <p>動画はYouTube公式チャンネルのコンテンツをAPIにより引用しており、本サイトが所有するものではありません。権利元からの要請があった場合は速やかに公開を停止します。</p>
      <!-- <p>© Bandai Namco Entertainment Inc.</p> -->
    </footer>

  </div>
</template>

<style scoped>
/* コンテンツ全体を縦に並べるためのルート */
.app-container {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  background: #000;
}

.medley-wrapper { 
  flex: 1; /* 余ったスペースをすべて使い、中央にモニターを配置 */
  background: #000; 
  display: flex; 
  justify-content: center; 
  align-items: center; 
  font-family: sans-serif; 
  position: relative; 
  overflow: hidden; 
  color: #fff; 
}

.bg-blur { position: absolute; inset: -20px; background-size: cover; background-position: center; filter: blur(50px) brightness(0.2); z-index: 0; transition: background-image 1.2s ease; }
.monitor { width: 95%; max-width: 500px; z-index: 1; }
.card { background: rgba(15, 15, 15, 0.95); backdrop-filter: blur(20px); border-radius: 24px; overflow: hidden; box-shadow: 0 25px 50px rgba(0,0,0,0.8); border: 1px solid rgba(255,255,255,0.1); }

.edit-card { height: 85vh; display: flex; flex-direction: column; }
.header { padding: 20px; border-bottom: 1px solid rgba(255,255,255,0.05); }

.main-title { margin: 0 0 10px 0; font-size: 16px; color: #f06; text-shadow: 0 0 10px rgba(255, 0, 102, 0.5); font-weight: bold; text-align: center; letter-spacing: 0.5px; }
.selection-status { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; font-size: 12px; color: #888; letter-spacing: 1px; }

.clear-btn { background: transparent; border: 1px solid #444; color: #888; font-size: 10px; padding: 4px 10px; border-radius: 4px; cursor: pointer; transition: 0.2s; }
.clear-btn:hover { border-color: #f06; color: #f06; }

.tag-filter-container { display: flex; flex-wrap: wrap; gap: 8px; max-height: 120px; overflow-y: auto; padding-right: 4px; }
.tag-btn { background: rgba(255,255,255,0.05); border: 1px solid rgba(255,255,255,0.1); color: #ccc; padding: 6px 12px; border-radius: 15px; font-size: 11px; cursor: pointer; transition: 0.2s; }
.tag-btn.active { background: #f06; border-color: #f06; color: #fff; }

.song-list-scroll { flex: 1; overflow-y: auto; padding: 10px; }
.song-item { display: flex; align-items: center; padding: 10px; margin-bottom: 8px; border-radius: 12px; background: rgba(255,255,255,0.03); cursor: pointer; transition: 0.2s; border: 1px solid transparent; }
.song-item.is-selected { background: rgba(255, 0, 102, 0.15); border: 1px solid #f06; }

.selection-order { width: 22px; height: 22px; display: flex; align-items: center; justify-content: center; background: rgba(255,255,255,0.1); border-radius: 50%; margin-right: 12px; font-size: 11px; font-weight: bold; }
.is-selected .selection-order { background: #f06; color: #fff; }

.mini-thumb { width: 60px; height: 34px; object-fit: cover; border-radius: 4px; margin-right: 12px; }
.song-info { flex: 1; display: flex; flex-direction: column; overflow: hidden; }
.song-title { font-size: 13px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

.tag-badges { display: flex; flex-wrap: wrap; gap: 4px; margin-top: 4px; }
.song-tag-badge { font-size: 9px; color: #f06; font-weight: bold; border: 1px solid rgba(255, 0, 102, 0.3); padding: 1px 5px; border-radius: 4px; }

.video-section { width: 100%; aspect-ratio: 16 / 9; background: #000; }
.info-area { padding: 30px 20px; min-height: 200px; text-align: center; }
.play-btn { background: #f06; color: #fff; border: none; padding: 16px 40px; border-radius: 30px; font-weight: bold; cursor: pointer; box-shadow: 0 4px 15px rgba(255, 0, 102, 0.4); }
.now-playing { font-size: 10px; color: #f06; font-weight: bold; margin-bottom: 8px; }
.title-text { font-size: 16px; margin-bottom: 20px; }

.progress-bar-bg { width: 100%; height: 4px; background: rgba(255,255,255,0.1); border-radius: 2px; margin-bottom: 25px; overflow: hidden; }
.progress-bar-fill { height: 100%; background: #f06; width: 0; animation: fill linear forwards; }
.progress-bar-fill.paused { animation-play-state: paused; }

.controls { display: flex; gap: 8px; justify-content: center; flex-wrap: wrap; }
.ctrl-btn { 
  display: inline-flex; align-items: center; justify-content: center; text-decoration: none;
  background: rgba(255,255,255,0.05); border: 1px solid rgba(255,255,255,0.1); 
  color: #fff; padding: 10px 16px; border-radius: 20px; font-size: 12px; cursor: pointer; transition: 0.2s;
}
.ctrl-btn:hover { background: rgba(255, 255, 255, 0.1); }
.original-link { border-color: rgba(255, 0, 0, 0.5); color: #ff4444; }
.original-link:hover { background: rgba(255, 0, 0, 0.1); }
.share { border-color: #f06; color: #f06; }

.footer { padding: 15px; }
.confirm-btn { width: 100%; background: #f06; color: #fff; border: none; padding: 15px; border-radius: 30px; font-weight: bold; cursor: pointer; transition: 0.3s; box-shadow: 0 4px 15px rgba(255, 0, 102, 0.3); }
.confirm-btn:hover:not(:disabled) { transform: translateY(-2px); box-shadow: 0 6px 20px rgba(255, 0, 102, 0.5); }
.confirm-btn:disabled { background: #222; color: #444; box-shadow: none; }

/* medley-wrapperの外側の免責事項 */
.disclaimer-outer {
  background: #000;
  text-align: center;
  font-size: 10px;
  color: rgba(255, 255, 255, 0.4);
  line-height: 1.6;
  padding: 20px;
  border-top: 1px solid rgba(255, 255, 255, 0.1);
  position: relative;
  z-index: 10;
}

@keyframes fill { from { width: 0%; } to { width: 100%; } }

.volume-control {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  margin-bottom: 15px;
  padding: 0 20px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 10px;
}

.volume-icon {
  font-size: 10px;
  font-weight: bold;
  color: #f06;
}

.volume-slider {
  flex: 1;
  accent-color: #f06; /* スライダーの色をテーマカラーに */
  cursor: pointer;
}

.volume-num {
  font-size: 12px;
  min-width: 25px;
  font-family: monospace;
}
</style>