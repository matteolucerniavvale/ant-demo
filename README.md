# Documentazione Tecnica: Live Shopping Interattiva con Ant Media Server

## 1. Panoramica dell'Architettura

La soluzione realizza un sistema di streaming **One-to-Many** a bassissima latenza (Low Latency WebRTC) per applicazioni di Live Commerce. L'architettura è **Client-Side Composition**: il mixaggio delle sorgenti video e grafiche avviene nel browser del presentatore, riducendo il carico server e permettendo interattività complessa senza post-produzione lato backend.

### Flusso Dati

1. **Presenter (Broadcaster):** Genera il feed audio/video composito e invia comandi di sincronizzazione.
2. **Ant Media Server (AMS):** Agisce come relay WebRTC (SFU) per la distribuzione dello stream e il routing dei messaggi Data Channel.
3. **Viewer (Player):** Riceve lo stream video e i metadati JSON per aggiornare l'interfaccia locale in tempo reale.

---

## 2. Stack Tecnologico

- **Core Server:** Ant Media Server Enterprise Edition (WebRTCAppEE).
- **Protocollo:** WebRTC (Video/Audio) + SCTP (Data Channels).
- **Frontend:** HTML5 Canvas API, JavaScript (ES6 Modules).
- **Libreria Client:** `@antmedia/webrtc_adaptor` (versione Snapshot).

---

## 3. Implementazione Presenter (`index.html`)

Il Presenter funge da mixer video e controller della logica di business.

### 3.1. Composizione Video (Canvas Mixing)

Il cuore del sistema è un loop di rendering (`requestAnimationFrame`) che disegna su un elemento `<canvas>` HTML5.

- **Sorgenti:**
- `videoBg`: Video promozionale (file locale o remoto con CORS abilitato).
- `webcam`: Stream `getUserMedia` locale.

- **Modalità di Streaming:**

1. **Video Mode:** Background video a tutto schermo + Webcam in Picture-in-Picture (top-right).
2. **Webcam Mode:** Webcam a tutto schermo con logica `object-fit: cover` calcolata matematicamente per preservare l'aspect ratio.

### 3.2. Generazione dello Stream

Lo stream inviato ad AMS non proviene direttamente dalla webcam, ma dalla Canvas:

```javascript
var localCanvasStream = canvas.captureStream(25); // 25 FPS
```

Questo permette di trasmettere esattamente ciò che viene renderizzato, inclusi cambi di layout istantanei.

### 3.3. Logica di Sincronizzazione (Heartbeat)

Per garantire che tutti gli spettatori (anche quelli che si connettono in ritardo) siano sincronizzati con lo stato del presentatore, è implementato un pattern **Heartbeat**:

- Un intervallo (`setInterval` 2000ms) invia costantemente lo stato corrente (`play` o `pause`) tramite Data Channel quando si è in Video Mode.
- In Webcam Mode, l'heartbeat viene sospeso per dare priorità ai comandi manuali (`force_product`).

---

## 4. Implementazione Viewer (`player.html`)

Il Viewer è un client passivo che reagisce agli stream e ai dati.

### 4.1. UI Overlay

L'interfaccia (Carosello prodotti, Wishlist, Product Card) **non** è impressa nel video (non è "burned-in"). È realizzata in HTML/CSS sovrapposto al tag `<video>`. Questo garantisce:

- Nitidezza del testo indipendente dalla qualità del video.
- Interattività locale (click sui bottoni).
- Riduzione della banda video necessaria.

### 4.2. Gestione Data Channel

Il player ascolta l'evento `data_received` dell'SDK Ant Media. I messaggi JSON in ingresso triggerano la logica locale:

- Ricezione `play`/`pause`: Avvia/Ferma il carosello automatico locale.
- Ricezione `force_product`: Forza la visualizzazione di un prodotto specifico e ferma il carosello.

---

## 5. Protocollo di Comunicazione (Data Channel)

La comunicazione avviene tramite payload JSON bidirezionali su canale SCTP.

### Presenter → Viewer (Comandi)

| Action    | Payload                                     | Descrizione                            | Trigger                       |
| --------- | ------------------------------------------- | -------------------------------------- | ----------------------------- |
| **Play**  | `{ "action": "play" }`                      | Avvia rotazione carosello lato client. | VideoBg Play o Heartbeat.     |
| **Pause** | `{ "action": "pause" }`                     | Ferma rotazione carosello lato client. | VideoBg Pause o Heartbeat.    |
| **Force** | `{ "action": "force_product", "index": N }` | Mostra prodotto N e stoppa rotazione.  | Click manuale in Webcam Mode. |

### Viewer → Presenter (Feedback)

| Action     | Payload                                       | Descrizione                        | Trigger                 |
| ---------- | --------------------------------------------- | ---------------------------------- | ----------------------- |
| **Add**    | `{ "action": "wishlist_add", "index": N }`    | Notifica aggiunta alla wishlist.   | Utente clicca "Add".    |
| **Remove** | `{ "action": "wishlist_remove", "index": N }` | Notifica rimozione dalla wishlist. | Utente clicca "Remove". |

---

## 6. Configurazione Ant Media Server

La connessione è stabilita via WebSocket Secure (WSS).

- **Endpoint:** `wss://[AMS-DOMAIN]:5443/WebRTCAppEE/websocket`
- **Media Constraints Viewer:** `{ video: false, audio: false }` (Poiché il viewer riceve solo, non invia media).
- **Error Handling:**
- `no_stream_exist`: Implementata logica di **Auto-Retry** (polling ogni 3 secondi) lato Viewer per gestire riconnessioni o avvio ritardato dello stream.

## 7. Note di Deployment

- **Requisiti CORS:** Le risorse video esterne caricate nella Canvas (`drawImage`) devono avere l'header `Access-Control-Allow-Origin: *` e attributo `crossorigin="anonymous"`, altrimenti la Canvas diventa "tainted" e il browser blocca lo streaming.
- **HTTPS:** Obbligatorio per l'accesso a `getUserMedia` (Webcam/Microfono) sui browser moderni.

```
ant-demo
├─ background.mp4
├─ index.html
└─ player.html

```
