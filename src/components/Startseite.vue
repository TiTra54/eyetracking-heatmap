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
//import 'webgazer';

// Zugriff auf heatmap.js über das globale Fensterobjekt
const h337 = window.h337

// Zustände für das Tracking
const isTracking = ref(false)
const trackingMode = ref('mouse') // Standardmäßig Maus-Tracking
const selectedImage = ref('stadt') // Standardmäßig Stadtbild
const sliderValue = ref(0)

let intervalId = null
let heatmapInstance = null

// Referenzen auf DOM-Elemente
const trackingArea = ref(null)
const heatmapContainer = ref(null)
const imageRef = ref(null)

// Mausposition innerhalb des Trackingbereichs
const mouse = { x: 0, y: 0 }

// Punkte mit Zeitstempel
const allHeatmapPoints = ref([])
const trackingStartedAt = ref(null)
const trackingEndedAt = ref(null)

// Bildquellen für die Auswahl
const imageSources = {
  stadt: '/City.png',
  ziel: '/Zielscheibe.png',
  fantasy: '/fantasy.png'
}

// Berechnet die Dauer des Trackings in Sekunden
const trackingDuration = computed(() => {
  if (trackingStartedAt.value && trackingEndedAt.value) {
    const differenz = trackingEndedAt.value - trackingStartedAt.value
    return Math.round(differenz / 1000)
  }
  return 0
})

// Aktualisiert die Mausposition relativ zum Trackingbereich
function updateMouse(event) {
  const area = trackingArea.value.getBoundingClientRect()
  mouse.x = event.clientX - area.left
  mouse.y = event.clientY - area.top
}

// Initialisiert die Heatmap, wenn das Bild geladen wurde
function onImageLoad() {
  const heatmap = heatmapContainer.value
  const image = imageRef.value

  const width = image.clientWidth
  const height = image.clientHeight

  // ⬅️ Heatmap-Container an exakte Bildgröße anpassen
  heatmap.style.width = width + 'px'
  heatmap.style.height = height + 'px'

  // ❗Vorherige Instanz zerstören, wenn vorhanden
  if (heatmapInstance) {
    heatmapInstance.setData({ max: 1, data: [] })
    heatmap.innerHTML = ''
    heatmapInstance = null
  }

  // 🔥 Neue Heatmap-Instanz erzeugen
  heatmapInstance = h337.create({
    container: heatmap,
    radius: 30,
    maxOpacity: 0.6,
    minOpacity: 0.1,
    blur: 0.85,
    renderer: 'canvas'
  })

  console.log('📷 Bild geladen:', width, 'x', height)
  console.log('🔥 Neue Heatmap initialisiert')

  // 🧪 Debug: Canvas sichtbar?
  setTimeout(() => {
    const canvas = heatmap.querySelector('canvas')
    if (canvas) {
      console.log('✅ Canvas vorhanden → Größe:', canvas.width, 'x', canvas.height)
      console.log('✅ Sichtbarkeit: ', getComputedStyle(canvas).display, getComputedStyle(canvas).opacity)
    } else {
      console.warn('⚠️ Kein Canvas im DOM gefunden!')
    }
  }, 100)
}

// Schaltet das Tracking ein oder aus (abhängig vom aktuellen Zustand)
function toggleTracking() {
  // Zustand umkehren (Start <-> Stop)
  isTracking.value = !isTracking.value

  if (isTracking.value) {
    // Wenn Tracking startet:
    allHeatmapPoints.value = []                // Vorherige Punkte zurücksetzen
    trackingStartedAt.value = Date.now()       // Startzeit festhalten

    // Je nach Modus das passende Tracking starten
    if (trackingMode.value === 'mouse') {
      startMouseTracking()
    } else if (trackingMode.value === 'eye') {
      startEyeTracking()
    }
  } else {
    // Wenn Tracking gestoppt wird:
    stopTracking()
  }
}

// Startet das Maus-Tracking (setzt in regelmäßigen Abständen neue Punkte)
function startMouseTracking() {
  intervalId = setInterval(() => {
    const area = trackingArea.value.getBoundingClientRect()  // Trackingbereich berechnen

    // Prüfen, ob Maus innerhalb des Bereichs ist
    const innerhalb =
        mouse.x >= 0 &&
        mouse.y >= 0 &&
        mouse.x <= area.width &&
        mouse.y <= area.height

    // Wenn Maus innerhalb → Punkt zur Heatmap hinzufügen
    if (innerhalb) {
      const punkt = {
        x: mouse.x,
        y: mouse.y,
        value: 1,
        timestamp: Date.now()
      }

      // Punkt an die Heatmap übergeben
      heatmapInstance?.addData(punkt)

      // Punkt zusätzlich in Liste speichern (für spätere Auswertung / Export)
      allHeatmapPoints.value.push(punkt)
    }
  }, 150) // Alle 150ms neuer Punkt
}

function startEyeTracking() {
  if (window.webgazer) {
    window.webgazer
        .setGazeListener((data, timestamp) => {
          console.log('👁️ Blickdaten:', data)
          if (data && isTracking.value) {
            const rect = trackingArea.value.getBoundingClientRect();
            const x = data.x - rect.left;
            const y = data.y - rect.top;

            const innerhalb =
                x >= 0 &&
                y >= 0 &&
                x <= rect.width &&
                y <= rect.height;

            if (innerhalb) {
              const punkt = {
                x: x,
                y: y,
                value: 1,
                timestamp: Date.now()
              };
              heatmapInstance?.addData(punkt);
              allHeatmapPoints.value.push(punkt);
            }
          }
        })
        .begin()
        .showVideoPreview(false)
        .showFaceOverlay(false)
        .showFaceFeedbackBox(false);

    console.log('WebGazer Eye-Tracking gestartet ✅');
  } else {
    console.error('WebGazer nicht verfügbar!');
  }
}

function stopEyeTracking() {
  if (window.webgazer) {
    window.webgazer.pause();                     // Blickdaten stoppen
    window.webgazer.end();                       // Kamera & Ressourcen beenden
    console.log('WebGazer Eye-Tracking vollständig gestoppt ⛔️');
  }
}

function stopTracking() {
  clearInterval(intervalId)

  if (trackingMode.value === 'eye') {
    stopEyeTracking()
  }

  trackingEndedAt.value = Date.now()
  sliderValue.value = trackingDuration.value

  // 🔍 DEBUG: Anzahl Punkte & Inhalte prüfen
  console.log('🟡 Tracking wurde gestoppt')
  console.log('🔢 Gesamtpunkte:', allHeatmapPoints.value.length)
  console.log('⏱️ Trackingdauer:', trackingDuration.value)

  // 🔥 Versuch: Heatmap direkt setzen
  if (heatmapInstance && allHeatmapPoints.value.length > 0) {
    const heatmapData = {
      max: 1,
      data: allHeatmapPoints.value
    }

    heatmapInstance.setData(heatmapData)
    console.log('✅ Heatmap gesetzt (nach Stop)')
    console.log('📦 Aktuelle Heatmap-Daten:', heatmapInstance.getData())
  } else {
    console.warn('❌ HeatmapInstance oder Daten fehlen')
  }

  // 🚫 Testweise: updateHeatmapToSlider() deaktivieren
  // updateHeatmapToSlider()
}

// Leert die Heatmap über den Button
function clearHeatmap() {
  resetHeatmap()
  console.log('Heatmap wurde geleert')
}

// Setzt die Heatmap und alle zugehörigen Daten zurück
function resetHeatmap() {
  if (heatmapInstance) {
    heatmapInstance.setData({ max: 1, data: [] })
  }
  allHeatmapPoints.value = []
  trackingStartedAt.value = null
  trackingEndedAt.value = null
  sliderValue.value = 0
  console.log('Heatmap und Trackingdaten wurden zurückgesetzt')
}

// Exportiert die Heatmap inklusive Bild als PNG
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

// Zeigt alle Punkte bis zu einem bestimmten Zeitpunkt (Schieberegler)
function updateHeatmapToSlider() {
  const cutoff = trackingStartedAt.value + sliderValue.value * 1000;
  const gefiltertePunkte = allHeatmapPoints.value.filter(p => p.timestamp <= cutoff);

  if (heatmapInstance) {
    heatmapInstance.setData({ max: 1, data: gefiltertePunkte });

    // 🔍 DEBUG: sicherstellen, dass nach setData auch neu gezeichnet wird
    setTimeout(() => {
      const canvas = heatmapContainer.value.querySelector('canvas');
      if (canvas) {
        console.log('✅ Canvas-Update sichtbar. Größe:', canvas.width, 'x', canvas.height);
      } else {
        console.warn('⚠️ Kein Canvas gefunden!');
      }
    }, 50);
  } else {
    console.warn('⚠️ Kein Heatmap-Instance beim Update!');
  }
}

// Beobachtet den Schieberegler und aktualisiert die Anzeige
watch(sliderValue, () => {
  console.log('🎚 Slider bewegt:', sliderValue.value)
  if (!isTracking.value && trackingStartedAt.value) {
    updateHeatmapToSlider()
  }
})

// Beobachtet den Bildwechsel und setzt die Heatmap zurück
watch(selectedImage, () => {
  resetHeatmap()
})

// Beobachtet den Moduswechsel und setzt die Heatmap zurück
watch(trackingMode, () => {
  resetHeatmap()
})

// Stoppt das Intervall beim Verlassen der Komponente
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
  height: 100%; /* ✅ Wichtig */
  pointer-events: none;
  background-color: rgba(255, 0, 0, 0.03); /* leichte Sichtbarkeit zur Kontrolle */
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