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
      <div class="heatmap-overlay" ref="heatmapContainer"></div>
    </div>


    <div class="buttons">
      <select v-model="selectedImage" class="image-select">
        <option value="">Kein Bild</option>
        <option value="/City.png">Testbild</option>
        <option value="/Zielscheibe.png">Zielscheibe</option>
        <!-- Weitere Bilder hier -->
      </select>
      <button @click="toggleTracking">{{ isTracking ? 'Stop Tracking' : 'Start Tracking' }}</button>
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
const h337 = window.h337
const allHeatmapPoints = ref([])
const trackingArea = ref(null)
const heatmapContainer = ref(null)
import html2canvas from 'html2canvas'
let heatmapInstance = null
const isTracking = ref(false)
const allPoints = ref([])
const selectedImage = ref('')

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
      .setRegression('ridge')
      //.showVideoPreview(false)
      //.showFaceOverlay(false)
      .showFaceFeedbackBox(false)
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

function clearHeatmap() {
  allPoints.value = []
  heatmapInstance.setData({ max: 1, data: [] })
  console.log('🧹 Heatmap geleert')
}

const exportFormat = ref("")

function handleExport() {
  if (exportFormat.value === "png") {
    exportAsPNG()
  } else if (exportFormat.value === "json") {
    exportAsJSON()
  } else if (exportFormat.value === "xlsx") {
    exportAsExcel()
  }

  // Zurücksetzen, damit erneut ausgewählt werden kann
  exportFormat.value = ""
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
  const json = JSON.stringify(allPoints.value, null, 2)
  const blob = new Blob([json], { type: "application/json" })
  const url = URL.createObjectURL(blob)

  const link = document.createElement("a")
  link.href = url
  link.download = "blickdaten.json"
  link.click()

  console.log("📄 JSON exportiert")
}
function exportAsExcel() {
  const data = allPoints.value.map(p => ({
    x: p.x,
    y: p.y,
    timestamp: new Date(p.timestamp).toISOString()
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

  console.log('📊 Excel-Datei exportiert')
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
  box-sizing: border-box;
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
  margin-top: 10px;
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
.image-select {
  margin: 1rem 0;
  padding: 0.4rem;
  font-size: 1rem;
}

.buttons {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 1rem;
  padding-bottom: 2rem;
}
</style>