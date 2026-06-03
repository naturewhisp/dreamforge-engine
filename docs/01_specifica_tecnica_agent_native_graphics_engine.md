# Specifica Tecnica di Sviluppo
# Agent-Native & Dream-Driven Graphics Engine

**Versione:** 1.0 sviluppo  
**Data:** 2026-06-03  
**Ambiente vincolante:** Windows, singola workstation, GPU con 12 GB VRAM, 64 GB RAM, SSD NVMe  
**Obiettivo:** costruire un motore grafico e operativo pensato nativamente per agenti AI locali, capace di generare, verificare e correggere scene 3D attraverso un ciclo deterministico `render -> observe -> correct`.

---

## 1. Sintesi esecutiva

Il progetto non deve essere trattato come un normale motore grafico con interfaccia umana sopra cui aggiungere agenti AI. Deve essere progettato come un **micromondo 3D verificabile**, dove ogni modifica prodotta da un agente è:

1. rappresentata come delta strutturato;
2. applicata a uno scene graph deterministico;
3. renderizzata in viste diagnostiche;
4. validata da un critic multimodale;
5. consolidata in telemetria riutilizzabile.

Il vincolo della macchina singola con 12 GB VRAM non è un dettaglio secondario: è il requisito architetturale primario. Tutte le decisioni tecniche devono evitare l'esaurimento della VRAM, il fallback di memoria condivisa Windows/WDDM, il parallelismo non controllato tra modelli e il trasferimento non necessario di texture o tensori.

La strategia corretta è progressiva:

- prima un prototipo web headless per validare il loop cognitivo;
- poi un core nativo con `wgpu`/`wgpu-native`;
- poi ottimizzazione locale dell'inferenza;
- solo dopo training LoRA/QLoRA e federazione A2A.

---

## 2. Decisione progettuale: progetto unico o due progetti?

### 2.1 Valutazione

Il sistema contiene due domini con cicli di sviluppo differenti:

| Dominio | Natura | Rischio | Frequenza di modifica |
|---|---|---:|---:|
| Engine deterministico 3D | infrastruttura grafica, fisica, scene graph, snapshot, IPC | alto | media |
| Agentic development layer | orchestrazione agenti, prompt, critic, dataset, LoRA, MCP/A2A | molto alto | alta |

Separarli completamente in due prodotti indipendenti creerebbe attrito: l'agente ha bisogno di API engine progettate su misura e l'engine deve emettere telemetria utile agli agenti.

Fonderli in un singolo blocco monolitico sarebbe però pericoloso: renderebbe difficile testare l'engine senza AI, sostituire modelli, isolare bug e fare benchmark.

### 2.2 Decisione raccomandata

Il progetto deve essere trattato come **un unico programma di sviluppo**, ma diviso in **due sottoprogetti/repository interni con contratti stabili**.

```text
agent-native-graphics/
  engine-core/              # runtime 3D deterministico, headless, testabile senza LLM
  agent-runtime/            # orchestrazione agenti, critic, actor, director, training data
  contracts/                # schema condivisi: scene graph, delta, snapshot metadata, telemetry
  tools/                    # benchmark, dataset inspection, regression runner
  docs/                     # specifiche, roadmap, ADR
```

### 2.3 Regola architetturale

L'engine non deve dipendere dagli agenti. Gli agenti dipendono dall'engine tramite contratti versionati.

Questo permette di:

- sviluppare il renderer e la fisica con test deterministici;
- cambiare modello AI senza riscrivere il motore;
- usare l'engine in modalità batch per generare dataset;
- disattivare completamente AI/training quando si misura performance grafica;
- mantenere una pipeline compatibile con il target 12 GB VRAM.

---

## 3. Principi fondativi

### 3.1 Agent UI Parity

Ogni funzionalità importante deve essere disponibile tramite API e contratti computazionali, non soltanto tramite GUI. L'interfaccia primaria del sistema è uno schema leggibile da agenti e validabile da codice.

### 3.2 Ground Truth deterministico

Il motore grafico e fisico è l'arbitro di verità. L'agente può proporre intenzioni, asset e trasformazioni, ma la validazione finale deriva da:

- collisioni;
- raycast;
- bounding volume;
- depth map;
- wireframe;
- vincoli fisici;
- test di regressione.

### 3.3 Dream-Driven Development, ma differito

Il concetto di sviluppo “dream-driven” va inteso come obiettivo di lungo periodo: gli agenti imparano dai tentativi falliti e dalle correzioni. Tuttavia il training notturno non deve essere implementato nella prima fase. Prima bisogna produrre dati affidabili.

### 3.4 VRAM-first design

Ogni componente deve dichiarare il proprio budget memoria. Se una funzione non può essere misurata in VRAM, latency e throughput, non è pronta per entrare nel path principale.

---

## 4. Architettura logica

```text
Utente / Regista
      |
      v
Director Agent
      |
      v
Task Planner
      |
      +---------------------------+
      |                           |
      v                           v
Actor / Architect Agent       Asset Pipeline
      |
      v
Scene Delta Contract
      |
      v
Engine Core
      |
      +--> Physics / Constraints
      +--> Scene Graph
      +--> Renderer
      +--> Diagnostic Snapshot
      |
      v
Visual Critic Agent
      |
      v
Correction Contract
      |
      +--> retry loop
      |
      v
Telemetry Store SQLite
```

---

## 5. Sottoprogetto A: `engine-core`

### 5.1 Responsabilità

`engine-core` è il runtime deterministico. Deve funzionare anche senza modelli AI.

Responsabilità:

- gestire scene graph;
- applicare delta atomici;
- eseguire vincoli geometrici;
- calcolare collisioni e raycast;
- renderizzare snapshot diagnostici;
- emettere telemetria strutturata;
- esporre API locali a bassa latenza;
- eseguire test di regressione grafici e fisici.

### 5.2 Linguaggio

Per la fase nativa si raccomanda **Rust** come prima scelta.

Motivazione:

- ecosistema maturo per `wgpu`;
- memory safety;
- ottima integrazione con serialization, testing e tooling;
- minore rischio rispetto a C++ per un progetto individuale;
- più stabilità ecosistemica rispetto a Zig per grafica/AI tooling oggi.

Zig rimane interessante per moduli specifici a basso livello, ma non come prima scelta per il core iniziale.

### 5.3 Rendering

Backend consigliato: `wgpu` / `wgpu-native`.

`wgpu` è una libreria grafica portabile basata su WebGPU e supporta esecuzione nativa su Vulkan, Metal, DirectX 12 e OpenGL ES, oltre a WebAssembly/browser. Per Windows il backend rilevante è DirectX 12 o Vulkan.

### 5.4 Scene graph minimo

L'MVP deve supportare solo primitive e asset essenziali:

```text
Scene
  Entity
    id: string
    transform: position, rotation, scale
    mesh_ref: primitive | asset_id
    material_ref: material_id
    physics: collider, mass, static/dynamic
    semantic_tags: array<string>
```

Primitive iniziali:

- cube;
- sphere;
- plane;
- capsule;
- cylinder;
- imported mesh placeholder.

### 5.5 Delta applicativi

Ogni modifica prodotta dall'Actor deve essere un delta atomico:

```json
{
  "op_id": "uuid",
  "scene_id": "scene_001",
  "operations": [
    {
      "type": "translate",
      "entity_id": "table_01",
      "axis": "y",
      "value": -0.42
    }
  ],
  "expected_state": {
    "intent": "table rests on floor",
    "constraints": ["no_floating", "no_clipping"]
  }
}
```

Il delta deve essere validato con schema prima di essere applicato.

### 5.6 Snapshot diagnostico

L'engine deve produrre un'immagine composita con tre pannelli:

1. RGB render;
2. depth map;
3. wireframe.

Raccomandazione MVP:

- risoluzione composita: 768x256 oppure 896x299;
- ogni pannello uguale dimensione;
- formato PNG per semplicità nel prototipo;
- metadata JSON separato con camera, entity IDs, bounding boxes, depth range.

La vecchia idea del frame unico 448x448 va corretta: 448x448 può essere utile come input modello, ma un triplo pannello orizzontale rischia di ridurre troppo la leggibilità. Meglio generare snapshot ad alta leggibilità e poi derivare una versione downscaled per il VLM.

### 5.7 Zero-copy GPU: posizione realistica

Non deve essere requisito MVP.

Livelli implementativi:

| Livello | Tecnica | Stato |
|---|---|---|
| L0 | PNG su disco / memoria | MVP |
| L1 | buffer CPU condiviso | fase 2 |
| L2 | GPU staging buffer + tensor conversion controllata | fase 3 |
| L3 | CUDA/D3D12/Vulkan external memory interop | ricerca avanzata |

Il documento originale parlava di passaggio diretto di puntatori VRAM al runtime AI. Questa è una direzione di ricerca, non un prerequisito di sviluppo. Richiede sincronizzazione esplicita, compatibilità runtime AI, gestione fence/semaphore e benchmark.

---

## 6. Sottoprogetto B: `agent-runtime`

### 6.1 Responsabilità

`agent-runtime` coordina gli agenti, i prompt, i modelli, il ciclo di correzione e il dataset.

Responsabilità:

- interpretare richieste utente;
- scomporre task;
- generare delta scene graph;
- invocare engine-core;
- leggere snapshot diagnostici;
- validare con critic multimodale;
- produrre correzioni;
- salvare telemetria;
- lanciare benchmark di regressione;
- in futuro addestrare adapter LoRA.

### 6.2 Ruoli agentici

#### Director

Coordina il task. Non produce geometria direttamente.

Output:

- obiettivo;
- vincoli;
- lista task;
- acceptance criteria.

#### Architect / Actor

Produce delta strutturati.

Output:

- `SceneDelta` valido;
- motivazione sintetica;
- vincoli attesi.

#### Visual Critic

Analizza snapshot e metadata. Non deve produrre chain-of-thought. Deve produrre output strutturato.

Output:

```json
{
  "status": "APPROVED",
  "checks": {
    "geometry": "pass",
    "depth_consistency": "pass",
    "lighting": "pass",
    "physics_plausibility": "pass"
  },
  "critique_nodes": []
}
```

In caso di errore:

```json
{
  "status": "REJECTED",
  "checks": {
    "geometry": "fail",
    "depth_consistency": "pass",
    "lighting": "pass",
    "physics_plausibility": "fail"
  },
  "critique_nodes": [
    {
      "id": "chair_01",
      "error_type": "floating",
      "severity": "high",
      "corrective_action": {
        "operation": "translate",
        "axis": "y",
        "delta_estimate": -0.35
      }
    }
  ]
}
```

### 6.3 Modello AI locale

Non fissare il progetto a un singolo modello nominale.

La strategia corretta è definire profili intercambiabili:

| Profilo | Uso | Requisito |
|---|---|---|
| Local-S | planner, actor testuale | bassa VRAM |
| Local-M | critic multimodale leggero | VRAM controllata |
| Hybrid | actor locale + critic remoto opzionale | qualità superiore |
| Research | training LoRA/QLoRA | solo dopo dataset stabile |

Nota importante: le taglie Gemma devono essere verificate contro le release effettive disponibili. Al 2026, le release ufficiali Gemma 4 indicano E2B, E4B, 31B e 26B A4B, non 7B/9B. Il progetto deve quindi usare una capability matrix, non nomi rigidi.

### 6.4 Server di inferenza

L'idea di un singolo server locale con ruoli intercambiabili resta valida, ma va implementata in modo conservativo.

Requisiti:

- un solo modello principale caricato alla volta;
- contesto limitato;
- KV cache dimensionata;
- batch controllato;
- nessun caricamento simultaneo actor+critic se supera budget;
- fallback esplicito a modello testuale se il critic multimodale non entra in VRAM.

---

## 7. Protocolli e comunicazione

### 7.1 Regola control plane / data plane

Separazione obbligatoria:

| Piano | Uso | Tecnologie |
|---|---|---|
| Data plane | scene delta, snapshot, telemetry ad alta frequenza | IPC locale, file, shared memory, FlatBuffers/Cap'n Proto |
| Control plane | tool discovery, risorse, integrazioni agente | MCP |
| Federation plane | agenti esterni, servizi remoti | A2A |

### 7.2 IPC locale

MVP:

- HTTP locale o stdio JSON-RPC per semplicità;
- file snapshot su disco;
- SQLite per telemetria.

Fase nativa:

- Named Pipes Windows o Unix sockets equivalenti;
- FlatBuffers o Cap'n Proto;
- opzionale gRPC per tooling non real-time.

### 7.3 MCP

MCP va usato per esporre:

- asset disponibili;
- scene disponibili;
- funzioni engine invocabili;
- log strutturati;
- documentazione contratti;
- dataset/telemetria come risorse.

Non va usato per trasportare frame diagnostici ad alta frequenza nel loop real-time.

### 7.4 A2A

A2A va introdotto solo dopo che il sistema locale è stabile.

Uso previsto:

- richiedere asset a generatori esterni;
- delegare musica, mesh refinement, texture synthesis;
- ricevere pacchetti asset validati;
- pubblicare una Agent Card del Director.

A2A non deve essere dipendenza necessaria per l'MVP.

---

## 8. Telemetria e dataset

### 8.1 Database SQLite

Tabelle minime:

```sql
scenes(
  scene_id TEXT PRIMARY KEY,
  created_at TEXT,
  description TEXT
);

attempts(
  attempt_id TEXT PRIMARY KEY,
  scene_id TEXT,
  task_id TEXT,
  attempt_index INTEGER,
  input_prompt TEXT,
  scene_delta_json TEXT,
  snapshot_path TEXT,
  metadata_json TEXT,
  critic_json TEXT,
  status TEXT,
  created_at TEXT
);

approved_transitions(
  transition_id TEXT PRIMARY KEY,
  scene_id TEXT,
  task_id TEXT,
  initial_state_hash TEXT,
  final_state_hash TEXT,
  approved_delta_json TEXT,
  created_at TEXT
);
```

### 8.2 Criteri di qualità del dataset

Un esempio entra nel dataset di training solo se:

- lo schema è valido;
- lo snapshot esiste;
- il critic produce JSON valido;
- esiste almeno un confronto prima/dopo;
- l'approval è riproducibile da test;
- non ci sono asset mancanti.

---

## 9. Training LoRA/QLoRA

### 9.1 Stato nel progetto

Training notturno: **non MVP**.

Deve partire solo dopo:

- almeno 1.000 transizioni approvate;
- benchmark di regressione stabile;
- metriche automatiche per clipping/floating/depth;
- dataset bilanciato tra successi e fallimenti;
- script di rollback adapter.

### 9.2 Obiettivo realistico

Il training non deve insegnare al modello “la grafica 3D”. Deve insegnare mapping specifici:

- come interpretare i pannelli diagnostici dell'engine;
- come riconoscere errori frequenti nello scene graph;
- come emettere correzioni compatibili con i contratti;
- come ridurre retry inutili.

### 9.3 Vincoli 12 GB VRAM

Configurazione iniziale consigliata:

- LoRA/QLoRA su modello piccolo;
- batch size 1;
- gradient accumulation;
- gradient checkpointing;
- vision encoder frozen;
- sequence length limitata;
- niente training mentre engine o inferenza restano in VRAM.

---

## 10. Sicurezza e robustezza

### 10.1 Sandbox scripting

Gameplay scripting solo tramite:

- Lua sandboxed;
- WASM runtime;
- nessun accesso diretto al filesystem;
- limiti CPU/time budget;
- API esplicite verso engine.

### 10.2 Validazione input agente

Ogni output agente deve passare per:

- JSON/schema validation;
- entity existence check;
- unit range check;
- transform sanity check;
- collision precheck;
- rollback se fallisce.

### 10.3 Rollback

Ogni delta deve essere reversibile o applicato su copia dello stato.

---

## 11. Metriche di successo

### 11.1 Metriche MVP

- percentuale di delta validi al primo tentativo;
- tempo medio `delta -> render -> critic`;
- numero medio retry per task;
- accuratezza critic su casi sintetici;
- VRAM peak durante loop;
- assenza fallback memoria condivisa;
- riproducibilità snapshot.

### 11.2 Metriche fase nativa

- frame diagnostici al secondo;
- latency IPC;
- tempo readback snapshot;
- dimensione scene gestibile;
- stabilità engine su batch di 1.000 task;
- crash rate.

---

## 12. Decisioni architetturali chiave

| Decisione | Scelta |
|---|---|
| Progetto | unico programma, due sottoprogetti |
| Engine MVP | web headless |
| Engine produzione | Rust + wgpu/wgpu-native |
| Agent layer | separato da engine |
| Comunicazione MVP | JSON/file/SQLite |
| Comunicazione avanzata | FlatBuffers/Cap'n Proto + IPC locale |
| MCP | control plane/tool discovery |
| A2A | federazione esterna post-MVP |
| Zero-copy GPU | ricerca avanzata, non MVP |
| Training LoRA | fase 4, non iniziale |
| Prompt critic | JSON strutturato, no chain-of-thought |

---

## 13. Riferimenti tecnici

- wgpu: https://wgpu.rs/
- wgpu-native: https://github.com/gfx-rs/wgpu-native
- Model Context Protocol: https://modelcontextprotocol.io/specification/
- MCP Resources: https://modelcontextprotocol.io/specification/2025-06-18/server/resources
- A2A Protocol: https://a2a-protocol.org/latest/
- A2A Specification: https://a2a-protocol.org/latest/specification/
- Gemma releases: https://ai.google.dev/gemma/docs/releases
- Gemma 4 model card: https://ai.google.dev/gemma/docs/core/model_card_4
- NVIDIA CUDA graphics interop: https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/graphics-interop.html
