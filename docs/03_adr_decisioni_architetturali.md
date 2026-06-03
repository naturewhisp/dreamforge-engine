# ADR — Decisioni Architetturali
# Agent-Native & Dream-Driven Graphics Engine

**Versione:** 1.0  
**Data:** 2026-06-03

---

## ADR-001 — Monorepo modulare invece di progetto monolitico o repository separati

### Stato

Accettata.

### Contesto

Il progetto combina un motore grafico deterministico e un runtime agentico sperimentale. I due domini hanno cicli di sviluppo diversi ma dipendono da contratti comuni.

### Decisione

Usare un monorepo modulare:

```text
engine-core/
agent-runtime/
contracts/
tools/
docs/
datasets/
benchmarks/
```

### Conseguenze

Positive:

- refactoring rapido;
- test end-to-end semplici;
- sviluppo individuale più fluido;
- contratti sempre sincronizzati.

Negative:

- rischio accoppiamento eccessivo;
- disciplina necessaria per mantenere l'engine indipendente dagli agenti.

### Regola

`engine-core` non deve importare codice da `agent-runtime`.

---

## ADR-002 — MVP web headless prima del motore nativo

### Stato

Accettata.

### Contesto

Scrivere subito un engine nativo rischia di spostare lo sforzo su rendering, GPU e tooling prima di validare il loop agentico.

### Decisione

Implementare prima un prototipo con Node.js, Three.js/Babylon.js e Playwright/Chromium headless.

### Conseguenze

Positive:

- feedback rapido;
- debugging visivo semplice;
- validazione dei contratti;
- dataset iniziale generabile subito.

Negative:

- overhead browser;
- performance non rappresentative;
- gestione memoria meno controllata.

### Exit rule

Passare al nativo solo dopo che il loop `render -> observe -> correct` funziona su errori sintetici.

---

## ADR-003 — Rust + wgpu per `engine-core`

### Stato

Accettata provvisoriamente.

### Contesto

Il motore nativo richiede portabilità GPU, controllo memoria, testabilità e buon ecosistema.

### Decisione

Usare Rust e `wgpu`/`wgpu-native`.

### Alternative considerate

- C++: massima flessibilità, maggiore rischio memory-safety.
- Zig: interessante, ecosistema grafico meno maturo.
- C#: produttivo, ma meno ideale per core GPU/engine a basso livello in questo scenario.

### Conseguenze

Positive:

- memory safety;
- ecosistema `wgpu`;
- test e tooling robusti;
- buona integrazione con serialization.

Negative:

- curva di apprendimento;
- interop AI/GPU avanzata potenzialmente più complessa.

---

## ADR-004 — MCP fuori dal path real-time

### Stato

Accettata.

### Contesto

MCP è utile per tool discovery, risorse e integrazioni LLM, ma non per trasporto frame o loop ad alta frequenza.

### Decisione

Usare MCP come control plane:

- asset discovery;
- scene discovery;
- dataset resources;
- log;
- benchmark;
- documentazione contratti.

### Conseguenze

Il loop principale userà IPC locale/file/shared memory, non MCP.

---

## ADR-005 — A2A solo post-MVP

### Stato

Accettata.

### Contesto

A2A abilita interoperabilità tra agenti, ma introduce rete, sicurezza, policy, autenticazione e asset ingestion.

### Decisione

Rimandare A2A a una fase successiva. L'MVP deve funzionare completamente offline/localmente.

### Conseguenze

Il sistema resta sviluppabile e testabile su macchina singola.

---

## ADR-006 — Zero-copy GPU non requisito MVP

### Stato

Accettata.

### Contesto

Il passaggio diretto di texture GPU al runtime AI è tecnicamente complesso e dipende da runtime, API grafiche, sincronizzazione e supporto interop.

### Decisione

Usare livelli progressivi:

1. PNG/file o buffer CPU;
2. memoria condivisa CPU;
3. staging GPU controllato;
4. external memory interop solo come ricerca avanzata.

### Conseguenze

L'MVP sarà meno efficiente ma molto più realizzabile.

---

## ADR-007 — Niente chain-of-thought nel prompt del critic

### Stato

Accettata.

### Contesto

Il critic deve produrre output utilizzabile da codice. Richiedere ragionamento verboso aumenta instabilità, parsing fragile e dipendenza da testo non operativo.

### Decisione

Il critic produce solo JSON strutturato con check, errori e azioni correttive.

### Conseguenze

- parsing più robusto;
- minor token usage;
- migliore automazione;
- audit più semplice.

---

## ADR-008 — Training LoRA/QLoRA dopo dataset e regressione

### Stato

Accettata.

### Contesto

Senza dataset validato, il training rischia di consolidare rumore.

### Decisione

Training solo dopo:

- 1.000+ transizioni approvate;
- benchmark regressione;
- metriche critic;
- rollback adapter.

### Conseguenze

Il progetto privilegia dati e misurabilità rispetto a training prematuro.
