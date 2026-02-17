# Documentazione Tecnica: Live Shopping Interattiva con Ant Media Server

## 1. Panoramica dell'Architettura

La soluzione realizza un sistema di streaming **One-to-Many** a bassissima latenza (Low Latency WebRTC) per applicazioni di Live Commerce.
L'architettura adotta un pattern **Master-Slave (Source of Truth)**: il Presenter (Master) detiene lo stato dell'applicazione e il timer del carosello, mentre i Viewer (Slaves) si limitano a replicare lo stato ricevuto, garantendo una sincronizzazione perfetta indipendentemente dalla latenza di rete o dal momento di connessione.

### Flusso Dati

1. **Presenter (Master):** Gestisce il loop temporale (7s), il mixaggio video (Canvas) e invia comandi di stato (Indice prodotto, Play/Pause).
2. **Ant Media Server (AMS):** Agisce come relay WebRTC (SFU) per la distribuzione dello stream e il routing dei messaggi Data Channel.
3. **Viewer (Slave):** Riceve lo stream video e i comandi JSON. Non calcola il tempo localmente, ma aggiorna la UI (Carosello, Product Card) solo su istruzione del Master.

---

## 2. Stack Tecnologico

- **Core Server:** Ant Media Server Enterprise Edition (WebRTCAppEE).
- **Protocollo:** WebRTC (Video/Audio) + SCTP (Data Channels).
- **Frontend:** HTML5 Canvas API, JavaScript (ES6 Modules), CSS3 Media Queries.
- **Libreria Client:** `@antmedia/webrtc_adaptor` (versione Snapshot).

---

## 3. Implementazione Presenter (`index.html`)

Il Presenter funge da mixer video e "Timekeeper" dell'applicazione.

### 3.1. Composizione Video (Canvas Mixing)

Il rendering avviene su un elemento `<canvas>` tramite `requestAnimationFrame`.

- **Sorgenti:** `videoBg` (Video promozionale) e `webcam` (User Media).
- **Modalità di Streaming:**
- **Video Mode:** Background video attivo + Webcam in Picture-in-Picture. Il carosello avanza automaticamente.
- **Webcam Mode:** Webcam a tutto schermo (simulazione `object-fit: cover`). Il carosello è manuale e controllato dal presentatore.

### 3.2. Master Timer & Badge "On Air"

In _Video Mode_, il presentatore esegue un `setInterval` (7000ms). Ad ogni tick:

1. Aggiorna l'indice locale e il badge visivo **"ON AIR"** (visibile solo al presentatore).
2. Invia il comando `update_product` a tutti gli spettatori via Data Channel.

### 3.3. Heartbeat di Sincronizzazione

Per allineare gli utenti che si connettono in ritardo (Late Joiners):

- Un intervallo separato (2000ms) invia il comando `sync_status` contenente l'**indice corrente** del prodotto e lo stato (play/pause).

---

## 4. Implementazione Viewer (`player.html`)

Il Viewer è un client passivo ottimizzato per la compatibilità e la responsività.

### 4.1. Gestione Autoplay & Recovery

Per aggirare le policy di Autoplay dei browser moderni:

- Il video parte sempre `muted`.
- Se la Promise `play()` fallisce (schermo nero), viene mostrato automaticamente un overlay interattivo **"Click to Start Stream"**.

### 4.2. UI Responsive (Mobile First)

L'interfaccia HTML sovrapposta si adatta al dispositivo tramite CSS Media Queries (`max-width: 768px`):

- **Desktop:** Layout orizzontale in basso.
- **Mobile:** Layout a "pila" verticale (Wishlist in alto, Carosello al centro, Scheda Prodotto in basso), sempre visibili e ottimizzati per il tocco.

### 4.3. Logica Slave

Il viewer non possiede logica temporale (`setInterval`). Ascolta i messaggi `data_received`:

- `update_product` / `force_product` / `sync_status`: Al ricevimento, aggiorna immediatamente l'indice del carosello e renderizza il prodotto corrispondente.

---

## 5. Protocollo di Comunicazione (Data Channel)

La comunicazione avviene tramite payload JSON bidirezionali.

### Presenter → Viewer (Comandi Master)

| Action         | Payload                                      | Descrizione                                | Trigger                                 |
| -------------- | -------------------------------------------- | ------------------------------------------ | --------------------------------------- |
| **Update**     | `{ "action": "update_product", "index": N }` | Avanzamento automatico del carosello.      | Timer (7s) del Presenter in Video Mode. |
| **Force**      | `{ "action": "force_product", "index": N }`  | Cambio prodotto manuale immediato.         | Click su Product Bar in Webcam Mode.    |
| **Sync**       | `{ "action": "sync_status", "index": N }`    | Allineamento stato per nuovi utenti.       | Heartbeat (2s) del Presenter.           |
| **Play/Pause** | `{ "action": "play" }` (o pause)             | Segnale di stato video (opzionale per UI). | Toggle manuale del Presenter.           |

### Viewer → Presenter (Feedback)

| Action     | Payload                                       | Descrizione                        | Trigger                 |
| ---------- | --------------------------------------------- | ---------------------------------- | ----------------------- |
| **Add**    | `{ "action": "wishlist_add", "index": N }`    | Notifica aggiunta alla wishlist.   | Utente clicca "Add".    |
| **Remove** | `{ "action": "wishlist_remove", "index": N }` | Notifica rimozione dalla wishlist. | Utente clicca "Remove". |

---

## 6. Configurazione e Deployment

- **Ant Media Server:** Endpoint WSS standard (`wss://.../WebRTCAppEE/websocket`).
- **CORS (Cross-Origin Resource Sharing):**
- Il video di background (`videoBg`) viene caricato da file locale per evitare il "Tainting" della Canvas.
- Se si usa un video remoto, il server ospitante _deve_ esporre l'header `Access-Control-Allow-Origin: *` e il tag video deve avere `crossorigin="anonymous"`.

- **HTTPS:** Obbligatorio per l'accesso ai dispositivi (Webcam) e per la stabilità del Data Channel su reti pubbliche.

### Struttura Progetto

```
ant-demo/
├─ background.mp4      # Video locale (Scaricato per evitare CORS)
├─ index.html          # Presenter (Master Logic)
└─ player.html         # Viewer (Slave Logic + Mobile UI)

```
