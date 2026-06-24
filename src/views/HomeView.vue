<script setup>
import { ref, computed, onUnmounted } from 'vue'

// ── STT ──────────────────────────────────────────────────────────────────────
const sttSupported = computed(
  () => 'SpeechRecognition' in window || 'webkitSpeechRecognition' in window,
)
const sttTranscript = ref('')
const sttInterim = ref('')
const sttListening = ref(false)
const sttError = ref('')
let recognition = null

function startSTT() {
  const SR = window.SpeechRecognition || window.webkitSpeechRecognition
  recognition = new SR()
  recognition.lang = 'zh-TW'
  recognition.interimResults = true
  recognition.continuous = true

  recognition.onstart = () => {
    sttListening.value = true
    sttError.value = ''
    sttInterim.value = ''
  }
  recognition.onresult = (e) => {
    let final = '',
      interim = ''
    for (const r of e.results) {
      r.isFinal ? (final += r[0].transcript) : (interim += r[0].transcript)
    }
    if (final) sttTranscript.value += final
    sttInterim.value = interim
  }
  recognition.onerror = (e) => {
    sttError.value = e.error
  }
  recognition.onend = () => {
    sttListening.value = false
    sttInterim.value = ''
  }
  recognition.start()
}

function stopSTT() {
  recognition?.stop()
}

function clearSTT() {
  sttTranscript.value = ''
  sttInterim.value = ''
  sttError.value = ''
}

// ── TTS ──────────────────────────────────────────────────────────────────────
const ttsSupported = computed(() => 'speechSynthesis' in window)
const ttsText = ref('你好，這是語音合成測試。Hello, this is a TTS test.')
const ttsSpeaking = ref(false)
const ttsRate = ref(1)
const ttsPitch = ref(1)
const ttsVolume = ref(1)
const ttsVoices = ref([])
const ttsSelectedVoice = ref('')
const ttsError = ref('')

function loadVoices() {
  const voices = speechSynthesis.getVoices()
  ttsVoices.value = voices
  if (!ttsSelectedVoice.value && voices.length) {
    const preferred = voices.find((v) => v.lang.startsWith('zh')) || voices[0]
    ttsSelectedVoice.value = preferred.name
  }
}
if (ttsSupported.value) {
  loadVoices()
  speechSynthesis.onvoiceschanged = loadVoices
}

function speak() {
  if (!ttsText.value.trim()) return
  speechSynthesis.cancel()
  const utter = new SpeechSynthesisUtterance(ttsText.value)
  const voice = ttsVoices.value.find((v) => v.name === ttsSelectedVoice.value)
  if (voice) utter.voice = voice
  utter.rate = ttsRate.value
  utter.pitch = ttsPitch.value
  utter.volume = ttsVolume.value
  utter.onstart = () => {
    ttsSpeaking.value = true
    ttsError.value = ''
  }
  utter.onend = () => {
    ttsSpeaking.value = false
  }
  utter.onerror = (e) => {
    ttsError.value = e.error
    ttsSpeaking.value = false
  }
  speechSynthesis.speak(utter)
}

function stopTTS() {
  speechSynthesis.cancel()
  ttsSpeaking.value = false
}

onUnmounted(() => {
  recognition?.stop()
  speechSynthesis.cancel()
})
</script>

<template>
  <div class="app">
    <h1 class="title">🎙 Web Speech API 測試</h1>

    <!-- ── STT ───────────────────────────────────────── -->
    <section class="card">
      <h2 class="card-title">STT — 語音轉文字</h2>

      <div v-if="!sttSupported" class="badge badge--warn">⚠ 此瀏覽器不支援 SpeechRecognition</div>
      <template v-else>
        <div class="row">
          <button
            class="btn"
            :class="sttListening ? 'btn--danger' : 'btn--primary'"
            @click="sttListening ? stopSTT() : startSTT()"
          >
            {{ sttListening ? '⏹ 停止錄音' : '🎤 開始錄音' }}
          </button>
          <button class="btn btn--ghost" @click="clearSTT">清除</button>
          <span v-if="sttListening" class="badge badge--live">● REC</span>
        </div>

        <div class="transcript-box">
          <span class="final">{{ sttTranscript }}</span
          ><span class="interim">{{ sttInterim }}</span>
          <span v-if="!sttTranscript && !sttInterim" class="placeholder"
            >辨識結果將出現在這裡…</span
          >
        </div>

        <p v-if="sttError" class="error">❌ 錯誤：{{ sttError }}</p>

        <div class="hint">語言設定：<code>zh-TW</code>（可在程式碼中修改）</div>
      </template>
    </section>

    <!-- ── TTS ───────────────────────────────────────── -->
    <section class="card">
      <h2 class="card-title">TTS — 文字轉語音</h2>

      <div v-if="!ttsSupported" class="badge badge--warn">⚠ 此瀏覽器不支援 SpeechSynthesis</div>
      <template v-else>
        <textarea v-model="ttsText" class="textarea" rows="3" placeholder="輸入要朗讀的文字…" />

        <!-- Voice selector -->
        <label class="label">語音</label>
        <select v-model="ttsSelectedVoice" class="select">
          <option v-for="v in ttsVoices" :key="v.name" :value="v.name">
            {{ v.name }} ({{ v.lang }})
          </option>
        </select>

        <!-- Sliders -->
        <div class="sliders">
          <label class="label"
            >速率 <b>{{ ttsRate }}</b></label
          >
          <input
            type="range"
            v-model.number="ttsRate"
            min="0.5"
            max="2"
            step="0.1"
            class="slider"
          />

          <label class="label"
            >音調 <b>{{ ttsPitch }}</b></label
          >
          <input type="range" v-model.number="ttsPitch" min="0" max="2" step="0.1" class="slider" />

          <label class="label"
            >音量 <b>{{ ttsVolume }}</b></label
          >
          <input
            type="range"
            v-model.number="ttsVolume"
            min="0"
            max="1"
            step="0.05"
            class="slider"
          />
        </div>

        <div class="row">
          <button class="btn btn--primary" :disabled="ttsSpeaking" @click="speak">🔊 播放</button>
          <button class="btn btn--danger" :disabled="!ttsSpeaking" @click="stopTTS">⏹ 停止</button>
          <span v-if="ttsSpeaking" class="badge badge--live">● 播放中</span>
        </div>

        <p v-if="ttsError" class="error">❌ 錯誤：{{ ttsError }}</p>
      </template>
    </section>
  </div>
</template>

<style scoped>
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.app {
  min-height: 100vh;
  background: #0f1117;
  color: #e2e8f0;
  font-family: 'Inter', 'Noto Sans TC', system-ui, sans-serif;
  padding: 2rem 1rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.5rem;
}

.title {
  font-size: 1.75rem;
  font-weight: 700;
  letter-spacing: -0.02em;
  color: #fff;
}

.card {
  width: 100%;
  max-width: 600px;
  background: #1a1d27;
  border: 1px solid #2d3148;
  border-radius: 12px;
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.9rem;
}

.card-title {
  font-size: 1rem;
  font-weight: 600;
  color: #a78bfa;
  text-transform: uppercase;
  letter-spacing: 0.06em;
}

/* Buttons */
.row {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  flex-wrap: wrap;
}

.btn {
  padding: 0.45rem 1rem;
  border-radius: 6px;
  border: none;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition:
    opacity 0.15s,
    transform 0.1s;
}
.btn:active {
  transform: scale(0.97);
}
.btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}
.btn--primary {
  background: #7c3aed;
  color: #fff;
}
.btn--primary:hover:not(:disabled) {
  background: #6d28d9;
}
.btn--danger {
  background: #dc2626;
  color: #fff;
}
.btn--danger:hover:not(:disabled) {
  background: #b91c1c;
}
.btn--ghost {
  background: #2d3148;
  color: #e2e8f0;
}
.btn--ghost:hover {
  background: #374161;
}

/* Badges */
.badge {
  font-size: 0.75rem;
  font-weight: 700;
  padding: 0.2rem 0.6rem;
  border-radius: 20px;
}
.badge--live {
  background: #dc2626;
  color: #fff;
  animation: pulse 1s infinite;
}
.badge--warn {
  background: #92400e;
  color: #fde68a;
}
@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.6;
  }
}

/* Transcript */
.transcript-box {
  min-height: 80px;
  background: #0f1117;
  border: 1px solid #2d3148;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  font-size: 1rem;
  line-height: 1.6;
}
.final {
  color: #e2e8f0;
}
.interim {
  color: #94a3b8;
  font-style: italic;
}
.placeholder {
  color: #4a5270;
}

/* Forms */
.textarea {
  width: 100%;
  background: #0f1117;
  border: 1px solid #2d3148;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  color: #e2e8f0;
  font-size: 0.9rem;
  resize: vertical;
  font-family: inherit;
}
.textarea:focus {
  outline: 2px solid #7c3aed;
}

.select {
  width: 100%;
  background: #0f1117;
  border: 1px solid #2d3148;
  border-radius: 6px;
  color: #e2e8f0;
  padding: 0.45rem 0.75rem;
  font-size: 0.875rem;
}

.label {
  font-size: 0.8rem;
  color: #94a3b8;
}

.sliders {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}
.slider {
  width: 100%;
  accent-color: #7c3aed;
}

.hint {
  font-size: 0.78rem;
  color: #4a5270;
}
.error {
  font-size: 0.85rem;
  color: #f87171;
}
</style>
