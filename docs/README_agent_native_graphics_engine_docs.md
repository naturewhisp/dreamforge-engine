# Pacchetto documenti — Agent-Native & Dream-Driven Graphics Engine

Questo pacchetto contiene tre documenti Markdown pronti per lo sviluppo:

1. `01_specifica_tecnica_agent_native_graphics_engine.md`  
   Specifica tecnica completa e ristrutturata.

2. `02_roadmap_implementazione_agent_native_graphics_engine.md`  
   Roadmap implementativa per milestone, con deliverable, task, rischi ed exit criteria.

3. `03_adr_decisioni_architetturali.md`  
   Decisioni architetturali chiave in formato ADR.

Decisione principale: il progetto va gestito come un unico programma di sviluppo in monorepo modulare, con due sottoprogetti interni distinti:

- `engine-core`: motore grafico/fisico deterministico, testabile senza AI;
- `agent-runtime`: orchestrazione agentica, critic, actor, dataset, training e protocolli.

Il vincolo Windows + 12 GB VRAM + 64 GB RAM è assunto come requisito fondante della progettazione.
