# Ampel-Router — Komplette Anleitung

---

## Schritt 1 — Kostenloser GraphHopper API-Key

1. Geh zu: **https://console.graphhopper.com**
2. Account erstellen (E-Mail reicht)
3. API-Key kopieren (sieht aus wie: `a1b2c3d4-...`)
4. Gratis-Tier: 500 Anfragen/Tag — für persönliche Nutzung völlig ausreichend

---

## Schritt 2 — Projektordner anlegen

```bash
mkdir ampel-router && cd ampel-router
```

---

## Schritt 3 — HTML-Datei erstellen

Füge folgenden Block komplett in dein Terminal ein.
**Ersetze `DEIN_API_KEY_HIER` mit deinem echten Key aus Schritt 1.**

```bash
cat > index.html << 'HTMLEOF'
<!DOCTYPE html>
<html lang="de">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Ampel-Router</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=DM+Mono:wght@400;500&family=Syne:wght@700;800&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --bg:        #0d0d0d;
      --panel:     #141414;
      --border:    #2a2a2a;
      --accent:    #c8f135;
      --red:       #ff4545;
      --text:      #e8e8e8;
      --muted:     #666;
      --radius:    10px;
    }

    body {
      font-family: 'DM Mono', monospace;
      background: var(--bg);
      color: var(--text);
      height: 100dvh;
      display: flex;
      flex-direction: column;
      overflow: hidden;
    }

    /* ── PANEL ── */
    #panel {
      background: var(--panel);
      border-bottom: 1px solid var(--border);
      padding: 14px 16px 12px;
      flex-shrink: 0;
      display: flex;
      flex-direction: column;
      gap: 10px;
    }

    .panel-top {
      display: flex;
      align-items: center;
      gap: 10px;
    }

    h1 {
      font-family: 'Syne', sans-serif;
      font-size: 1.15rem;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: var(--accent);
      flex: 1;
    }

    /* ── INPUTS ── */
    .field {
      display: flex;
      gap: 6px;
      align-items: center;
    }

    .field-label {
      font-size: 0.65rem;
      color: var(--muted);
      text-transform: uppercase;
      letter-spacing: 0.08em;
      width: 28px;
      flex-shrink: 0;
    }

    input[type="text"] {
      flex: 1;
      padding: 8px 11px;
      background: var(--bg);
      border: 1px solid var(--border);
      border-radius: var(--radius);
      color: var(--text);
      font-family: 'DM Mono', monospace;
      font-size: 0.82rem;
      outline: none;
      transition: border-color 0.2s;
    }

    input[type="text"]:focus { border-color: var(--accent); }
    input[type="text"]::placeholder { color: var(--muted); }

    #gps-btn {
      padding: 7px 11px;
      background: transparent;
      border: 1px solid var(--border);
      border-radius: var(--radius);
      color: var(--accent);
      cursor: pointer;
      font-size: 0.85rem;
      transition: all 0.15s;
      flex-shrink: 0;
    }

    #gps-btn:hover { background: var(--accent); color: var(--bg); }

    /* ── OPTION ROWS ── */
    .options-row {
      display: flex;
      gap: 6px;
      flex-wrap: wrap;
    }

    .chip {
      padding: 5px 13px;
      border: 1px solid var(--border);
      border-radius: 20px;
      background: transparent;
      color: var(--muted);
      font-family: 'DM Mono', monospace;
      font-size: 0.75rem;
      cursor: pointer;
      transition: all 0.15s;
      white-space: nowrap;
    }

    .chip:hover { border-color: var(--text); color: var(--text); }

    .chip.active {
      background: var(--accent);
      border-color: var(--accent);
      color: var(--bg);
      font-weight: 500;
    }

    .chip.active.red {
      background: var(--red);
      border-color: var(--red);
      color: white;
    }

    /* ── ROUTE BUTTON ── */
    #route-btn {
      padding: 9px;
      width: 100%;
      background: var(--accent);
      color: var(--bg);
      border: none;
      border-radius: var(--radius);
      font-family: 'Syne', sans-serif;
      font-size: 0.9rem;
      font-weight: 700;
      letter-spacing: 0.03em;
      cursor: pointer;
      transition: opacity 0.15s;
    }

    #route-btn:hover { opacity: 0.88; }
    #route-btn:disabled { opacity: 0.4; cursor: not-allowed; }

    /* ── STATUS ── */
    #status {
      font-size: 0.72rem;
      color: var(--muted);
      min-height: 15px;
      text-align: center;
    }

    #status.error { color: var(--red); }
    #status.ok    { color: var(--accent); }

    /* ── MAP ── */
    #map { flex: 1; }

    /* ── LEAFLET OVERRIDES ── */
    .leaflet-popup-content-wrapper {
      background: var(--panel);
      color: var(--text);
      border: 1px solid var(--border);
      border-radius: var(--radius);
      box-shadow: none;
    }

    .leaflet-popup-tip { background: var(--panel); }
  </style>
</head>
<body>

<div id="panel">
  <div class="panel-top">
    <h1>🚦 Ampel-Router</h1>
  </div>

  <div class="field">
    <span class="field-label">Von</span>
    <input type="text" id="start-input" placeholder="Startadresse..." />
    <button id="gps-btn" onclick="useGPS()" title="GPS nutzen">📍</button>
  </div>

  <div class="field">
    <span class="field-label">Nach</span>
    <input type="text" id="end-input" placeholder="Zieladresse..." />
  </div>

  <div class="options-row" id="mode-options">
    <button class="chip active" data-value="bike" onclick="setOption('mode', this)">🚴 Fahrrad</button>
    <button class="chip" data-value="foot" onclick="setOption('mode', this)">🏃 Joggen</button>
    <button class="chip" data-value="car" onclick="setOption('mode', this)">🚗 Auto</button>
  </div>

  <div class="options-row" id="avoid-options">
    <button class="chip active" data-value="minimize" onclick="setOption('avoid', this)">🟡 Minimieren</button>
    <button class="chip red" data-value="avoid" onclick="setOption('avoid', this)">🔴 Komplett vermeiden</button>
  </div>

  <button id="route-btn" onclick="getRoute()">Route berechnen</button>
  <div id="status"></div>
</div>

<div id="map"></div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
// ─────────────────────────────────────────
//  ⚙️  API KEY HIER EINTRAGEN
// ─────────────────────────────────────────
const API_KEY = 'DEIN_API_KEY_HIER';
// ─────────────────────────────────────────

const map = L.map('map', { zoomControl: true }).setView([52.3676, 4.9041], 13);

L.tileLayer('https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png', {
  attribution: '© OpenStreetMap © CARTO',
  maxZoom: 19
}).addTo(map);

let mode      = 'bike';
let avoidMode = 'minimize';
let gpsCoords = null;
let routeLayer = null;
let markerGroup = L.layerGroup().addTo(map);

// ── Option toggles ──
function setOption(group, btn) {
  const container = group === 'mode' ? '#mode-options' : '#avoid-options';
  document.querySelectorAll(container + ' .chip').forEach(b => b.classList.remove('active'));
  btn.classList.add('active');
  if (group === 'mode')  mode      = btn.dataset.value;
  if (group === 'avoid') avoidMode = btn.dataset.value;
}

// ── Status ──
function setStatus(msg, type = '') {
  const el = document.getElementById('status');
  el.textContent = msg;
  el.className = type;
}

// ── GPS ──
function useGPS() {
  setStatus('GPS wird abgerufen…');
  navigator.geolocation.getCurrentPosition(
    pos => {
      gpsCoords = [pos.coords.latitude, pos.coords.longitude];
      document.getElementById('start-input').value = `📍 ${gpsCoords[0].toFixed(5)}, ${gpsCoords[1].toFixed(5)}`;
      setStatus('Standort gesetzt.', 'ok');
    },
    err => setStatus('GPS-Fehler: ' + err.message, 'error')
  );
}

// ── Geocoding ──
async function geocode(query) {
  const url = `https://graphhopper.com/api/1/geocode?q=${encodeURIComponent(query)}&locale=de&limit=1&key=${API_KEY}`;
  const res  = await fetch(url);
  const data = await res.json();
  if (!data.hits?.length) throw new Error(`Adresse nicht gefunden: "${query}"`);
  const { lat, lng } = data.hits[0].point;
  return [lat, lng];
}

// ── Custom Model (Ampel-Proxy via Straßenklassen) ──
//
// GraphHopper kennt Ampeln als OSM-Knoten. Wir benutzen
// Straßenklassen als Proxy: Hauptstraßen (PRIMARY, SECONDARY)
// haben die meisten Ampeln. Wir senken deren Priorität.
//
function buildCustomModel() {
  if (avoidMode === 'minimize') {
    return {
      priority: [
        { if: "road_class == TRUNK",     multiply_by: "0.5" },
        { if: "road_class == PRIMARY",   multiply_by: "0.6" },
        { if: "road_class == SECONDARY", multiply_by: "0.75" }
      ]
    };
  } else {
    // Komplett vermeiden: drastische Abwertung von Hauptstraßen
    return {
      priority: [
        { if: "road_class == MOTORWAY",  multiply_by: "0.01" },
        { if: "road_class == TRUNK",     multiply_by: "0.02" },
        { if: "road_class == PRIMARY",   multiply_by: "0.05" },
        { if: "road_class == SECONDARY", multiply_by: "0.12" }
      ]
    };
  }
}

// ── Haupt-Routing ──
async function getRoute() {
  const startVal = document.getElementById('start-input').value.trim();
  const endVal   = document.getElementById('end-input').value.trim();

  if (!startVal || !endVal) {
    setStatus('Bitte Start und Ziel eingeben.', 'error');
    return;
  }

  const btn = document.getElementById('route-btn');
  btn.disabled = true;
  setStatus('Route wird berechnet…');

  try {
    // Start auflösen
    let from;
    if (gpsCoords && startVal.startsWith('📍')) {
      from = gpsCoords;
    } else {
      from = await geocode(startVal);
    }

    const to = await geocode(endVal);

    const body = {
      points: [[from[1], from[0]], [to[1], to[0]]],
      profile: mode,
      custom_model: buildCustomModel(),
      'ch.disable': true,
      locale: 'de',
      instructions: false,
      calc_points: true,
      points_encoded: false
    };

    const res  = await fetch(`https://graphhopper.com/api/1/route?key=${API_KEY}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body)
    });

    const data = await res.json();
    if (!res.ok) throw new Error(data.message || `API-Fehler ${res.status}`);

    const path   = data.paths[0];
    const latlngs = path.points.coordinates.map(([lng, lat]) => [lat, lng]);

    // Alte Route entfernen
    if (routeLayer) map.removeLayer(routeLayer);
    markerGroup.clearLayers();

    // Route zeichnen
    routeLayer = L.polyline(latlngs, {
      color: '#c8f135',
      weight: 5,
      opacity: 0.9,
      lineCap: 'round',
      lineJoin: 'round'
    }).addTo(map);

    // Marker
    const iconStyle = (label, color) => L.divIcon({
      html: `<div style="background:${color};color:#0d0d0d;font-family:Syne,sans-serif;font-weight:800;font-size:11px;padding:4px 8px;border-radius:6px;white-space:nowrap;box-shadow:0 2px 8px rgba(0,0,0,0.5)">${label}</div>`,
      className: '',
      iconAnchor: [0, 10]
    });

    L.marker(from, { icon: iconStyle('START', '#c8f135') }).addTo(markerGroup);
    L.marker(to,   { icon: iconStyle('ZIEL',  '#ff4545') }).addTo(markerGroup);

    map.fitBounds(routeLayer.getBounds(), { padding: [24, 24] });

    const km  = (path.distance / 1000).toFixed(1);
    const min = Math.round(path.time / 60000);
    setStatus(`✅  ${km} km · ca. ${min} min`, 'ok');

  } catch (e) {
    setStatus('❌ ' + e.message, 'error');
  } finally {
    btn.disabled = false;
  }
}

// Enter-Taste
document.addEventListener('keydown', e => {
  if (e.key === 'Enter') getRoute();
});
</script>
</body>
</html>
HTMLEOF
```

---

## Schritt 4 — Lokalen Server starten

GPS funktioniert nur über `http://localhost`, nicht über `file://`.
Wähle eine der Optionen:

**Option A — Python (meistens schon installiert):**
```bash
python3 -m http.server 8080
```

**Option B — Node (falls installiert):**
```bash
npx serve . -p 8080
```

---

## Schritt 5 — Im Browser öffnen

```
http://localhost:8080
```

---

## Schritt 6 — API-Key eintragen (falls noch nicht gemacht)

Öffne `index.html` in einem Texteditor und ersetze `DEIN_API_KEY_HIER`:

```bash
# macOS:
open index.html

# Linux:
nano index.html
```

Suche nach dieser Zeile und ersetze den Platzhalter:
```javascript
const API_KEY = 'DEIN_API_KEY_HIER';
```

---

## Fertige Dateistruktur

```
ampel-router/
└── index.html    ← alles in einer Datei
```

---

## Wichtiger Hinweis zur Genauigkeit

Die App vermeidet Ampeln **indirekt** über Straßenklassen als Proxy:

- 🟡 **Minimieren** → Hauptstraßen werden weniger bevorzugt, aber nicht ausgeschlossen
- 🔴 **Komplett vermeiden** → Hauptstraßen werden fast vollständig gemieden; der Weg kann deutlich länger werden

**Warum kein 100% präzises Ampel-Routing?**
Echte Ampel-Daten in OpenStreetMap sind nie vollständig. In Amsterdam ist die Abdeckung gut, aber nicht perfekt. Dieser Ansatz ist der beste Kompromiss ohne eigenen Routing-Server.
