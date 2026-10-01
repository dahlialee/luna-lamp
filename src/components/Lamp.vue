<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount, watch } from 'vue'

/* ---------- Đồng hồ + ngày ---------- */
const now = ref(new Date())
const pad = (n) => String(n).padStart(2, '0')
const clock = computed(() => `${pad(now.value.getHours())}:${pad(now.value.getMinutes())}`)
const seconds = computed(() => pad(now.value.getSeconds()))
const dateText = computed(() =>
  now.value.toLocaleDateString('vi-VN', { weekday: 'long', day: '2-digit', month: '2-digit', year: 'numeric' })
)
const locationText = ref('Đang xác định vị trí...')

function updateLocation() {
  if (!navigator.geolocation) {
    locationText.value = 'Trình duyệt không hỗ trợ định vị'
    return
  }

  navigator.geolocation.getCurrentPosition(
    async ({ coords }) => {
      try {
        const params = new URLSearchParams({
          format: 'jsonv2',
          lat: String(coords.latitude),
          lon: String(coords.longitude),
          zoom: '14',
          addressdetails: '1',
          'accept-language': 'vi',
        })
        const response = await fetch(`https://nominatim.openstreetmap.org/reverse?${params}`)
        if (!response.ok) throw new Error('Reverse geocoding failed')

        const { address } = await response.json()
        const district = address.city_district || address.district || address.county || address.borough || address.suburb
        const city = address.city || address.town || address.municipality || address.village || address.state_district || address.state
        locationText.value = [district, city].filter((part, index, parts) => part && parts.indexOf(part) === index).join(', ')
          || 'Không tìm thấy tên khu vực'
      } catch {
        locationText.value = 'Không thể tra cứu địa chỉ'
      }
    },
    () => { locationText.value = 'Không thể xác định vị trí' },
    { enableHighAccuracy: false, timeout: 10000, maximumAge: 300000 },
  )
}

/* ---------- Độ sáng / độ ấm (lưu localStorage) ---------- */
const saved = JSON.parse(localStorage.getItem('lamp') || '{}')
const brightness = ref(saved.brightness ?? 80) // 5..100
const warmth = ref(saved.warmth ?? 70)         // 0 = trắng lạnh, 100 = cam ấm
watch([brightness, warmth], () => localStorage.setItem('lamp', JSON.stringify({ brightness: brightness.value, warmth: warmth.value })))

const lerp = (a, b, t) => Math.round(a + (b - a) * t)
const mix = (cool, warm, t) => cool.map((c, i) => lerp(c, warm[i], t))
const rgb = (c, a = 1) => `rgba(${c[0]},${c[1]},${c[2]},${a})`

const lampStyle = computed(() => {
  const t = warmth.value / 100
  const core = mix([255, 252, 246], [255, 214, 150], t)  // tâm
  const edge = mix([238, 244, 255], [255, 160, 70], t)   // rìa
  const glow = mix([200, 220, 255], [255, 120, 30], t)   // quầng sáng
  const b = brightness.value / 100
  return {
    background: `radial-gradient(ellipse at 50% 45%, ${rgb(core)} 0%, ${rgb(edge)} 78%, ${rgb(glow)} 100%)`,
    boxShadow: `0 0 ${20 + 60 * b}px ${6 + 22 * b}px ${rgb(glow, 0.15 + 0.5 * b)}, inset 0 0 40px 6px ${rgb(glow, 0.55)}`,
    filter: `brightness(${0.12 + 0.88 * b})`,
    '--ink': rgb(mix([70, 80, 100], [168, 72, 28], t), 0.9),
  }
})

/* ---------- Pomodoro ---------- */
const MODES = {
  focus: { label: 'Tập trung', min: 25 },
  short: { label: 'Nghỉ ngắn', min: 5 },
  long: { label: 'Nghỉ dài', min: 15 },
}
const mode = ref('focus')
const remaining = ref(MODES.focus.min * 60) // giây
const running = ref(false)
const done = reactive({ count: 0 })
let endAt = 0
let tick = null

const pomoText = computed(() => `${pad(Math.floor(remaining.value / 60))}:${pad(remaining.value % 60)}`)

function beep() {
  try {
    const ctx = new (window.AudioContext || window.webkitAudioContext)()
    ;[0, 0.25, 0.5].forEach((d) => {
      const o = ctx.createOscillator(), g = ctx.createGain()
      o.frequency.value = 660; g.gain.value = 0.08
      o.connect(g).connect(ctx.destination)
      o.start(ctx.currentTime + d); o.stop(ctx.currentTime + d + 0.15)
    })
  } catch {}
}

function setMode(m) {
  mode.value = m
  running.value = false
  remaining.value = MODES[m].min * 60
}
function toggle() {
  if (running.value) { running.value = false; return }
  endAt = Date.now() + remaining.value * 1000
  running.value = true
}
function reset() { setMode(mode.value) }

function onTick() {
  now.value = new Date()
  if (!running.value) return
  remaining.value = Math.max(0, Math.round((endAt - Date.now()) / 1000))
  if (remaining.value === 0) {
    running.value = false
    beep()
    if (mode.value === 'focus') {
      done.count++
      setMode(done.count % 4 === 0 ? 'long' : 'short')
    } else {
      setMode('focus')
    }
  }
}

watch([pomoText, running], () => {
  document.title = running.value ? `${pomoText.value} · ${MODES[mode.value].label}` : 'Đèn đồng hồ'
})

/* ---------- Bảng điều khiển tự ẩn khi không dùng ---------- */
const panelVisible = ref(false)
let hideTimer = null
function showPanel() {
  panelVisible.value = true
  clearTimeout(hideTimer)
}
function hidePanelLater() {
  clearTimeout(hideTimer)
  hideTimer = setTimeout(() => (panelVisible.value = false), 4000)
}
function wake() {
  showPanel()
  hidePanelLater()
}

onMounted(() => {
  tick = setInterval(onTick, 250)
  updateLocation()
})
onBeforeUnmount(() => { clearInterval(tick); clearTimeout(hideTimer) })
</script>

<template>
  <main class="stage" @keydown="wake">
    <section class="lamp" :style="lampStyle" aria-label="Đồng hồ" @mouseenter="showPanel" @mouseleave="hidePanelLater" @touchstart="wake">
      <p class="date">{{ dateText }}</p>
      <p class="location">
        {{ locationText }}
        <a class="location-source" href="https://www.openstreetmap.org/copyright" target="_blank" rel="noreferrer"></a>
      </p>
      <p class="time">{{ clock }}<span class="sec">{{ seconds }}</span></p>

      <div class="pomo">
        <span class="mode">{{ MODES[mode].label }}</span>
        <span class="count" :class="{ idle: !running }">{{ pomoText }}</span>
      </div>
    </section>

    <section class="panel" :class="{ show: panelVisible }" @mouseenter="showPanel" @mouseleave="hidePanelLater" @touchstart="wake" @focusin="showPanel">
      <div class="row buttons">
        <button v-for="(m, key) in MODES" :key="key" :class="{ on: mode === key }" @click="setMode(key)">{{ m.label }}</button>
        <span class="sep" />
        <button class="primary" @click="toggle">{{ running ? 'Tạm dừng' : 'Bắt đầu' }}</button>
        <button @click="reset">Đặt lại</button>
      </div>
      <div class="row sliders">
        <label>Độ sáng
          <input type="range" min="5" max="100" v-model.number="brightness" />
        </label>
        <label>Độ ấm
          <input type="range" min="0" max="100" v-model.number="warmth" class="warm" />
        </label>
      </div>
    </section>
  </main>
</template>

<style>
html, body, #app { margin: 0; height: 100%; width: 100%; background: #000; }
</style>

<style scoped>
.stage {
  min-height: 100vh;
  background: #000;
  display: grid;
  grid-template-rows: 1fr auto;
  justify-items: center;
  align-items: center;
  padding: 4vh 4vw 3vh;
  box-sizing: border-box;
  font-family: 'Inter', system-ui, -apple-system, 'Segoe UI', sans-serif;
}

/* khung oval (dạng viên thuốc) */
.lamp {
  width: min(88vw, 880px);
  aspect-ratio: 1.75 / 1;
  border-radius: 999px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: clamp(4px, 1.4vw, 14px);
  color: var(--ink);
  transition: box-shadow .4s, background .4s, filter .25s;
  user-select: none;
}
.date { margin: 0; font-size: clamp(12px, 2vw, 20px); letter-spacing: .02em; text-transform: capitalize; }
.location { margin: 0; font-size: clamp(11px, 1.6vw, 16px); opacity: .75; }
.location-source { margin-left: 6px; color: inherit; font-size: .75em; opacity: .75; }
.time { margin: 0; font-size: clamp(56px, 13vw, 140px); font-weight: 300; line-height: 1; font-variant-numeric: tabular-nums; }
.sec { font-size: .28em; margin-left: .25em; opacity: .65; vertical-align: top; position: relative; top: .35em; }

.pomo { display: flex; align-items: baseline; gap: 12px; margin-top: clamp(4px, 1.5vw, 16px); }
.mode { font-size: clamp(11px, 1.6vw, 16px); opacity: .75; }
.count { font-size: clamp(20px, 3.6vw, 36px); font-variant-numeric: tabular-nums; }
.count.idle { opacity: .7; }

/* bảng điều khiển */
.panel {
  width: min(88vw, 880px);
  display: grid;
  gap: 14px;
  color: #b9a48f;
  opacity: 0;
  transform: translateY(6px);
  pointer-events: none;
  transition: opacity .4s, transform .4s;
}
.panel.show, .panel:focus-within { opacity: 1; transform: none; pointer-events: auto; }
.row { display: flex; flex-wrap: wrap; gap: 10px; align-items: center; justify-content: center; }
.sep { width: 1px; height: 22px; background: #3a3a3a; margin: 0 6px; }

button {
  background: transparent; color: inherit; border: 1px solid #3a3a3a; border-radius: 999px;
  padding: 7px 16px; font: inherit; font-size: 14px; cursor: pointer; transition: background .2s, color .2s, border-color .2s;
}
button:hover { border-color: #8a6a4a; }
button.on { border-color: #d98a45; color: #f0b27a; }
button.primary { background: #d98a45; border-color: #d98a45; color: #1a0f05; }
button:focus-visible, input:focus-visible { outline: 2px solid #f0b27a; outline-offset: 2px; }

.sliders { gap: 28px; }
label { display: flex; align-items: center; gap: 10px; font-size: 14px; }
input[type='range'] { width: clamp(120px, 28vw, 240px); accent-color: #d98a45; }
input.warm { accent-color: #ff9a4a; }

@media (prefers-reduced-motion: reduce) {
  .lamp, .panel, button { transition: none; }
}
</style>