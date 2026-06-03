# AGENTS.md

## Missione del repository

Questo repository implementa un Agent-Native & Dream-Driven Graphics Engine pensato per sviluppo locale su singola workstation Windows con 12 GB VRAM, 64 GB RAM e SSD NVMe.

Il vincolo hardware non è temporaneo: è parte del problema progettuale.

## Regole assolute

- Non assumere GPU cloud, cluster, memoria VRAM superiore a 12 GB o servizi remoti obbligatori.
- Non introdurre dipendenze runtime pesanti senza motivazione tecnica documentata.
- Non usare MCP, A2A, HTTP o JSON polling nel path real-time del motore.
- Non implementare training QLoRA, zero-copy GPU interop o federazione A2A prima che le milestone precedenti siano completate.
- Ogni modifica deve preservare la separazione tra `engine-core` e `agent-runtime`.

## Architettura del repository

- `engine-core/`: motore grafico/fisico deterministico, scene graph, snapshot diagnostici, scripting sandbox.
- `agent-runtime/`: orchestrazione agentica, actor, critic, telemetry, dataset e valutazione.
- `docs/`: specifiche, roadmap, ADR e note di progettazione.
- `tools/`: script di benchmark, validazione, generazione dataset e manutenzione.
- `examples/`: scene minime e casi di test riproducibili.

## Priorità di sviluppo

1. Costruire un micromondo 3D verificabile.
2. Implementare il loop Render-Observe-Correct.
3. Salvare telemetria utile in SQLite.
4. Misurare memoria, latenza e stabilità.
5. Solo dopo valutare training, zero-copy e federazione esterna.

## Regole per engine-core

- Il motore deve essere utilizzabile senza agenti AI.
- Le API devono essere deterministiche e testabili.
- Ogni snapshot diagnostico deve poter essere riprodotto da seed, scene state e camera state.
- Evitare allocazioni non necessarie nel loop di rendering.
- Ogni feature grafica deve avere almeno un test o esempio minimo.

## Regole per agent-runtime

- Gli agenti devono produrre output strutturato, non testo libero.
- Il critic non deve restituire chain-of-thought; deve restituire solo verifiche, errori e correzioni azionabili.
- Ogni correzione deve riferirsi a entity ID stabili.
- La telemetria deve distinguere tentativi falliti, critiche e payload approvato.
- Non addestrare adapter se il dataset non è validato e versionato.

## Testing e benchmark

Prima di considerare completata una modifica, verificare almeno:

- test unitari del modulo modificato;
- scena esempio riproducibile, se la modifica tocca il rendering;
- misurazione VRAM/RAM se la modifica tocca rendering, inferenza o snapshot;
- nessuna regressione sui casi minimi già presenti.

## Policy memoria

Target operativo:

- nessun fallback WDDM intenzionale su memoria condivisa;
- logging di VRAM usata nei benchmark;
- snapshot iniziali piccoli e fissi;
- batch size minimo per ogni pipeline AI locale;
- preferire streaming e delta rispetto a payload completi.

## Divieti temporanei

Fino al completamento dell'MVP:

- niente training notturno QLoRA;
- niente CUDA/Vulkan/D3D zero-copy obbligatorio;
- niente A2A produttivo;
- niente plugin marketplace;
- niente editor visuale completo;
- niente generazione asset fotorealistici come requisito core.

## Stile di contribuzione

- Fare modifiche piccole e verificabili.
- Aggiornare gli ADR quando si prende una decisione architetturale.
- Non duplicare logica tra engine e agent runtime.
- Preferire codice semplice, misurabile e debuggabile.
- Documentare trade-off e limiti, non solo soluzioni.