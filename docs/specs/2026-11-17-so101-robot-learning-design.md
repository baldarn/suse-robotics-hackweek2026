# SO-101 Robot Learning Stack — Design (SUSE Hackweek, 30 nov 2026)

## Intento e perché

Progetto personale per l'Hackweek SUSE (inizio 30 novembre 2026, 5 giorni), fatto da solo con l'aiuto dell'AI.
L'obiettivo è **imparare e mostrare l'intero stack robotico moderno** su hardware reale: teleoperazione,
raccolta dati, addestramento di policy, inferenza autonoma, valutazione, orchestrazione tramite agente AI.
Non c'è una finalità aziendale: il valore è la competenza acquisita e un progetto open source dimostrabile.
Un minimo di infrastruttura (K3s, Fleet, Grafana) è incluso **solo se costa poco**.

## Obiettivi

1. Il braccio SO-101 esegue **in autonomia** almeno un compito di manipolazione appreso da dimostrazioni.
2. Ciclo completo riproducibile: teleop → dataset → training → deploy → valutazione → iterazione.
3. Almeno **due compiti** con policy distinte, così che un agente possa scegliere quale eseguire.
4. Un **agente AI via MCP** che da chat lancia/ferma compiti e legge lo stato del robot.
5. Telemetria visibile in **Grafana**; deploy delle versioni di policy via **Fleet (GitOps)** su **K3s**.
6. Video demo + README pubblicabili (repo GitHub pubblico).

**Criterio di successo minimo:** compito 1 con successo ≥ 7/10 tentativi autonomi, registrato in video.

## Non-obiettivi

- ROS 2 / MoveIt / pianificazione classica.
- LeKiwi o basi mobili (eventuale fase 2 dopo l'hackweek).
- Qualsiasi narrativa aziendale o pitch di prodotto.
- Multi-robot reale, sicurezza/hardening da produzione, multi-tenancy.

## Anti-obiettivi (fallimento anche se "funziona")

- Settimana spesa sull'infrastruttura con il robot che non fa nulla da solo.
  → L'infrastruttura si fa **dopo** che il compito 1 funziona, ed è la prima cosa da tagliare.
- Demo che funziona solo con cablaggi/hack non ripetibili e non documentati.

## Hardware e vincoli

| Componente | Ruolo |
|---|---|
| SO-101 leader + follower | già assemblati, calibrati e configurati con LeRobot |
| Raspberry Pi 5 | nodo "robot": follower + camere collegate, esegue il client robot |
| Desktop con RTX 3090 | training, policy server per l'inferenza, servizi (Grafana, K3s server, MCP) |
| Camera USB sul polso | osservazione per la policy |
| Camera USB frontale | osservazione per la policy |
| Terza camera USB (dell'utente) | ripresa del video demo (non usata dalla policy) — *mia scelta* |

- Esperienza ROS di base; Python a posto; lavoro da solo con AI.
- Il collo di bottiglia è il tempo davanti al robot (registrazione, valutazione), non il codice:
  tutto il software preparabile va pronto prima del 30/11.

## Approcci considerati

- **A. Pipeline di robot learning con LeRobot (scelto).** Teleop → dataset → ACT → inferenza → agente.
  Parte dalla configurazione esistente, è l'approccio dominante oggi, demo forte. Richiede GPU (disponibile) e camere (disponibili).
- **B. Stack classico ROS 2 + MoveIt + percezione.** Scartato: porting/setup del SO-101 su ROS 2 con esperienza base
  rischia di consumare metà settimana e butta via la configurazione LeRobot esistente.
- **A+B ibrido.** Scartato: non sta in una settimana da solo.

## Architettura

```
                         LAN
[Pi 5]  ───────────────────────────────  [Desktop RTX 3090]
 ├─ SO-101 follower (USB)                  ├─ LeRobot policy server (GPU, nativo)
 ├─ camera polso + frontale (USB)          ├─ training (nativo)
 ├─ LeRobot robot client (container K3s)   ├─ K3s server: Prometheus + Grafana, Fleet
 │    └─ endpoint metriche Prometheus      ├─ server MCP  ←── client AI (es. Claude Code)
 └─ K3s agent                              └─ repo Git (sorgente GitOps per Fleet)
```

### 1. Teleoperazione e raccolta dati
- Registrazione con `lerobot-record` (o equivalente nella versione LeRobot installata), leader → follower,
  camere polso + frontale. Dataset salvati in locale e pushati su Hugging Face Hub (pubblici).
- Compiti (*mia scelta*, modificabile):
  - **Compito 1:** prendere un cubo da una zona e metterlo in una scatola.
  - **Compito 2:** riprendere il cubo dalla scatola e rimetterlo nella zona di partenza
    (si concatena col compito 1 → demo continua e reset automatico tra i tentativi).
- **Condizioni del compito 1** (decise con l'utente): cubo in posizione casuale in una zona ~15×15 cm,
  orientamento circa costante; scatola fissa; successo = cubo rilasciato dentro la scatola; massimo 30 s per tentativo.
  Le dimostrazioni coprono tutta la zona in modo uniforme.
- **Colori** (*mia scelta*): nel dataset del compito 1 si alternano 2–3 cubi di colori diversi (uno alla volta),
  così la policy impara "un cubo qualsiasi" ed è più robusta. Più cubi presenti insieme solo nello stretch SmolVLA.
- ~50 episodi per compito come punto di partenza; aggiungerne se il successo è basso.
- Le camere vanno **fissate fisicamente** e non spostate tra registrazione e inferenza.

### 2. Training
- Policy **ACT** su RTX 3090, una per compito. Run lunghe la notte.
- Modelli pushati su Hugging Face Hub con versioni/tag.
- Stretch: **SmolVLA** fine-tuned con istruzioni testuali (un'unica policy condizionata dal linguaggio):
  più cubi colorati sul tavolo contemporaneamente, istruzione "metti il cubo rosso nella scatola".
  L'agente traduce la richiesta in chat nell'istruzione. Dataset dedicato da registrare.
- Tempo per il robot in preparazione (17–29 nov): confermato disponibile → il calendario resta com'è.

### 3. Inferenza
- **Inferenza asincrona LeRobot**: policy server sul desktop (GPU), robot client sul Pi.
  Il client invia osservazioni (camere + stato giunti), riceve chunk di azioni.
- Il client specifica quale policy caricare → cambiare policy = cambiare configurazione del client.
- **Fallback** se la catena Pi↔desktop dà problemi (latenza, banda camere, driver): follower e camere
  collegati direttamente al desktop. La demo del robot autonomo ha priorità sull'architettura.

### 4. Valutazione
- Script di valutazione: N tentativi per compito, esito registrato (successo/fallimento, durata),
  report in un file versionato. Opzionale: registrazione degli episodi di valutazione come dataset.
- **Fine episodio** (*mia scelta*): ACT non ha un segnale di "fatto" → un episodio autonomo dura un tempo
  fisso (timeout per compito) o termina con `stop`, poi il braccio torna in posa di riposo.
- **Esito** (*mia scelta*): nella valutazione ufficiale lo marca l'umano (tasto s/f nello script).
- **Reset** (*mia scelta*): nella valutazione ufficiale del compito 1 il cubo si riposiziona **a mano**,
  così il criterio minimo non dipende dal compito 2 (che è tagliabile). Il concatenamento 1→2 è solo per la demo.

### 5. Telemetria (Grafana)
- Exporter Prometheus nel robot client (o sidecar) con: posizione, carico, tensione, temperatura per servo;
  latenza del loop, frequenza azioni, policy attiva, stato (idle/running/stopped).
- Prometheus + Grafana su K3s sul desktop, una dashboard versionata nel repo.
- Le camere live si guardano con il visualizzatore integrato di LeRobot (rerun), non in Grafana.
- *Scelta:* niente MQTT/Timescale — senza la narrativa IoT è complessità inutile.

### 6. K3s + Fleet
- K3s server sul desktop, Pi come agent. Robot client in container sul Pi con accesso a `/dev` (seriale + video).
- Policy server e training restano **nativi** sul desktop (evitare il setup GPU in K3s) — *mia scelta*.
- Fleet (standalone, senza Rancher) sincronizza dal repo: manifest del robot client, catalogo
  "compito → policy (repo HF + revisione)", Prometheus/Grafana. Demo: commit che aggiorna la revisione
  di una policy → il Pi usa la nuova versione.

### 7. Agente AI (MCP)
- Server MCP sul desktop con tool:
  - `list_tasks` — compiti disponibili dal catalogo
  - `run_task(name)` — avvia un episodio autonomo del compito
  - `stop` — ferma subito il robot (arresto di emergenza software)
  - `go_home` — porta il braccio in posa di riposo
  - `get_status` — stato e telemetria principale (temperature, policy attiva, ultimo esito)
  - `get_snapshot` — foto dalla camera frontale, così l'agente (modello con visione) può giudicare
    da solo se il compito è riuscito e decidere il passo successivo — *mia scelta*
- `run_task` è bloccante fino al timeout del compito (o a `stop`) e restituisce durata ed esito "sconosciuto";
  l'agente usa `get_snapshot` per valutare.
- Client: un assistente AI qualsiasi con supporto MCP (es. Claude Code).
- Il server MCP deve funzionare anche senza K3s (backend che lancia il client direttamente), così
  l'agente non dipende dall'infrastruttura opzionale.

### 8. AI privata: OS SUSE e LLM locale (richiesta dell'utente)
- **OS del Pi:** priorità a un OS SUSE (SLE Micro o openSUSE MicroOS/Tumbleweed aarch64) con K3s; timebox
  **mezza giornata** in preparazione, poi fallback Raspberry Pi OS 64-bit.
- **Agente con LLM locale:** tutto gira in casa, nessun servizio cloud nel ciclo di controllo né nell'agente.
  - Ollama + Open WebUI sul desktop (i componenti dello stack SUSE AI; usare le immagini SUSE AI se l'accesso
    è semplice, altrimenti upstream — *mia scelta*).
  - Modello open con **tool calling + visione** (serve per `get_snapshot`), che stia nella 3090 insieme al
    policy server (ACT occupa poco). Scelta del modello lasciata al builder, con una prova comparativa.
  - Il server MCP espone anche un trasporto HTTP; Open WebUI lo usa come tool server.
  - Claude Code resta il client di riferimento e il fallback: se il modello locale non è affidabile,
    la demo usa Claude e il confronto locale/cloud finisce nel README.

### Sicurezza
- Arresto immediato sempre disponibile (tool `stop` + scollegare l'alimentazione del follower).
- Limiti di coppia/velocità ragionevoli in inferenza; nessuna persona nello spazio di lavoro durante i test autonomi.

## Calendario

| Quando | Obiettivo | Fatto quando |
|---|---|---|
| 17–29 nov (prep) | OS sul Pi; camere fissate; LeRobot su Pi e desktop; registrazione dataset compito 1; primo training ACT | Policy compito 1 addestrata |
| Lun 30 | Inferenza asincrona Pi↔desktop; valutazione compito 1; registrazione compito 2 | Il braccio fa il compito 1 da solo |
| Mar 1 | Training compito 2 (notte); più dati/iterazione compito 1; exporter + Grafana | Dashboard live, successo compito 1 ≥ 7/10 |
| Mer 2 | K3s + robot client in container + Fleet | Un commit cambia la policy sul Pi |
| Gio 3 | Server MCP + agente | Compiti lanciati da chat |
| Ven 4 | Video demo, README, pagina hackweek | Repo pubblico + video |

Ordine di taglio se si è in ritardo: Fleet → LLM locale → K3s → Grafana → compito 2. Mai tagliare compito 1 + valutazione.

## Lasciato al builder

- Versione di LeRobot e comandi esatti (le CLI cambiano tra versioni: verificare su quella installata).
- OS del Pi: vedi sezione 8.
- Risoluzione/FPS delle camere, iperparametri ACT, numero di step di training.
- Struttura del repo, linguaggio dell'exporter, implementazione del server MCP (Python consigliato per coerenza con LeRobot).
- Come il server MCP avvia/ferma gli episodi (Job K3s o processo locale), purché `stop` sia immediato.
