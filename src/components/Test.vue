<template>
  <div class="test-page">
    <header class="main-header">
      <h1>👁️ Eye-Tracking</h1>
    </header>

    <div class="heatmap-area" ref="trackingArea">
      <div class="heatmap-overlay" ref="heatmapContainer"></div>
    </div>

    <div class="buttons">
      <button @click="toggleTracking">{{ isTracking ? 'Stop Tracking' : 'Start Tracking' }}</button>
      <button @click="clearHeatmap" :disabled="!heatmapInstance">Heatmap leeren</button>
      <button
          @click="exportHeatmap"
          :disabled="!heatmapInstance || allPoints.length === 0"
      >
        Export
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
const h337 = window.h337
const allHeatmapPoints = ref([])
const trackingArea = ref(null)
const heatmapContainer = ref(null)
import html2canvas from 'html2canvas'
let heatmapInstance = null
const isTracking = ref(false)

const allPoints = ref([])

function initHeatmap() {
  const width = 800
  const height = 600

  const container = heatmapContainer.value
  container.style.width = width + 'px'
  container.style.height = height + 'px'

  heatmapInstance = h337.create({
    container,
    radius: 30,
    maxOpacity: 0.6,
    minOpacity: 0.1,
    blur: 0.85,
    renderer: 'canvas'
  })

  console.log('🔥 Heatmap initialisiert (Debug-Modus)')
}
function toggleTracking() {
  if (isTracking.value) {
    stopEyeTracking()
  } else {
    startEyeTracking()
  }
}

function startEyeTracking() {
  if (!window.webgazer) {
    console.error('❌ WebGazer nicht geladen')
    return
  }

  isTracking.value = true
  window.webgazer
      .setGazeListener((data, timestamp) => {
        if (!data || !isTracking) return

        const area = trackingArea.value.getBoundingClientRect()
        const x = data.x - area.left
        const y = data.y - area.top

        const innerhalb = x >= 0 && y >= 0 && x <= area.width && y <= area.height

        if (innerhalb) {
          const punkt = { x, y, value: 1, timestamp: Date.now() }
          heatmapInstance.addData(punkt)
          allPoints.value.push(punkt)
        }
      })
      .begin()
      .showVideoPreview(false)
      .showFaceOverlay(false)
      .showFaceFeedbackBox(false)

  console.log('🟢 Eye-Tracking gestartet')
}

function stopEyeTracking() {
  isTracking.value = false
  if (window.webgazer) {
    window.webgazer.pause()
    console.log('🛑 Eye-Tracking gestoppt')
    console.log('🔴 Punkte beim Stop:', allHeatmapPoints.value)
    console.log('🔥 Sichtbare Heatmap vor update:', heatmapInstance.getData())
  }

  // Zeige alle bisher gespeicherten Punkte
  if (heatmapInstance && allHeatmapPoints?.value?.length > 0) {
    heatmapInstance.setData({
      max: 1,
      data: allHeatmapPoints.value
    });
  }
}

const isCalibrating = ref(false)
const calibrationIndex = ref(0)

// 9 Punkte für die Kalibrierung (Positionen anpassen bei Bedarf)
const calibrationPoints = [
  { x: 50, y: 50 },
  { x: 400, y: 50 },
  { x: 750, y: 50 },
  { x: 50, y: 300 },
  { x: 400, y: 300 },
  { x: 750, y: 300 },
  { x: 50, y: 550 },
  { x: 400, y: 550 },
  { x: 750, y: 550 }
]

// Klick auf Kalibrierungspunkt
function handleCalibrationClick(index) {
  calibrationIndex.value++

  if (calibrationIndex.value >= calibrationPoints.length) {
    isCalibrating.value = false
    webgazer.clearData()
    startEyeTracking()
    console.log('✅ Kalibrierung abgeschlossen')
  }
}

function clearHeatmap() {
  allPoints.value = []
  heatmapInstance.setData({ max: 1, data: [] })
  console.log('🧹 Heatmap geleert')
}

function exportHeatmap() {
  if (!trackingArea.value) return

  html2canvas(trackingArea.value).then(canvas => {
    const link = document.createElement('a')
    link.href = canvas.toDataURL('image/png')
    link.download = 'heatmap-export.png'
    link.click()
    console.log('📦 Heatmap exportiert')
  })
}

onMounted(() => {
  initHeatmap()
})
</script>

<style scoped>
.test-page {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 2rem;
}
.main-header {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 50px;
  background-color: #2c3e50;
  color: white;
  font-size: 0.6rem;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.heatmap-area {
  width: 800px;
  height: 600px;
  position: relative;
  border: 2px dashed red;
  margin-bottom: 1.5rem;
}

.heatmap-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  background-color: rgba(0, 255, 0, 0.02);
}

.buttons {
  display: flex;
  gap: 1rem;
}
</style>