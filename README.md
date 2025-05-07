# Eye-Tracking Heatmap Webanwendung

Diese Webanwendung erlaubt die Erfassung und Visualisierung von Blickdaten über die Webcam.
Die gesammelten Daten werden live als Heatmap dargestellt und können anschließend exportiert werden.
Die Anwendung eignet sich zur Analyse visueller Aufmerksamkeit, Usability-Tests oder Studien zur Nutzerinteraktion.

---

## ✨ Funktionen

- **Bildauswahl**: Nutzer können ein beliebiges Bild hochladen
- **Dynamische Anpassung**: Die Heatmap wird exakt an die Bildgröße angepasst
- **Kalibrierung**: 9-Punkte-Klick zur Verbesserung der Genauigkeit
- **Webcam-basiertes Eye-Tracking** über [WebGazer.js](https://webgazer.cs.brown.edu/)
- **Heatmap bleibt sichtbar**, auch nach dem Stoppen des Trackings
- **Heatmap leeren** für eine neue Messung
- **Export der Blickdaten**:
    - PNG (Screenshot)
    - JSON (normierte Gaze-Daten)
    - Excel (.xlsx)

---

## 🚀 Technologie-Stack

- **Vue.js 3** (Composition API)
- **heatmap.js** – Darstellung der Blickpunkte als Heatmap
- **WebGazer.js** – Eye-Tracking über Webcam
- **html2canvas** – Screenshot-Export als PNG
- **SheetJS (xlsx)** – Excel-Export

---

## 🧭 Ablauf

1. **Bild auswählen**  
   Lade ein eigenes Bild hoch, das als Hintergrund für die Blickerfassung dient.

2. **Tracking starten**  
   Beim Klick auf „Start Tracking“ wird die Webcam aktiviert und eine 9-Punkte-Kalibrierung angezeigt.

3. **Kalibrierung durchführen**  
   Klicke nacheinander auf die neun eingeblendeten Punkte, um die Blickerfassung zu kalibrieren.

4. **Eye-Tracking läuft**  
   Nach der Kalibrierung werden Blickpunkte erfasst und auf der Heatmap live angezeigt.

5. **Tracking beenden (optional)**  
   Mit „Stop Tracking“ kann das Eye-Tracking beendet werden. Die Heatmap bleibt sichtbar.

6. **Heatmap exportieren**  
   Die gesammelten Daten lassen sich in verschiedenen Formaten speichern:
    - **PNG**: Screenshot des Trackingbereichs inkl. Bild und Heatmap
    - **JSON**: Blickdaten mit Zeitstempel und normierten Koordinaten
    - **Excel**: Gaze-Daten als `.xlsx` zur Weiterverarbeitung in Tabellen

---

## 📦 Projekt starten

Installiere die benötigten Pakete und starte den Entwicklungsserver:

```bash
npm install
npm run dev
```


