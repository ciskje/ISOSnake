# Piano di implementazione — Isosnake (Snake isometrico neon in una pagina web)

Idea: *voglio un gioco snake isometrico in una pagina web*.
Tutte le decisioni seguenti derivano dall'intervista grill del 2026-09-21 (14/14 domande chiuse) + richieste dirette dell'utente: schermata Help nel gioco ed effetti particellari.

## Panoramica

Un gioco Snake completo e giocabile, reso in 3D reale con Three.js, in un **unico file HTML** con CSS e JS inline. Estetica neon su sfondo scuro, griglia 20×20 con muri a attraversamento (wrap), power-up temporizzati, progressione a livelli per soglia di punteggio con ostacoli progressivi, **effetti particellari** sui momenti chiave, high-score persistente in `localStorage`, menu/pausa/game-over, **schermata Help** (controlli, regole, power-up, livelli), SFX sintetici WebAudio con mute, controlli tastiera (frecce + WASD) e touch (swipe). Su desktop la camera è orbitabile (OrbitControls); su touch è fissa per non confliggere con gli swipe.

## Stack e struttura del file

- **Un solo file**: `index.html` in `Z:/Lavorazioni/2026/ai/isosnake/`.
- **Three.js da CDN** con import map (no build, no npm):
  ```html
  <script type="importmap">
  { "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
      "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"
  } }
  </script>
  ```
  Import di `OrbitControls` da `three/addons/controls/OrbitControls.js`.
- **JS modulare dentro un unico `<script type="module">`**, organizzato in sezioni commentate: `config`, `state`, `scene`, `entities`, `game` (loop e regole), `powerups`, `levels`, `input`, `audio`, `ui`.
- **CSS inline** in `<style>`: HUD, menu, overlay, pulsante mute, responsive mobile.
- Nessun asset esterno: geometrie e materiali procedurali, audio sintetico.

## Scena e rendering

- **Renderer**: `WebGLRenderer` con `antialias`, `setPixelRatio(min(devicePixelRatio, 2))`, `outputColorSpace = SRGBColorSpace`, tone mapping ACES per far brillare il neon.
- **Scena**:
  - Pavimento: piano 20×20 con `MeshStandardMaterial` scuro (quasi nero, metalness medio) + griglia di linee neon (o `GridHelper` custom colorato) con emissive.
  - **Snake**: un `Mesh` per cella (BoxGeometry 0.9) con `MeshStandardMaterial` emissive; la testa è un cubo leggermente più grande con "occhi" (due sfere piccole) e colore distinto. Ombre: una sola `DirectionalLight` con ombre abilitate + `AmbientLight`/`HemisphereLight` di riempimento.
  - **Cibo**: ottaedro o sfera pulsante (scale sinusoide) emissive magenta/ciano.
  - **Power-up**: icone distinguibili (toro, cono, cubo traslucido, piramide) con colore proprio e bobbing, anello di "tempo restante" (anello che si svuota) sopra l'oggetto.
  - **Ostacoli**: cubi più alti (h=1.2) colore viola/arancio neon.
- **Effetto neon**: materiali emissivi saturi + `UnrealBloomPass` (EffectComposer, `three/addons/postprocessing/*`) con soglia alta (solo le superfici emissive bloomano). Parametri: strength ~0.9, radius ~0.6, threshold ~0.75 — da ritoccare per mantenere 60 FPS su mobile.
- **Camera**:
  - Vista standard isometrica: posizione `(~14, ~14, ~14)` guardando il centro della griglia.
  - **Desktop**: `OrbitControls` abilitato (drag = rotazione, rotellina = zoom, con `minDistance/maxDistance` e `maxPolarAngle` per non andare sotto il pavimento).
  - **Rilevamento touch**: `('ontouchstart' in window)` o Media Query → su touch **niente OrbitControls**, camera fissa in isometrica, swipe riservati al gameplay.
  - **Reset vista**: all'inizio di ogni nuova partita la camera torna alla vista isometrica standard (decisione `orbit-persistence: reset`).

## Regole e stato del gioco

- **Griglia**: 20×20 celle, coordinate `(x, y)` con z=0; la snake vive sul piano.
- **Stati**: `menu → playing → (paused) → gameover → playing`. Transizioni da tastiera (Enter/Space), tap su touch, pulsanti a schermo.
- **Movimento**: tick-based; base ~7 celle/sec, +~0.5 celle/sec per livello (con pavimento minimo di velocità). Ogni tick la testa si sposta di 1 cella; la coda si ritrae (salvo crescita).
- **Muri**: **wrap** — la testa che esce da un bordo rientra dal lato opposto (la mesh attraversa o teletrasa con un breve fade; scelta implementativa: teletraso istantaneo con particella/glow per chiarezza visiva).
- **Collisione letale**: solo con il proprio corpo (salvo Ghost) e con gli ostacoli (salvo Ghost).
- **Cibo**: 1 attivo alla volta in cella libera (non su snake, ostacoli, power-up); al mangiare: +1 lunghezza, +punti (10 × moltiplicatore 2x se attivo).
- **Anti-inversione**: la coda di input scarta la direzione opposta a quella corrente (max 1 direzione in coda per tick, buffer di 2).

## Power-up

Spawn casuali (probabilità per tick, es. ~4% quando non ce n'è uno attivo), in cella libera, **scadenza temporizzata** (es. 8–12 s, indicata dall'anello di countdown). Set completo:

| Power-up | Colore | Effetto | Durata |
| --- | --- | --- | --- |
| **2x Punti** | oro | moltiplica i punti del cibo | ~10 s |
| **Slow-mo** | ciano | -35% velocità tick | ~6 s |
| **Ghost** | verde traslucido | la snake attraversa sé stessa e gli ostacoli (la mesh diventa semi-trasparente) | ~6 s |
| **Shrink** | rosa | -3 segmenti (min. lunghezza 3) | istantaneo |

Mantenere un solo power-up attivo nella scena alla volta; effetti non si sommano (nuovo spawn sostituisce il tipo, estende la durata).

## Livelli e ostacoli

- **Progressione**: soglia di **punteggio** — ogni 500 punti → livello successivo (il 2x accelera la salita: accettato esplicitamente).
- **A ogni livello**: +velocità, compaiono **2–6 blocchi ostacolo** neon (cubi alti) in celle libere, mai sulla snake né adiacenti alla testa in modo da non uccidere all'istante; i livelli alti hanno più ostacoli (2 + livello, capped a 6).
- **Game over**: collisione corpo/ostacolo (senza Ghost). Overlay con punteggio, livello raggiunto, high-score, "Gioca ancora".

## Punteggio e high-score

- Punti base 10/cibo, ×2 con oro attivo.
- **High-score persistente**: `localStorage` (`isosnake.highscore.v1`) con punteggio e livello massimo; mostrato nel menu e nell'overlay game-over ("nuovo record" evidenziato).
- Stato mute anche persistente (`isosnake.muted`).

## Audio (WebAudio sintetico)

- Nessun file: `OscillatorNode` + `GainNode` per SFX brevi.
- Eventi: **mangi** (blip ascendente), **power-up** (arpeggio), **livello su** (crescendo), **game over** (discendente), **tick** opzionale disattivato di default.
- **Pulsante mute** sempre visibile (icona altoparlante), stato salvato in `localStorage`; il `AudioContext` parte al primo gesto utente (policy autoplay).

## UI e layout

- **HUD** (DOM, sopra il canvas): punteggio a sinistra, livello a destra, barra/countdown power-up attivo in alto al centro; pulsante mute in un angolo.
- **Menu iniziale**: titolo "ISO SNAKE", pulsanti Gioca / Mute, high-score, hint controlli (frecce/WASD o swipe).
- **Pause**: `P` o `Esc` (desktop), tap su icona (touch).
- **Schermata Help**: pulsante `?`/"Help" nel menu e nell'HUD (icona) + tasto `H`. Apre un overlay con:
  - **Controlli**: frecce/WASD per la direzione, `P`/`Esc` pausa, `H` help, `Enter` per confermare; su touch swipe in una direzione.
  - **Obiettivo**: mangia per crescere e salire di livello; evita il tuo corpo e gli ostacoli; i bordi sono attraversabili (wrap).
  - **Power-up**: tabella icona+colore+effetto+durata (2x oro, Slow-mo ciano, Ghost verde, Shrink rosa).
  - **Livelli e punteggi**: ogni 500 punti → livello su (+velocità, +2–6 ostacoli); high-score salvato sul dispositivo.
  - Chiudibile con `Esc`, tap fuori, o pulsante "Chiudi"; la partita resta in pausa finché l'Help è aperta (se aperta da in-play).
- **Responsive**: canvas a tutto schermo; HUD con `clamp()` su font-size; su mobile niente OrbitControls e swipe full-screen (meno l'HUD).
- Accessibilità di base: `tabindex` sul container, aria-label sui pulsanti, contrasto elevato già dato dal tema neon.

## Effetti particellari

Sistema particellare leggero su `THREE.Points` + `BufferGeometry` con attributi per posizione/velocità/età/colore, aggiornato nel loop (nessuna dipendenza esterna). Pool riciclabile (es. 512 particelle) per non allocare in runtime.

Eventi coperti:

| Evento | Effetto |
| --- | --- |
| **Mangi** | Burst di 12–20 scintille dal cibo, colore del cibo, gravità leggera, fade 0.4 s |
| **Power-up raccolto** | Burst radiale del colore del power-up + anello d'onda che si espande e svanisce |
| **Power-up scade** | Piccola dissolvenza particellare sulla testa della snake |
| **Livello su** | Ondata di particelle lungo il perimetro della griglia + flash del pavimento |
| **Ostacolo spawn** | Particelle che convergono nel punto di spawn |
| **Wrap** | Scia di particelle che segue la testa per ~6 frame al teletraso |
| **Game over** | Esplosione della testa (30–40 particelle multicolore) + camera shake breve (0.25 s) |

Parametri condivisi: `size` decrescente con l'età, `additive blending` per il glow, `depthWrite=false`. Su mobile ridurre il conteggio burst del 50% se FPS < 50 (check automatico).

## Piano di implementazione (iterazioni)

1. **Scheletro**: `index.html` con import map, scena Three.js, pavimento + griglia neon, bloom, camera isometrica standard, resize handler.
2. **Gameplay core**: loop tick, snake (testa + corpo), cibo, crescita, wrap, collisione corpo, game over, HUD punteggio.
3. **Input**: tastiera (frecce + WASD, buffer anti-inversione) e touch swipe; rilevamento piattaforma.
4. **Camera**: OrbitControls su desktop (limiti distanza/angolo), fissa su touch, reset a ogni nuova partita.
5. **Power-up**: spawn, countdown, 4 effetti, anello di durata, UI HUD.
6. **Livelli**: soglia punteggio, +velocità, spawn ostacoli 2–6 per livello, verifica celle libere.
7. **Audio**: SFX WebAudio + mute persistente.
8. **Particellari**: pool riciclabile + effetti per mangiare, power-up, livello, ostacolo, wrap, game over; tuning mobile.
9. **Menu e polish**: menu iniziale, pause, overlay game over, high-score localStorage, **schermata Help** (controlli, regole, power-up, livelli), hint controlli, tuning bloom/velocità su mobile.

## Verifica

- Aprire `index.html` in un browser desktop: giocare 3 partite complete, verificare wrap, power-up, livello 2 (ostacoli), game over, high-score che sopravvive al refresh.
- Help: aprirla da menu, da HUD in gioco (il gioco va in pausa) e con `H`; chiudere con `Esc`/tap; contenuto leggibile su mobile.
- Particellari: ogni evento della tabella produce il suo effetto; nessun calo FPS su mobile (burst ridotti automaticamente sotto 50 FPS).
- Test touch: viewport mobile (DevTools) — swipe funzionanti, niente OrbitControls, HUD leggibile.
- Performance: 60 FPS con bloom attivo su mobile mid-range (altrimenti ridurre `bloom strength`/`pixelRatio`).
- Edge case: Shrink su snake corta (min 3), Ghost su ostacolo e su sé, power-up spawnato in cella occupata (deve riprovare), pausa durante un effect (i timer si congelano).

## Interview transcript

- **rendering** (Tecnologie): *Con quale tecnologia rendi la scena isometrica?* → **Three.js (WebGL) con cubi 3D veri**
- **structure** (Tecnologie): *Come organizzi il codice?* → **Un solo file HTML con CSS e JS inline**
- **scope** (Gameplay): *Quale scope di gameplay vuoi nel primo taglio?* → **Classico + power-up, livelli, score high-score persistente e menu**
- **controls** (Gameplay): *Quali input supporti?* → **Tastiera + swipe/direzioni su touch**
- **look** (Estetica): *Quale estetica visiva preferisci?* → **Neon/glow: sfondo scuro, bordi luminosi, effetto glow**
- **camera** (Scena 3D): *Come gestisci la camera 3D?* → **OrbitControls: rotazione/zoom col drag, gameplay solo su tasti**
- **grid** (Scena 3D): *Quante celle fa la griglia di gioco?* → **20x20**
- **walls** (Gameplay): *Cosa succede quando la testa tocca il bordo della griglia?* → **Attraversamento: esce da una parte e rientra dall'altra**
- **powerups** (Gameplay): *Quale set di power-up implementi (apparsi casualmente, con scadenza temporizzata)?* → **Set completo: 2x punti (oro), Slow-mo, Ghost, Shrink (taglia -3)**
- **levels** (Gameplay): *Come progressi i livelli?* → **Soglia di punteggio (es. ogni 500 punti)**
- **obstacles** (Gameplay): *I livelli introducono ostacoli statici nella griglia?* → **Sì: da livello 2 in poi compaiono blocchi neon (2-6 per livello, mai sulla snake)**
- **audio** (Gameplay): *Che audio vuoi?* → **Sintetici via WebAudio (mangi, power-up, game over) + pulsante mute**
- **touch-orbit-conflict** (Scena 3D): *Con OrbitControls il drag tocca gli swipe di gioco su touch. Come lo risolvi?* → **Desktop: orbita/zoom col drag. Touch: camera fissa + swipe per giocare**
- **orbit-persistence** (Scena 3D): *La vista orbitata deve sopravvivere tra partite e menu?* → **No: al nuovo gioco torna l'isometrica standard**
- **write-plan** (Convergenza): *Tutte le decisioni sono chiuse. Scrivo il piano di implementazione?* → **Sì, genera il piano**
- **(nota post-piano)**: *genera anche una pagina di help nel gioco* → aggiunta sezione **Schermata Help** a UI e layout, iterazione 8 e verifica.
