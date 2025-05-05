<template>
  <div class="test-page">
    <header class="main-header">
      <h1>👁️ Eye-Tracking</h1>
    </header>

    <div class="heatmap-area" ref="trackingArea">
      <img
          v-if="selectedImage"
          :src="selectedImage"
          class="background-image"
          alt="Tracking-Bild"
      />
      <div class="heatmap-overlay" ref="heatmapContainer">
        <!-- Kalibrierungspunkte -->
        <div v-if="isCalibrating" class="calibration-overlay">
          <div
              v-for="(point, index) in calibrationPoints"
              :key="index"
              class="calibration-point"
              :style="{ top: point.y + 'px', left: point.x + 'px', opacity: currentCalibrationStep === index ? 1 : 0.3 }"
              :class="{ clicked: index < currentCalibrationStep }"
              @click="handleCalibrationClick(index)"
          ></div>
        </div>
      </div>
    </div>

    <div class="buttons">
      <select v-model="selectedImage" class="image-select">
        <option value="">Kein Bild</option>
        <option value="/City.png">City</option>
        <option value="/Zielscheibe.png">Zielscheibe</option>
        <option value="/fantasy.png">Fantasy</option>
      </select>
      <button @click="toggleTracking">
        {{ isTracking ? 'Stop Tracking' : 'Start Tracking' }}
      </button>
      <button @click="clearHeatmap" :disabled="!heatmapInstance">Heatmap leeren</button>
      <div class="export-menu">
        <select v-model="exportFormat" @change="handleExport" :disabled="allPoints.length === 0">
          <option disabled value="">Exportieren als...</option>
          <option value="png">PNG (Screenshot)</option>
          <option value="json">JSON (Blickdaten)</option>
          <option value="xlsx">Excel (.xlsx)</option>
        </select>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import * as XLSX from 'xlsx'
import html2canvas from 'html2canvas'

const h337 = window.h337
const trackingArea = ref(null)
const heatmapContainer = ref(null)
let heatmapInstance = null

const isTracking = ref(false)
const allPoints = ref([])
const selectedImage = ref('')
const exportFormat = ref('')

// Kalibrierung
const isCalibrating = ref(false)
const calibrationPoints = ref([])
const currentCalibrationStep = ref(0)

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
    renderer: 'canvas',
  })

  console.log('🔥 Heatmap initialisiert')
}

function toggleTracking() {
  if (isTracking.value) {
    stopEyeTracking()
  } else {
    startWebGazerAndCalibrate()
  }
}

function startWebGazerAndCalibrate() {
  if (!window.webgazer) {
    console.error('❌ WebGazer nicht geladen')
    return
  }

  window.webgazer
      .setRegression('ridge')
      .showFaceFeedbackBox(false)
      .begin()
      .then(() => {
        console.log('📷 Kamera bereit, Kalibrierung startet')
        startCalibration()
      })
}

function startCalibration() {
  const area = trackingArea.value.getBoundingClientRect()
  const padding = 50
  const positions = [
    { x: padding, y: padding },
    { x: area.width / 2, y: padding },
    { x: area.width - padding, y: padding },
    { x: padding, y: area.height / 2 },
    { x: area.width / 2, y: area.height / 2 },
    { x: area.width - padding, y: area.height / 2 },
    { x: padding, y: area.height - padding },
    { x: area.width / 2, y: area.height - padding },
    { x: area.width - padding, y: area.height - padding },
  ]

  calibrationPoints.value = positions
  currentCalibrationStep.value = 0
  isCalibrating.value = true
}

function handleCalibrationClick(index) {
  if (index !== currentCalibrationStep.value) return
  currentCalibrationStep.value++

  if (currentCalibrationStep.value >= calibrationPoints.value.length) {
    isCalibrating.value = false
    startEyeTrackingAfterCalibration()
  }
}

function startEyeTrackingAfterCalibration() {
  isTracking.value = true

  window.webgazer.setGazeListener((data, timestamp) => {
    if (!data || !isTracking.value) return

    const area = trackingArea.value.getBoundingClientRect()
    const x = data.x - area.left
    const y = data.y - area.top

    if (x >= 0 && y >= 0 && x <= area.width && y <= area.height) {
      const punkt = { x, y, value: 1, timestamp: Date.now() }
      heatmapInstance.addData(punkt)
      allPoints.value.push(punkt)
    }
  })

  console.log('🟢 Eye-Tracking gestartet')
}

function stopEyeTracking() {
  isTracking.value = false

  if (window.webgazer) {
    window.webgazer.pause()
    console.log('🛑 Eye-Tracking gestoppt')
  }

  // Kein setData() aufrufen → Heatmap bleibt einfach sichtbar!
  console.log('ℹ️ Tracking gestoppt – Heatmap bleibt erhalten')
}

function clearHeatmap() {
  allPoints.value = []
  heatmapInstance.setData({ max: 1, data: [] })
  console.log('🧹 Heatmap geleert')
}

function handleExport() {
  if (exportFormat.value === 'png') {
    exportAsPNG()
  } else if (exportFormat.value === 'json') {
    exportAsJSON()
  } else if (exportFormat.value === 'xlsx') {
    exportAsExcel()
  }
  exportFormat.value = ''
}

function exportAsPNG() {
  if (!trackingArea.value) return

  html2canvas(trackingArea.value).then(canvas => {
    const link = document.createElement('a')
    link.href = canvas.toDataURL('image/png')
    link.download = 'heatmap-export.png'
    link.click()
    console.log('📦 PNG exportiert')
  })
}

function exportAsJSON() {
  const area = trackingArea.value.getBoundingClientRect()

  const data = allPoints.value.map(p => ({
    content: selectedImage.value || 'Kein Bild',
    x: p.x,
    y: p.y,
    x_norm: (p.x / area.width).toFixed(4),
    y_norm: (p.y / area.height).toFixed(4),
    timestamp: new Date(p.timestamp).toISOString(),
  }))

  const json = JSON.stringify(data, null, 2)
  const blob = new Blob([json], { type: "application/json" })
  const url = URL.createObjectURL(blob)

  const link = document.createElement("a")
  link.href = url
  link.download = "blickdaten.json"
  link.click()

  console.log("📄 JSON mit normierten Daten exportiert")
}

function exportAsExcel() {
  const area = trackingArea.value.getBoundingClientRect()

  const data = allPoints.value.map(p => ({
    content: selectedImage.value || 'Kein Bild',
    x: p.x,
    y: p.y,
    x_norm: (p.x / area.width).toFixed(4),
    y_norm: (p.y / area.height).toFixed(4),
    timestamp: new Date(p.timestamp).toISOString(),
  }))

  const worksheet = XLSX.utils.json_to_sheet(data)
  const workbook = XLSX.utils.book_new()
  XLSX.utils.book_append_sheet(workbook, worksheet, 'GazeData')

  const excelBuffer = XLSX.write(workbook, { bookType: 'xlsx', type: 'array' })
  const blob = new Blob([excelBuffer], { type: 'application/octet-stream' })

  const link = document.createElement('a')
  link.href = URL.createObjectURL(blob)
  link.download = 'blickdaten.xlsx'
  link.click()

  console.log('📊 Excel-Datei mit normierten Daten exportiert')
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
  padding: 1rem;
  box-sizing: border-box;
  width: 100%;
  max-width: 100vw;
  overflow-x: hidden;
}

.main-header {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 50px;
  background-color: #2c3e50;
  color: white;
  font-size: 0.7rem;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

/* Desktop-specific styles */
@media only screen and (min-width: 1025px) {
  .heatmap-area {
    width: 90%;
    max-width: 1200px;
    height: 75vh;
    min-height: 600px;
  }

  .background-image {
    object-fit: cover; /* Changed from contain to cover for better expansion */
  }

  .buttons {
    max-width: 1200px;
    gap: 1.5rem;
  }

  .buttons button {
    font-size: 1.1rem;
    padding: 0.8rem 1.5rem;
  }
}

/* Default styles (for tablets and below) */
.heatmap-area {
  width: 100%;
  max-width: 800px;
  height: 60vh;
  min-height: 400px;
  position: relative;
  border: 2px dashed red;
  margin-top: 60px;
  margin-bottom: 1.5rem;
  background-color: #fbfffb;
}

.heatmap-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 1;
}

.background-image {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: contain;
  z-index: 0;
}

.calibration-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 10;
  pointer-events: auto;
}

.calibration-point {
  position: absolute;
  width: 30px;
  height: 30px;
  background-color: rgba(255, 0, 0, 0.7);
  border-radius: 50%;
  transform: translate(-50%, -50%);
  cursor: pointer;
  transition: opacity 0.3s ease;
}
.calibration-point.clicked {
  background-color: rgba(0, 200, 0, 0.7); /* grün (nach Klick) */
  transform: translate(-50%, -50%) scale(1.1);
}
.image-select {
  margin: 1rem 0;
  padding: 0.5rem;
  font-size: 1rem;
  width: 90%;
  max-width: 300px;
}

.buttons {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 0.8rem;
  padding: 0.5rem;
  width: 100%;
  max-width: 800px;
}

.buttons button, .export-menu select {
  padding: 0.6rem 1rem;
  font-size: 0.9rem;
  min-width: 120px;
  border-radius: 8px;
  border: 1px solid #ccc;
}

.export-menu select {
  width: 100%;
  max-width: 200px;
}

/* iPad specific adjustments */
@media only screen and (min-width: 768px) and (max-width: 1024px) {
  .heatmap-area {
    height: 70vh;
    max-height: 700px;
  }

  .buttons {
    gap: 1.2rem;
  }

  .buttons button {
    font-size: 1rem;
    padding: 0.8rem 1.2rem;
  }
}

/* iPhone specific adjustments */
@media only screen and (max-width: 767px) {
  .heatmap-area {
    height: 55vh;
    min-height: 350px;
    margin-top: 55px;
  }

  .buttons {
    gap: 0.6rem;
    padding: 0.3rem;
  }

  .buttons button {
    font-size: 0.85rem;
    padding: 0.5rem 0.8rem;
    min-width: 100px;
  }

  .export-menu select {
    font-size: 0.85rem;
  }

  .calibration-point {
    width: 25px;
    height: 25px;
  }
}

/* Landscape orientation adjustments */
@media only screen and (max-width: 1024px) and (orientation: landscape) {
  .heatmap-area {
    height: 75vh;
    min-height: 300px;
  }

  .buttons {
    flex-direction: row;
    flex-wrap: nowrap;
    overflow-x: auto;
    padding-bottom: 0.5rem;
    justify-content: flex-start;
  }

  .buttons > * {
    flex-shrink: 0;
  }
}
</style>