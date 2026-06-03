# Roadmap di Implementazione
# Agent-Native & Dream-Driven Graphics Engine

**Versione:** 1.0 sviluppo  
**Data:** 2026-06-03  
**Ambiente vincolante:** Windows, 12 GB VRAM, 64 GB RAM, SSD NVMe  

---

## 1. Obiettivo della roadmap

Questa roadmap trasforma il concept in un percorso implementativo misurabile. Il principio guida è evitare di costruire subito l'intero sistema finale. Prima bisogna dimostrare che un agente può modificare una scena, ricevere feedback visivo strutturato e correggere l'errore in modo riproducibile.

---

## 2. Struttura progetto raccomandata

```text
agent-native-graphics/
  engine-core/
  agent-runtime/
  contracts/
  tools/
  docs/
  datasets/
  benchmarks/
```

### Repository unico o multiplo?

Raccomandazione iniziale: **monorepo modulare**.

Motivo:

- facilita refactoring rapido dei contratti;
- mantiene sincronizzati engine e agent layer;
- semplifica sviluppo individuale;
- permette test end-to-end locali;
- evita overhead organizzativo prematuro.

Quando separare in repository distinti:

- quando `engine-core` ha API stabili;
- quando esiste una suite di test completa;
- quando agent-runtime può essere sostituito senza modifiche all'engine;
- quando nasce una necessità di distribuzione separata.

---

## 3. Fase 0 — Preparazione e baseline

**Durata stimata:** 1-2 settimane  
**Obiettivo:** definire ambiente, metriche e contratti minimi.

### Deliverable

- repository iniziale;
- schema `Scene`, `Entity`, `SceneDelta`, `CriticResult`;
- script per rilevare VRAM e processi GPU;
- ADR iniziali;
- dataset sintetico di 10 scene base;
- benchmark manuale documentato.

### Task

1. Creare monorepo.
2. Definire cartella `contracts/`.
3. Scrivere JSON Schema per scene graph e delta.
4. Scrivere casi minimi:
   - cubo su piano;
   - sfera sospesa;
   - due cubi compenetrati;
   - luce errata;
   - scala incoerente.
5. Definire metriche MVP:
   - VRAM peak;
   - tempo render;
   - tempo critic;
   - validità JSON;
   - retry count.

### Exit criteria

- gli schema validano esempi positivi e rifiutano esempi errati;
- esiste una baseline VRAM a sistema vuoto;
- esiste un formato univoco per snapshot metadata.

---

## 4. Fase 1 — MVP web headless

**Durata stimata:** 3-5 settimane  
**Obiettivo:** dimostrare il loop `render -> observe -> correct` senza engine nativo.

### Stack

- Node.js;
- Three.js o Babylon.js;
- Playwright/Chromium headless;
- SQLite;
- JSON files.

### Deliverable

- renderer headless;
- scene graph minimale;
- applicatore delta;
- snapshot RGB/depth/wireframe;
- critic adapter;
- retry loop;
- salvataggio SQLite;
- report benchmark.

### Task tecnici

1. Implementare `scene-loader`.
2. Implementare `apply-delta`.
3. Implementare render RGB.
4. Implementare depth pass.
5. Implementare wireframe pass.
6. Comporre snapshot diagnostico.
7. Salvare metadata:
   - camera;
   - bounding boxes;
   - entity IDs;
   - depth range;
   - hash scena.
8. Scrivere `critic-runner`.
9. Scrivere `correction-loop` con massimo 3 retry.
10. Salvare ogni tentativo in SQLite.

### Exit criteria

- il sistema corregge almeno 3 classi di errori sintetici:
  - oggetto sospeso;
  - oggetto compenetrato;
  - scala manifestamente errata;
- almeno 50 task batch completati senza crash;
- snapshot e metadata sono riproducibili;
- nessun componente supera budget VRAM stabilito.

### Rischi

| Rischio | Mitigazione |
|---|---|
| Chromium consuma troppa memoria | scene minime, batch seriale, chiusura processi |
| depth/wireframe non coerenti | test visivi manuali iniziali |
| critic multimodale instabile | casi sintetici semplici, metadata espliciti |

---

## 5. Fase 2 — Contratti stabili e test suite

**Durata stimata:** 2-3 settimane  
**Obiettivo:** stabilizzare l'interfaccia tra agenti ed engine prima della riscrittura nativa.

### Deliverable

- versione 1 degli schema;
- test suite contract-based;
- fixture scene;
- validator CLI;
- dataset viewer minimale.

### Task

1. Bloccare `SceneGraph v0.1`.
2. Bloccare `SceneDelta v0.1`.
3. Bloccare `CriticResult v0.1`.
4. Creare `contracts validate` CLI.
5. Creare test fixture.
6. Aggiungere hash deterministico stato scena.
7. Scrivere guida per nuove skill geometriche.

### Exit criteria

- ogni delta è validabile senza caricare il renderer;
- ogni critic result è machine-readable;
- ogni snapshot ha metadata coerente;
- il loop può essere testato in modalità offline.

---

## 6. Fase 3 — Engine nativo `engine-core`

**Durata stimata:** 6-10 settimane  
**Obiettivo:** sostituire il renderer web con core nativo performante e controllabile.

### Stack consigliato

- Rust;
- wgpu/wgpu-native;
- SQLite;
- serde/JSON inizialmente;
- FlatBuffers o Cap'n Proto in seconda iterazione.

### Deliverable

- executable `engine-core`;
- scene loader;
- renderer RGB/depth/wireframe;
- raycast/constraint module;
- snapshot exporter;
- benchmark VRAM/latency;
- compatibility layer con `agent-runtime`.

### Task tecnici

1. Creare scaffold Rust.
2. Integrare `wgpu`.
3. Implementare rendering primitive.
4. Implementare camera deterministica.
5. Implementare depth pass.
6. Implementare wireframe pass.
7. Implementare export snapshot.
8. Implementare collisioni/raycast minimi.
9. Implementare API locale.
10. Confrontare output con MVP web.

### Exit criteria

- riproduce almeno 80% dei casi MVP;
- batch 1.000 scene senza crash;
- picco VRAM misurato e documentato;
- latenza inferiore al prototipo web;
- nessuna dipendenza da browser.

---

## 7. Fase 4 — Agent runtime locale ottimizzato

**Durata stimata:** 4-8 settimane  
**Obiettivo:** eseguire agenti locali entro i 12 GB VRAM.

### Deliverable

- inference profile manager;
- prompt registry;
- role switching;
- critic JSON-only;
- context LOD;
- monitor VRAM;
- fallback policy.

### Task

1. Definire profili modello:
   - Local-S;
   - Local-M;
   - Hybrid.
2. Implementare prompt registry.
3. Implementare parsing robusto JSON output.
4. Implementare context summarization.
5. Implementare modello singolo caricato alla volta.
6. Misurare KV cache per contesti diversi.
7. Implementare retry con backoff e rollback.
8. Integrare monitor VRAM.

### Exit criteria

- actor e critic possono operare serialmente su stessa workstation;
- il sistema non supera budget VRAM;
- output non valido sotto soglia concordata;
- fallback testuale disponibile se critic multimodale fallisce.

---

## 8. Fase 5 — Skill geometriche e Context LOD

**Durata stimata:** 3-6 settimane  
**Obiettivo:** togliere calcoli geometrici fragili dal modello e portarli nel motore.

### Skill prioritarie

1. `place_on_surface(entity, surface)`
2. `align_to_grid(entity, grid_size)`
3. `resolve_intersection(entity_a, entity_b)`
4. `fit_inside(entity, volume)`
5. `look_at(entity, target)`
6. `summarize_scene_focus(focus_entity)`

### Exit criteria

- riduzione retry medi;
- meno errori di floating/clipping;
- scene più grandi senza saturare context window;
- output actor più semplice e stabile.

---

## 9. Fase 6 — MCP locale

**Durata stimata:** 2-4 settimane  
**Obiettivo:** esporre engine e dataset come tool/resources standard senza inserirli nel path real-time.

### Esposizione MCP

- lista scene;
- lista asset;
- lettura metadata;
- invocazione validator;
- consultazione telemetry;
- documentazione contratti;
- esecuzione benchmark.

### Exit criteria

- un client MCP può scoprire tool e risorse;
- nessun frame diagnostico passa obbligatoriamente da MCP;
- MCP è disattivabile senza rompere il loop principale.

---

## 10. Fase 7 — Dataset e regressione

**Durata stimata:** 4-8 settimane  
**Obiettivo:** preparare i dati prima di qualsiasi training.

### Deliverable

- dataset validator;
- train/validation/test split;
- regression scenes;
- metriche critic;
- confusion matrix errori;
- rollback adapter design.

### Task

1. Raccogliere 1.000+ transizioni approvate.
2. Classificare errori:
   - clipping;
   - floating;
   - scale;
   - occlusion;
   - lighting;
   - wrong asset.
3. Creare benchmark statico.
4. Creare golden outputs.
5. Misurare critic baseline.
6. Definire soglia minima per accettare training.

### Exit criteria

- dataset coerente;
- benchmark riproducibile;
- metriche automatiche disponibili;
- training giustificato da dati, non da intuizione.

---

## 11. Fase 8 — Training LoRA/QLoRA sperimentale

**Durata stimata:** 4-10 settimane  
**Obiettivo:** migliorare critic/actor su errori specifici dell'engine.

### Prerequisiti

- dataset validato;
- modello piccolo compatibile con VRAM;
- script di training isolato;
- engine spento durante training;
- rollback automatico.

### Configurazione iniziale

- batch size 1;
- gradient accumulation;
- gradient checkpointing;
- LoRA su layer testuali/connettore;
- vision encoder frozen se multimodale;
- sequence length limitata.

### Exit criteria

- miglioramento misurabile su validation set;
- nessun peggioramento oltre soglia su regression set;
- adapter disattivabile;
- training riproducibile.

---

## 12. Fase 9 — A2A e asset federation

**Durata stimata:** 3-6 settimane  
**Obiettivo:** consentire al Director di delegare asset o sottotask ad agenti esterni.

### Use case iniziali

- generazione texture;
- creazione musica;
- refinement mesh;
- concept art;
- validazione esterna.

### Exit criteria

- Agent Card pubblicabile;
- autenticazione/policy minima;
- asset ingestion controllata;
- scansione file ricevuti;
- A2A disattivabile senza compromettere MVP locale.

---

## 13. Piano di milestone sintetico

| Milestone | Risultato | Go/No-Go |
|---|---|---|
| M0 | contratti + baseline | schema e metriche esistono |
| M1 | MVP web loop | correzione errori sintetici |
| M2 | contratti stabili | test offline completi |
| M3 | engine nativo | batch 1.000 scene stabile |
| M4 | runtime locale | sotto budget VRAM |
| M5 | skill geometriche | retry medi ridotti |
| M6 | MCP | discovery/tooling funzionante |
| M7 | dataset | 1.000 transizioni validate |
| M8 | LoRA | miglioramento misurabile |
| M9 | A2A | federazione opzionale |

---

## 14. Priorità assolute

1. Non iniziare dal training.
2. Non iniziare dallo zero-copy GPU.
3. Non iniziare da A2A.
4. Non costruire subito un editor completo.
5. Costruire prima il ciclo verificabile più piccolo possibile.

---

## 15. Definition of Done dell'MVP

L'MVP è completo quando:

- una scena viene caricata;
- l'Actor genera un delta valido;
- l'engine applica il delta;
- lo snapshot diagnostico viene prodotto;
- il Critic approva o rifiuta con JSON valido;
- in caso di rifiuto l'Actor corregge;
- la transizione finale viene salvata in SQLite;
- tutto gira su Windows nella workstation target senza superare il budget memoria.
