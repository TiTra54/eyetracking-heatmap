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
          ref="uploadedImage"
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
      <label v-if="!selectedImage" class="custom-file-upload">
        📁 Bild auswählen
        <input
            type="file"
            accept="image/*"
            @change="handleImageUpload"
            hidden
        />
      </label>
      <button v-else @click="removeImage" class="custom-file-upload">
        🗑️ Bild löschen
      </button>
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
import { ref, onMounted, watch, nextTick } from 'vue'
import * as XLSX from 'xlsx'
import html2canvas from 'html2canvas'

// Heatmap-Objekt von heatmap.js
const h337 = window.h337

// Refs für DOM-Elemente
const trackingArea = ref(null)
const heatmapContainer = ref(null)
const uploadedImage = ref(null)

// Zustand des Trackings
const isTracking = ref(false)                  // Gibt an, ob das Eye-Tracking gerade läuft
const allPoints = ref([])                      // Gesammelte Blickpunkte (für Export etc.)
const selectedImage = ref('')                  // Aktuell ausgewähltes bzw. hochgeladenes Bild
const exportFormat = ref('')                   // Aktuelles Exportformat (png, json, xlsx)

// Kalibrierung: Vorbereitung und Ablaufsteuerung
const isCalibrating = ref(false)               // Gibt an, ob aktuell kalibriert wird
const calibrationPoints = ref([])              // Array mit allen Kalibrierungspunkten
const currentCalibrationStep = ref(0)          // Aktueller Schritt der Kalibrierung

let heatmapInstance = null                           // Heatmap-Instanz

// Initialisiert die Heatmap mit Standardgröße (z. B. wenn kein Bild aktiv ist)
function initHeatmap() {
  const width = 800
  const height = 600

  trackingArea.value.style.width = width + 'px'
  trackingArea.value.style.height = height + 'px'
  heatmapContainer.value.style.width = width + 'px'
  heatmapContainer.value.style.height = height + 'px'

  heatmapInstance = h337.create({
    container: heatmapContainer.value,
    radius: 30,
    maxOpacity: 0.6,
    minOpacity: 0.1,
    blur: 0.85,
    renderer: 'canvas',
  })

  console.log('Heatmap initialisiert mit Standardgröße')
}

// Liest Bilddatei ein und setzt sie als Background
function handleImageUpload(event) {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = e => {
    selectedImage.value = e.target.result
    console.log('Benutzerbild geladen')
  }
  reader.readAsDataURL(file)
}

// Reagiert auf neue Bildauswahl: Größe auslesen und Heatmap anpassen
watch(selectedImage, () => {
  if (!selectedImage.value) {
    initHeatmap()
    return
  }

  const img = uploadedImage.value
  if (!img) return

  img.onload = () => {
    const width = img.offsetWidth
    const height = img.offsetHeight
    trackingArea.value.style.width = width + 'px'
    trackingArea.value.style.height = height + 'px'
    heatmapContainer.value.style.width = width + 'px'
    heatmapContainer.value.style.height = height + 'px'
    heatmapInstance = h337.create({
      container: heatmapContainer.value,
      radius: 30,
      maxOpacity: 0.6,
      minOpacity: 0.1,
      blur: 0.85,
      renderer: 'canvas',
    })
    console.log(`Heatmap angepasst auf: ${width} x ${height}`)
  }

  if (img.complete) {
    img.onload()
  }
})

// Entfernt das aktuelle Bild und setzt Heatmap zurück
function removeImage() {
  selectedImage.value = ''
  allPoints.value = []
  if (heatmapInstance) {
    heatmapInstance.setData({ max: 1, data: [] })
  }
  console.log('Bild entfernt, Heatmap geleert')
}

// Startet oder stoppt das Tracking je nach Zustand
function toggleTracking() {
  if (isTracking.value) {
    stopEyeTracking()
  } else {
    startWebGazerAndCalibrate()
  }
}

// Initialisiert WebGazer und beginnt die Kalibrierung
function startWebGazerAndCalibrate() {
  if (!window.webgazer) {
    console.error('WebGazer nicht geladen')
    return
  }

  window.webgazer
      .setRegression('ridge')
      .showFaceFeedbackBox(false)
      .begin()
      .then(() => {
        console.log('Kamera bereit, Kalibrierung startet')
        startCalibration()
      })
}

// Generiert 9 Kalibrierungspunkte im Trackingbereich
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

// Verarbeitet Klick auf Kalibrierungspunkt
function handleCalibrationClick(index) {
  if (index !== currentCalibrationStep.value) return
  currentCalibrationStep.value++

  if (currentCalibrationStep.value >= calibrationPoints.value.length) {
    isCalibrating.value = false
    startEyeTrackingAfterCalibration()
  }
}

// Startet GazeListener und sammelt Blickdaten
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

  console.log('Eye-Tracking gestartet')
}

// Stoppt WebGazer, Tracking bleibt sichtbar
function stopEyeTracking() {
  isTracking.value = false
  if (window.webgazer) {
    window.webgazer.pause()
    console.log('Eye-Tracking gestoppt')
  }
}

// Leert Heatmap-Daten
function clearHeatmap() {
  allPoints.value = []
  heatmapInstance.setData({ max: 1, data: [] })
  console.log('Heatmap geleert')
}

// Export-Logik
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

// Exportiert sichtbaren Bereich als PNG
function exportAsPNG() {
  if (!trackingArea.value) return
  html2canvas(trackingArea.value).then(canvas => {
    const link = document.createElement('a')
    link.href = canvas.toDataURL('image/png')
    link.download = 'heatmap-export.png'
    link.click()
    console.log('PNG exportiert')
  })
}

// Exportiert Gaze-Daten als JSON mit normalisierten Koordinaten
function exportAsJSON() {
  const area = trackingArea.value.getBoundingClientRect()
  const data = allPoints.value.map(p => ({
    //content: selectedImage.value || 'Kein Bild',
    x: p.x,
    y: p.y,
    x_norm: (p.x / area.width).toFixed(4),
    y_norm: (p.y / area.height).toFixed(4),
    timestamp: new Date(p.timestamp).toISOString(),
  }))
  const json = JSON.stringify(data, null, 2)
  const blob = new Blob([json], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = 'blickdaten.json'
  link.click()
  console.log('JSON exportiert')
}

// Exportiert Gaze-Daten als Excel (.xlsx)
function exportAsExcel() {
  const area = trackingArea.value.getBoundingClientRect()
  const data = allPoints.value.map(p => ({
    //content: selectedImage.value || 'Kein Bild',
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
  console.log('Excel exportiert')
}

// Initialisiert Standard-Heatmap beim Laden der Seite
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

.heatmap-area {
  width: 800px;
  height: 600px;
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
  background-color: rgba(0, 200, 0, 0.7);
  transform: translate(-50%, -50%) scale(1.1);
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

.buttons button,
.export-menu select {
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

.custom-file-upload {
  display: inline-block;
  padding: 0.6rem 1.2rem;
  font-size: 0.95rem;
  cursor: pointer;
  background-color: #f0f0f0;
  border: 1px solid #ccc;
  border-radius: 8px;
  text-align: center;
  transition: background-color 0.2s;
}
.custom-file-upload:hover {
  background-color: #e0e0e0;
}
</style>