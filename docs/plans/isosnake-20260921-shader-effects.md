# Piano — Effetti shader per Isosnake

Idea: *vorrei qualche effetto shader a questo gioco dentro index.html (snake isometrico)*.
Decisioni dall'intervista grill del 2026-09-21 (4/4 chiuse): **pacchetto completo** (pavimento animato + post-processing arcade + serpente iridescente) · intensità **arcade equilibrato** · priorità **desktop-first**.

## Panoramica

Tutto dentro `Z:/Lavorazioni/2026/ai/isosnake/index.html` (1011 righe, single-file), senza nuove dipendenze: solo `three` 0.160 e `three/addons` già caricati via import map. Fatti verificati sul codice: la pipeline è già `EffectComposer` con `RenderPass → UnrealBloomPass(0.5, 0.45, 0.6) → OutputPass`; il pavimento è `PlaneGeometry` + `MeshStandardMaterial` (`receiveShadow`) con griglia e bordo come `LineSegments` statici; la snake è un pool di `BoxGeometry` con due materiali standard condivisi (`matHead`/`matBody`, emissive) e ghost mode via `transparent/opacity`; il game over ha già camera shake (`G.shake`) e un delay di 900 ms sull'overlay; il power-up Slow-mo usa `CFG.slowFactor = 0.65`. Gli effetti shader si integrano su questa base senza toccare la logica di gioco (`tick`, regole, power-up, audio). Tre moduli:

1. **Pavimento neon animato** — shader custom sopra il pavimento (griglia vivente, onda dal cibo, ripple sugli eventi).
2. **Serpente iridescente** — gradiente scorrevole + pulse testa→coda, iniettato via `onBeforeCompile` sui materiali standard esistenti (ombre, illuminazione e ghost mode restano intatti).
3. **Post-processing arcade** — un `ShaderPass` unico (aberrazione cromatica, vignetta, scanline, glitch) con trigger sugli eventi di gioco; glitch "cinematografico" al game over.

Intensità **arcade equilibrato**: ogni effetto ha valori base discreti e "kick" temporanei sugli eventi (mangi, power-up, livello, morte), mai saturi in modo permanente. **Desktop-first**: pixelRatio già cap a 2, shader leggeri (niente loop pesanti, niente texture procedurali costose), nessun downgrade mobile richiesto ma senza esplosione di costo.

## Modulo 1 — Pavimento neon animato

**Struttura.**

- **Conservare** il piano standard sotto (per `receiveShadow` e fog).
- **Sostituire** il `LineSegments` griglia + bordo (`gridGeo`/`borderGeo` + `LineBasicMaterial`) con un **overlay plane** (stessa `PlaneGeometry`, `y` leggermente sopra, `transparent`, `depthWrite:false`) con un `ShaderMaterial` custom che disegna griglia + bordo + dinamiche.
- Rimuovere le geometrie e i materiali `LineSegments` superflui (sostituiti dallo shader).

**Uniformi dello shader pavimento:**

| Uniform | Tipo | Ruolo |
| --- | --- | --- |
| `uTime` | float | tempo scalato (vedi Modulo 4) |
| `uFoodPos` | vec2 | posizione cibo (onda radiale di richiamo) |
| `uHeadPos` | vec2 | posizione testa (alone leggero che segue la snake) |
| `uRipplePos` | vec2 | centro dell'ultima onda (cibo mangiato / power-up / centro per level-up) |
| `uRippleT` | float | età dell'onda (0→1, oltre 1 spenta) |
| `uRippleColor` | vec3 | colore dell'onda (magenta=cibo, colore power-up, ciano=livello) |
| `uGridA` / `uGridB` | vec3 | colori base griglia / bordo |

**Fragment (idee, valori da tune a "equilibrato"):**

- Griglia AA: linea su `fract(uv*20)` con `fwidth` per anti-alias, intensità base ~0.35.
- **Onda radiale dal cibo**: anello pulsante lento centrato su `uFoodPos` (sinusoidale, raggio crescente, ampiezza ~0.15) — guida l'occhio verso il cibo.
- **Ripple evento**: anello espandente `smoothstep` guidato da `uRippleT` (raggio `t*14`, spessore che si assottiglia), colore `uRippleColor` — il pavimento "reagisce" a mangiare/power/level-up.
- **Border pulse**: il bordo esterno respira lentamente + flash sul wrap (usa l'evento `wrapFx` già esistente in `G`).
- **Alone testa**: gaussiana morbida sotto la testa (raggio ~2 celle, intensità ~0.12), ciano-verde.
- `uHeadPos`/`uFoodPos` aggiornati ogni frame in `updateVisuals` (le posizioni sono già calcolate lì).

**Wiring eventi** (funzioni esistenti, una riga ciascuna):
- `tick()` → `willEat`: `fxRipple(pos, magenta)`.
- `applyPower(type)`: `fxRipple(pos, colore power-up)`.
- `fxLevelUp()`: `fxRipple(centro, ciano)` + flash bordo.
- wrap (`G.wrapFx`): flash del border.

`fxRipple(pos, color)` = setter di `uRipplePos/uRippleT/uRippleColor` (sempre una sola onda attiva: la nuova sostituisce).

## Modulo 2 — Serpente iridescente

**Approccio scelto: `onBeforeCompile`** su `matHead`/`matBody` (non un `ShaderMaterial` puro): si conservano illuminazione, `castShadow`, e la logica ghost esistente (`setGhost` che tocca `transparent`/`opacity`), e il bloom continua a vedere l'emissive.

**Per segmento:** ogni mesh del pool (`makeSeg`) ottiene la **sua istanza di materiale** (clone di `matBody`/`matHead`) con una uniform extra `uIdx` (0 = testa, cresce verso la coda) e `uCount` (lunghezza corrente). `setSnakeLength` e `applyPower('shrink')` aggiornano `uCount` e ri-normalizzano gli `uIdx`. Ghost: `setGhost` deve poi iterare sulle istanze (oggi tocca i 2 materiali condivisi → si estende al loop su `segMeshes`).

**Iniezione nel fragment:**
- **Gradiente scorrevole**: mix verde→ciano lungo il corpo: `mix(corpo, testa, smoothstep(0,1, fract(uIdx/uCount - uTime*0.25)))` — un'onda di colore che scivola dalla testa verso la coda.
- **Pulse di luminosità**: picco di emissive extra quando `fract(uIdx/uCount - uTime*0.5)` è vicino a 0 (pulsante che viaggia testa→coda, ~2 Hz).
- **Testa**: glow più intenso + leggero shimmer.
- Valori "equilibrati": delta colore contenuto rispetto ai colori attuali, emissive extra max ~+0.4 — la snake resta verde-neon riconoscibile, non diventa arcobaleno.

**Note di compatibilità:**
- `onBeforeCompile` + `customProgramCacheKey` per non far collidere il program cache con i materiali standard non modificati.
- Occhi (sferette `matEye`) invariati.
- Durante **Ghost** (opacity 0.42) l'iridescenza resta visibile ma attenuata naturalmente dall'opacity.

## Modulo 3 — Post-processing arcade

**Pipeline** (ordine): `RenderPass → UnrealBloomPass → **FXPass (nuovo)** → OutputPass`. Un solo `ShaderPass` custom (import `three/addons/postprocessing/ShaderPass.js`, stesso pattern delle pass già importate) con tutti gli effetti, per limitare i passaggi full-screen aggiuntivi a uno.

**Uniformi:**

| Uniform | Base (equilibrato) | Kick evento |
| --- | --- | --- |
| `uAberr` | 0.0008 (uv) | +0.0025 su ~150 ms al mangiare; +0.003 su power-up |
| `uGlitch` | 0 | → 1.0 in 100 ms al game over, decay in ~0.9 s (allineato all'esplosione esistente da 900 ms) |
| `uVign` | 0.32 | invariato |
| `uScan` | 0.05 (opacità scanline) | invariato |
| `uTime` | tempo scalato | — |

**Fragment:**
- **Aberrazione cromatica**: offset RGB radiale dal centro proporzionale a `uAberr * dist` — dà "velocità" al frame.
- **Vignetta**: scuritura morbida ai bordi (radiale `smoothstep`).
- **Scanline**: `sin(uv.y * height * π)` attenuatissimo (0.05) + rolling molto lento — alone CRT discreto.
- **Glitch**: quando `uGlitch > 0` — slice orizzontali spostate (offset per riga, hash su `uTime`), RGB split amplificato ×4, leggera distorsione di posizione; tutto moltiplicato per `uGlitch`, quindi a 0 è un no-op economico.

**Wiring:**
- `doGameOver()` → `fx.triggerGlitch()` (decay esponenziale nel loop) — coesiste col `G.shake` già presente.
- `tick()` mangia → `fx.bumpAberration(0.0025)`.
- `applyPower()` → `fx.bumpAberration(0.003)`.
- Kick implementati come target con decay esponenziale nel `frame()` (stesso pattern dello `G.shake` già nel codice): un oggetto `FX` centralizzato (vedi Modulo 4).

## Modulo 4 — Tempo globale e intensità

- **`uTimeScale`** globale: normalmente 1; con power-up **Slow-mo** attivo → `0.65` (uguale a `CFG.slowFactor`), con transizione morbida (lerp ~200 ms). Guida `uTime` di pavimento, serpente e FXPass: tutto il mondo "rallenta" insieme al gameplay — effetto coeso.
- **Oggetto `FX`** unico (accanto allo stato `G`): detiene i riferimenti alle uniformi dei 3 moduli, i target di decay (`aberration`, `glitch`, `timeScale`) e gli helper `fxRipple / bumpAberration / triggerGlitch / setTimeScale`. Nel `frame()` un passo `FX.update(dt)` fa i decays e scrive le uniformi — nessun aggiornamento sparpagliato.
- **Scala di intensità (equilibrato)**: tutti i valori "base" sopra sono il punto di partenza; i "kick" durano 150–900 ms. Regola di tuning: in steady-state la scena deve essere ~10-15% più "viva" di oggi, non trasformata; i picchi si vedono solo negli eventi.

## Iterazioni di implementazione

1. **Modulo 1**: overlay shader pavimento (griglia AA + border + onda cibo + ripple + alone testa), rimuovo i `LineSegments` griglia/bordo, wiring `fxRipple` su mangia/power/level/wrap.
2. **Modulo 2**: per-segment material instances + `onBeforeCompile` iridescenza e pulse; estendo `setGhost` alle istanze; verifico ombre e ghost.
3. **Modulo 3**: `FXPass` (aberrazione, vignetta, scanline, glitch) in pipeline tra bloom e Output; wiring game over + kick su eventi; decay nel loop.
4. **Modulo 4**: `FX.update` centralizzato + `uTimeScale` slow-mo; pass di tuning finale dei valori su desktop.

Ordine scelto perché 1 e 2 sono indipendenti tra loro e da 3; 4 chiude e allinea i tempi.

## Verifica

- Aprire `index.html` in desktop browser, giocare 3 partite complete:
  - Pavimento: onda segue il cibo; ripple magenta al mangiare; ripple colorato al power-up; flash ciano al level-up; flash bordo al wrap; ombre dei segmenti ancora visibili sul pavimento.
  - Snake: gradiente e pulse scorrono testa→coda; colore ancora riconoscibile come prima; **Ghost** resta semi-trasparente; **Shrink** non fa sparire l'iridescenza (`uIdx` ri-normalizzato).
  - Post: aberrazione percepibile come micro-kick al mangiare; al game over glitch ~0.9 s coeso col camera shake; vignetta/scanline presenti ma non invadenti.
  - Slow-mo: pavimento, snake e post rallentano insieme per 6 s.
- Performance (desktop-first): 60 FPS stabili a pixelRatio 2 su GPU desktop; nessun nuovo asset; un solo passaggio shader aggiuntivo (FXPass).
- Regressione: menu/pausa/help invariati; su touch il gioco resta giocabile (shader attivi ma leggeri).
- Edge case: ripple durante pausa (deve congelarsi col tempo scalato/fermato), glitch + overlay game over a 900 ms (niente flicker sopra l'overlay), ghost + iridescenza insieme.

## Interview transcript

- **fx-package** (Effetti shader) — status: `answered`
  - Domanda: *Quali effetti shader vuoi aggiungere? (il gioco ha già bloom neon + particelle additive + fog)*
  - Opzioni:
    1. `floor` — *Pavimento neon animato* — Griglia custom in movimento via ShaderMaterial: pulse radiale dal cibo, increspature, onde. Impatto visivo massimo sul campo.
    2. `post` — *Post-processing arcade* — ShaderPass full-screen: aberrazione cromatica, glitch alla morte, scanline/CRT, vignetta.
    3. `snake` — *Serpente iridescente* — Shader del corpo: gradiente scorrevole lungo i segmenti + pattern/pulse animato.
    4. `all` — *Pacchetto completo* — I tre effetti sopra insieme (pavimento + post-processing + serpente).
  - Raccomandata: `floor` — *Massimo impatto per minimo rischio e senza cambiare il gameplay.*
  - Scelta utente: **Pacchetto completo**
  - Motivo: (nessuno indicato)
  - Nota di status: nessuna
- **intensity** (Intensità) — status: `answered`
  - Domanda: *Quale intensità visiva vuoi? (quanto devono 'colpire' gli effetti)*
  - Opzioni:
    1. `balanced` — *Arcade equilibrato* — Effetti ben visibili ma non invadenti; si legge ancora bene la griglia e la partita.
    2. `subtle` — *Sottile/ambientale* — Tocco leggero ed elegante, quasi di sfondo.
    3. `flashy` — *Flashy al massimo* — Tutto spinto, molto cinematografico, rischio di saturare lo schermo.
  - Raccomandata: `balanced` — *Miglior comprometto tra impatto e giocabilità.*
  - Scelta utente: **Arcade equilibrato**
  - Motivo: (nessuno indicato)
  - Nota di status: nessuna
- **perf** (Performance) — status: `answered`
  - Domanda: *Priorità di performance per gli shader?*
  - Opzioni:
    1. `adaptive` — *Adattivo auto* — Riduce/attiva i shader in base a FPS/mobile (come già fanno le particelle con particleScale).
    2. `desktop` — *Desktop-first* — Massima qualità, presuppone desktop con GPU.
    3. `mobile` — *Mobile-first* — Shader molto leggeri, presuppone telefono.
  - Raccomandata: `adaptive` — *Coesistente con l'approccio già usato nel file (particleScale dinamico).*
  - Scelta utente: **Desktop-first**
  - Motivo: (nessuno indicato)
  - Nota di status: nessuna
- **write-plan** (Chiusura) — status: `answered`
  - Domanda: *Decisioni complete (pacchetto completo · arcade equilibrato · desktop-first). Scrivo il piano di implementazione?*
  - Opzioni:
    1. `yes` — *Sì, scrivi il piano* — Genera il piano dettagliato con le sezioni derivate dai contenuti.
    2. `review` — *Prima rivedo lo stato* — Apri lo stato completo (JSON) prima di procedere.
  - Raccomandata: `yes` — *Tutte le dipendenze sono risolte; nessun'altra decisione aperta.*
  - Scelta utente: **Sì, scrivi il piano**
  - Motivo: (nessuno indicato)
  - Nota di status: nessuna
- Note utente: (nessuna — `notes` vuoto)
