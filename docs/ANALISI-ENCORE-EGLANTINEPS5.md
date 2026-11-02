# Pokopia su PS5: Eden Encore contro EglantinePS5

Studio del 2 novembre 2026, ramo `perf/fast-first-pipelines` di questo fork.

## Il sintomo

- **EglantinePS5** (core Eglantine `3140648`): Pokopia è senza glitch grafici, ma ogni tanto
  si blocca per 1–2 secondi.
- **Eden Encore** (build personale `01.000.041`, cartella `PPSA99008`): niente blocchi, ma il
  core è più vecchio e la grafica è meno corretta.

## La causa: le pipeline grafiche compilate durante il gioco

Ogni combinazione nuova di shader e stato grafico diventa una *pipeline* Vulkan, e il driver
della console (RADV di Mihawk, compilatore ACO) la deve compilare prima che il disegno parta.
Quando una zona o un effetto ne porta decine tutte insieme, il tempo di compilazione si somma.

**EglantinePS5.** Nel core Eglantine fissato da EglantinePS5, i giochi Switch 2 (motore delle macro
nativo, `native_turing`) non saltano mai un disegno la cui pipeline non è pronta:
`PipelineCache::BuiltPipeline` restituisce la pipeline anche se non è costruita, e il worker
del renderer la aspetta (`build_condvar.wait` registrato in `GraphicsPipeline::ConfigureDraw` e
`ComputePipeline::Configure`). È una scelta del core (commit "build pipelines synchronously for
native titles", 30/10, presa da Eden-NS2): niente oggetti mancanti, ma mentre ACO compila con
ottimizzazione piena alla GPU non arriva lavoro nuovo, e si vede un blocco.

**Encore.** Il binario di Encore che hai compilato contiene un core Eglantine di fine ottobre:
- ha "Runtime pipelines in the last 10 s", `EDEN_NX2_SPLIT` e `EDEN_NX2_MME_JIT`;
- non ha `EDEN_NX2_SHADER_REUSE`, "Draw skipped (" né le *read traps*.

È quindi precedente alla costruzione sincrona per i titoli nativi. Su quel core Encore fa due cose:
1. **Shader asincroni attivi in tutti i profili Vulkan** (`headless/encore_performance_policy.h`):
   un disegno senza pipeline pronta viene saltato invece di aspettare. Il blocco sparisce, ma per
   qualche frame mancano degli oggetti: da qui parte dei glitch.
2. **Fast-first pipelines** (`headless/dev_vulkan.h`, `tools/prepare-vulkan-port.py`; nel
   binario c'è la stringa "Optimized pipeline rebuild failed"): la prima volta la pipeline è
   compilata *senza* ottimizzazioni (`VK_PIPELINE_CREATE_DISABLE_OPTIMIZATION_BIT`, con ACO
   molto più veloce), poi un thread in background la ricompila ottimizzata.

Altre differenze di impostazioni non spiegano il sintomo: VRAM `Aggressive`, presentazione
asincrona, cache shader su disco e flushing reattivo sono uguali nei due progetti.

## Cosa è stato portato su EglantinePS5

Il meglio dei due: il disegno continua ad aspettare la sua pipeline (niente glitch, come
EglantinePS5), ma l'attesa è molto più corta (come la fast-first di Encore).

1. **Fast-first pipelines nel core**: `tools/eglantine-patches/0010-fast-first-pipelines.patch`.
   - Vale per le pipeline grafiche e compute. Gli shader compute enormi dei giochi Switch 2
     (fino a 28.000 istruzioni, vedi `NX2_NOTES.md` riga 121) sono i più lenti da compilare.
   - Una pipeline costruita mentre il gioco gira è compilata subito senza ottimizzazioni, poi
     di nuovo ottimizzata da un thread a bassa priorità (`VkPipelineOptimizer`). I bind
     successivi passano alla versione ottimizzata appena è pronta; le due varianti hanno lo
     stesso layout, quindi descrittori e push constant restano validi.
   - Le pipeline caricate dalla cache su disco all'avvio sono compilate ottimizzate subito, come
     prima. La variante non ottimizzata non finisce nella cache su disco (dura una sessione).
   - Alla chiusura del gioco: l'ottimizzatore è dichiarato fra le cache e i worker delle
     pipeline, quindi si ferma dopo i worker che gli mettono lavoro in coda e prima delle
     pipeline su cui lavora.
2. **Interruttore `fast_pipelines`** (attivo di default) in `headless/settings_store.h`, anche
   per singolo gioco. La sessione imposta `EDEN_FAST_FIRST_PIPELINES` prima di avviare il
   renderer, e la riga "Performance:" del log dice se è attivo. Salvare
   Impostazioni > Prestazioni dal menu lo conserva. È coperto da `headless/preferences_check.cpp`.
3. **Misura nel log** (`eden_log.txt`):
   - "The GPU worker waited … ms for a pipeline being built", per ogni attesa oltre 100 ms;
   - "Pipeline build waits in the last … s: …", un riepilogo ogni 10 s o più.

   È esattamente il blocco che vedi, quindi si può confrontare una sessione con
   `fast_pipelines` attivo e una con `"fast_pipelines": false`.

Lo script di port (`tools/prepare-vulkan-port.py`) trova tutti i suoi punti di aggancio. Le
quattro patch del core si tolgono e si rimettono in modo pulito, e i controlli delle impostazioni
passano. Il comportamento sulla console **non è ancora verificato**.

## Come provarlo

1. Scarica lo ZIP dalla run "Release build" del ramo `perf/fast-first-pipelines` e installalo
   come al solito (`/data/homebrew/PPSA98515`).
2. Gioca a Pokopia nelle stesse zone in cui si bloccava. Gli shader già nella cache vengono
   caricati ottimizzati alla schermata di caricamento; i blocchi possibili riguardano solo shader
   mai visti.
3. Prendi `/data/eglantineps5/logs` e `eden_log.txt` via FTP e cerca le righe
   "Pipeline build waits".
4. Per un confronto A/B, metti `"fast_pipelines": false` dentro `"performance"` in
   `/data/eglantineps5/config/eglantineps5.json`, oppure nel `"performance"` del gioco
   `0400A3A01EE4C000` dentro `"games"`.

## Ricerca su GitHub e altrove

| Progetto | Cosa porta | Stato per questo fork |
|---|---|---|
| [niakw/ProsperoEden-Encore](https://github.com/niakw/ProsperoEden-Encore) | Fast-first pipelines; shader asincroni di default; fix all'acquisizione dello swapchain (`OUT_OF_DATE`/`SUBOPTIMAL`, timeout di 10 s invece di attesa infinita); vertex input statico per shader senza attributi (colore piatto nelle cutscene con RADV) | Fast-first **portata**. Gli shader asincroni no: per i giochi Switch 2 il core attuale non salta draw. Fix swapchain e vertex input sono candidati da valutare |
| [zortos293/Eglantine](https://github.com/zortos293/Eglantine) `nx2` | 355 commit dopo il pin: shader mesh/task (`VK_EXT_mesh_shader`), molti fix di traduzione `nx2`, sync delle query prima delle letture DMA delle macro, conditional rendering ignorato per i titoli Switch 2, budget memoria della GPU integrata, riuso delle immagini ASTC; anche "destroy graphics pipelines unused for 9000 frames" (pipeline ricompilate dopo un po': la fast-first ne riduce il costo) | Le patch 0001, 0002 (di EglantinePS5) e 0010 non si applicano più al core nuovo: l'aggiornamento va fatto insieme a heydemoura, poi la 0010 va riadattata |
| [heydemoura/EglantinePS5 PR #10](https://github.com/heydemoura/EglantinePS5/pull/10) (bozza) | Caricamento di Mario Kart World 117 s → 66 s (aggiornamenti locali della mappa GPU, metadati RomFS estratti), readback mappato | Non pronta per una release: un controllo fallisce e non è testata su console nella forma finale |
| [heydemoura/EglantinePS5 PR #8](https://github.com/heydemoura/EglantinePS5/pull/8) | Build Zen 2 / ThinLTO, routine di mappatura 36× più veloce in un test sintetico | Sperimentale: i test su console si fermano prima del gameplay |
| [sylvieistaken/selevet-eglantine](https://github.com/sylvieistaken/selevet-eglantine) | Build Eglantine con PGO, ThinLTO e mappa GPU più veloce (per PC Zen 3) | Idee per la build PS5 (CPU Zen 2), non ancora misurate sulla console |
| PokeDinner (Franseven, git.eden-emu.dev) | Fork di Eglantine: cielo di Pokopia blu, riflessi degli alberi, fix confluiti nel core Eglantine | Arrivano con l'aggiornamento del core |
| Eden-NS2 e porting Android | Stessa base del core | Niente di specifico per la PS5 |

## Prossimi passi

1. **Test su console** di questa build, con i log (righe "Pipeline build waits").
2. Se restano blocchi **senza** attese di pipeline nel log, la causa è altrove: memoria condivisa
   (`stderr.log`, righe `EDEN_HEAP_SITE` e `EDEN_IMAGE_REFUSED`), I/O o JIT. Si guarda lì, con
   i dati.
3. Quando EglantinePS5 aggiorna il core, rifare la patch 0010 sul core nuovo.
4. Valutare i fix swapchain di Encore come patch separata: correggono casi di errore e non
   cambiano il caso normale.
