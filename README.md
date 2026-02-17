# Documentazione Tecnica: Live Shopping Interattiva con Ant Media Server

## 1. Panoramica dell'Architettura

La soluzione implementa un sistema di streaming **One-to-Many** a bassissima latenza (<500ms) basato su WebRTC. L'architettura segue il pattern **Master-Slave (Source of Truth)** per garantire coerenza assoluta tra ciò che il presentatore trasmette e ciò che gli spettatori vedono a livello di interfaccia interattiva.

### Flusso Dati e Sincronizzazione

1. **Presenter (Master):** Unico detentore dello stato temporale. Gestisce il mixaggio video (Canvas), l'acquisizione audio (Microfono) e il timer del carosello (7s).
2. **Ant Media Server (AMS):** Relay SFU per la distribuzione massiva del flusso A/V e dei metadati via Data Channel.
3. **Viewer (Slave):** Client passivo che renderizza lo stream e reagisce esclusivamente ai comandi ricevuti dal Master, eliminando drift temporali tra i partecipanti.

---

## 2. Stack Tecnologico

- **Server:** Ant Media Server Enterprise Edition (WebRTCAppEE).
- **Protocolli:** WebRTC (A/V), SCTP (Data Channels), WSS (Signaling).
- **Composizione:** HTML5 Canvas API (Video Mixing).
- **Audio:** Web Audio API / MediaStream (Microphone capture).
- **Frontend:** JavaScript (ES6 Modules), CSS3 (Flexbox/Grid, Media Queries).

---

## 3. Implementazione Presenter (`index.html`)

Il Presenter funge da "regia" della sessione live.

### 3.1. Composizione A/V (Clean Feed + Mic)

Il segnale inviato ad AMS è uno stream composito creato dinamicamente:

- **Video:** Generato catturando il contesto della Canvas (`canvas.captureStream(25)`).
- **Audio:** Acquisito tramite `getUserMedia({ audio: true })` e iniettato nello stream della Canvas tramite `addTrack()`.
- **Mixing:** \* _Video Mode:_ Mix di file video locale e webcam in PiP.
- _Webcam Mode:_ Webcam full-screen con calcolo dinamico dell'aspect ratio (Cover).

### 3.2. Regia e Sincronizzazione Master

Il Master controlla l'esperienza degli utenti tramite invio di payload JSON:

- **Automatico:** Un `setInterval` locale (7s) in Video Mode fa avanzare il carosello e notifica simultaneamente i viewer (`update_product`).
- **Manuale:** In Webcam Mode, il selettore prodotti permette al presentatore di forzare la visualizzazione di un item specifico (`force_product`).
- **Badge ON AIR:** Un indicatore visivo mostra al presentatore il prodotto attualmente "in onda" lato spettatore.

---

## 4. Implementazione Viewer (`player.html`)

Il Viewer è progettato per la massima resilienza e accessibilità mobile.

### 4.1. Gestione Audio e Autoplay Recovery

Per superare le restrizioni browser (Autoplay Policy):

- Il video viene inizializzato come `muted`.
- Un **Play Overlay** ("Click to Join") intercetta l'interazione utente necessaria per sbloccare sia il video che l'audio.
- Presente un toggle manuale **Mute/Unmute** per permettere all'utente di gestire il volume del microfono del presentatore.

### 4.2. Layout Mobile Responsive

Tramite Media Queries, l'interfaccia si trasforma per l'uso verticale:

- **Desktop:** Wishlist, Carosello e Product Card affiancati orizzontalmente.
- **Mobile:** Struttura a pila verticale ottimizzata. La **Wishlist è sempre visibile** (max-height limitata e scrollabile) sopra il carosello, mantenendo la centralità del video.

---

## 5. Protocollo di Comunicazione (Data Channel)

Payload JSON scambiati via SCTP per interattività real-time.

### Comandi Master → Viewer

| Action     | Payload                                      | Funzione                                |
| ---------- | -------------------------------------------- | --------------------------------------- |
| **Update** | `{ "action": "update_product", "index": N }` | Cambio prodotto ciclico (Video Mode).   |
| **Force**  | `{ "action": "force_product", "index": N }`  | Cambio prodotto manuale (Webcam Mode).  |
| **Sync**   | `{ "action": "sync_status", "index": N }`    | Allineamento costante per late joiners. |

### Feedback Viewer → Master

| Action     | Payload                                       | Funzione                 |
| ---------- | --------------------------------------------- | ------------------------ |
| **Add**    | `{ "action": "wishlist_add", "index": N }`    | Notifica aggiunta item.  |
| **Remove** | `{ "action": "wishlist_remove", "index": N }` | Notifica rimozione item. |

---

## 6. Note di Deployment e Sicurezza

- **HTTPS:** Indispensabile per il funzionamento di `getUserMedia` e dei Data Channel.
- **CORS:** Il file `background.mp4` deve risiedere nella stessa origine dell'applicazione o essere servito con header `Access-Control-Allow-Origin: *` e caricato con attributo `crossorigin="anonymous"` per non "contaminare" la Canvas.
- **Permissions:** L'applicazione richiede esplicitamente i permessi per Camera e Microfono lato Presentatore.

### Struttura Finale del Progetto

```
ant-demo/
├─ background.mp4      # Video di sfondo locale
├─ index.html          # Interfaccia Presenter (Master)
└─ player.html         # Interfaccia Viewer (Slave) + Mobile UI
```
