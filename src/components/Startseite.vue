<template>
    <div class="startseite">
        <header class="main-header">
            <h1>👁️ Eye-Tracking</h1>
        </header>

        <div class="controls">
          <label for="trackingMode">Tracking-Modus:</label>
          <select id="trackingMode" v-model="trackingMode">
            <option value="mouse">Maus-Tracking</option>
            <option value="eye">Eye-Tracking</option>
          </select>
        </div>
        <div class="controls">
          <label for="imageSelect">Bild auswählen:</label>
          <select id="imageSelect" v-model="selectedImage">
            <option value="stadt">Stadtbild</option>
            <option value="ziel">Zielscheibe</option>
            <option value="fantasy">Fantasy-Welt</option>
          </select>
        </div>

        <div class="heatmap-wrapper" ref="trackingArea" @mousemove="updateMouse">
            <div class="stacked-container">
              <img
                  :src="imageSources[selectedImage]"
                  alt="Content"
                  ref="imageRef"
                  class="background-image"
                  @load="onImageLoad"
              />
                <div ref="heatmapContainer" class="heatmap-overlay"></div>
            </div>
        </div>

        <div class="buttons">
            <button @click="toggleTracking">{{ isTracking ? 'Stop Tracking' : 'Start Tracking' }}</button>
            <button @click="clearHeatmap" :disabled="!heatmapInstance">Heatmap leeren</button>
            <button
                @click="exportHeatmap"
                :disabled="!heatmapInstance || allHeatmapPoints.length === 0"
            >
                Export
            </button>
        </div>

        <div class="slider-container" v-if="!isTracking && trackingDuration > 0">
            <label for="timeSlider">Zeige bis Sekunde: {{ sliderValue }} / {{ trackingDuration }}</label>
            <input
                id="timeSlider"
                type="range"
                :min="0"
                :max="trackingDuration"
                v-model="sliderValue"
            />
        </div>
    </div>
</template>

<script setup>
import { ref, computed, onBeforeUnmount, watch } from 'vue'
import html2canvas from 'html2canvas'
const h337 = window.h337  // Zugriff auf heatmap.js über das globale Fensterobjekt

// Reaktiver Zustand für das Tracking
const isTracking = ref(false)
let intervalId = null
let heatmapInstance = null

// Refs auf DOM-Elemente
const trackingArea = ref(null)
const heatmapContainer = ref(null)
const imageRef = ref(null)

// Mauskoordinaten relativ zum Trackingbereich
const mouse = { x: 0, y: 0 }

// Alle erfassten Punkte mit Zeitstempel
const allHeatmapPoints = ref([])

// Start und Ende des Trackings
const trackingStartedAt = ref(null)
const trackingEndedAt = ref(null)

const trackingMode = ref('mouse') // Standard: Maus-Tracking
const selectedImage = ref('stadt') // Standardmäßig Stadtbild

const imageSources = {
  stadt: '/City.png',
  ziel: '/Zielscheibe.png',
  fantasy: '/fantasy.png'
}

// Dauer des Trackings in Sekunden (abgeleitet)
const trackingDuration = computed(() => {
  let dauer = 0;

  if (trackingStartedAt.value && trackingEndedAt.value) {
    const differenz = trackingEndedAt.value - trackingStartedAt.value;
    dauer = Math.round(differenz / 1000);
  } else {
    dauer = 0;
  }

  return dauer;
})

// Aktueller Wert des Zeit-Schiebereglers
const sliderValue = ref(0)

// Berechnung der Mausposition relativ zum Container
function updateMouse(e) {
    const rect = trackingArea.value.getBoundingClientRect()
    mouse.x = e.clientX - rect.left
    mouse.y = e.clientY - rect.top
}

// Initialisierung der Heatmap nach dem Laden des Bildes
function onImageLoad() {
    const heatmap = heatmapContainer.value
    const image = imageRef.value

    const width = image.clientWidth
    const height = image.clientHeight

    // Heatmap-Größe an Bild anpassen
    heatmap.style.width = width + 'px'
    heatmap.style.height = height + 'px'

    // Heatmap erstellen mit Konfiguration
    heatmapInstance = h337.create({
        container: heatmap,
        radius: 30,
        maxOpacity: 0.6,
        minOpacity: 0.1,
        blur: 0.85,
        renderer: 'dom',
    })

    console.log('Heatmap wurde beim Bild-Load initialisiert')
}

// Startet oder stoppt das Tracking
function toggleTracking() {
  isTracking.value = !isTracking.value
  if (isTracking.value) {
    allHeatmapPoints.value = []
    trackingStartedAt.value = Date.now()

    if (trackingMode.value === 'mouse') {
      // normales Maus-Tracking starten
      intervalId = setInterval(() => {
        const rect = trackingArea.value.getBoundingClientRect()
        if (mouse.x >= 0 && mouse.y >= 0 && mouse.x <= rect.width && mouse.y <= rect.height) {
          const point = { x: mouse.x, y: mouse.y, value: 1, timestamp: Date.now() }
          heatmapInstance?.addData(point)
          allHeatmapPoints.value.push(point)
        }
      }, 150)
    } else if (trackingMode.value === 'eye') {
      console.log('👁️ Eye-Tracking wäre hier aktiviert...')
      // Hier später WebGazer starten
    }

  } else {
    clearInterval(intervalId)
    trackingEndedAt.value = Date.now()
    sliderValue.value = trackingDuration.value
    updateHeatmapToSlider()
  }
}

// Leert die Heatmap und setzt alle Zustände zurück
function clearHeatmap() {
    heatmapInstance?.setData({ max: 1, data: [] }) // Sichtbare Punkte löschen
    allHeatmapPoints.value = []
    trackingStartedAt.value = null
    trackingEndedAt.value = null
    sliderValue.value = 0
    console.log('Heatmap geleert & Reset durchgeführt')
}

// Screenshot der aktuellen Heatmap + Bild exportieren
function exportHeatmap() {
    if (allHeatmapPoints.value.length === 0) {
        console.warn('Kein Heatmap-Inhalt für den Export vorhanden')
        return
    }

    html2canvas(trackingArea.value).then(canvas => {
        const link = document.createElement('a')
        link.href = canvas.toDataURL('image/png')
        link.download = 'heatmap.png'
        link.click()
    })
}

// Zeigt nur Punkte an, die zum Zeitpunkt des Sliders passen
function updateHeatmapToSlider() {
    const cutoff = trackingStartedAt.value + sliderValue.value * 1000
    const visiblePoints = allHeatmapPoints.value.filter(p => p.timestamp <= cutoff)
    heatmapInstance?.setData({ max: 1, data: visiblePoints })
}

// Wenn der Slider bewegt wird, Heatmap aktualisieren
watch(sliderValue, () => {
    if (!isTracking.value && trackingStartedAt.value) {
        updateHeatmapToSlider()
    }
})

// Beim Verlassen der Komponente: Intervall stoppen
onBeforeUnmount(() => {
    clearInterval(intervalId)
})
</script>

<style scoped>
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

.startseite {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding-top: 60px;
}

.heatmap-wrapper {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 100%;
    max-width: 800px;
    margin: 2rem auto;
}

.stacked-container {
    position: relative;
    width: 100%;
}

.background-image {
    position: absolute;
    top: 0;
    left: 0;
    z-index: 1;
    width: 100%;
    height: auto;
    display: block;
}

.heatmap-overlay {
    position: absolute;
    top: 0;
    left: 0;
    z-index: 2;
    width: 100%;
    height: auto;
    pointer-events: none;
    will-change: transform, opacity;
    contain: strict;
}

.buttons {
    display: flex;
    gap: 1rem;
    margin-bottom: 2rem;
}

.slider-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-bottom: 2rem;
}

.slider-container input[type="range"] {
    width: 300px;
    margin-top: 0.5rem;
}
</style>