# TRI / HIXX – Vollständiger Datenbaum

**Erstellt:** 15. September 2026  
**Geltungsbereich:**

- `/home/carlos/PROJEKTE/triAI-Engine`
- `/home/carlos/PROJEKTE/hixx-native`

Dieser Baum unterscheidet absichtlich zwischen **Quellwahrheit**,
**persistenten Nutzdaten**, **generierten Artefakten** und **flüchtigem
Laufzeitzustand**. `target/`, Kbuild-Objekte und vollständige Git-Worktrees
werden nicht einzeln aufgelistet, da sie reproduzierbar bzw. Kopien sind.

## 1. Physische Daten- und Eigentumsgrenzen

```text
/home/carlos/PROJEKTE/
├── triAI-Engine/                    Produktiver Engine-/Orchestrator-Code
│   ├── models/                      39 GiB – lokale Modell- und Chunk-Daten
│   ├── data/                        44 KiB – kuratierte Routing-/Lerndaten
│   ├── src/                         864 KiB – Rust-Quellcode
│   ├── kernel/                      1,1 MiB – separater triAI-Kernelversuch
│   └── target/                      generierte Rust-Build-Artefakte
│
└── hixx-native/                     eigenständiger HIXX-IPC-Prototyp
    ├── src/                         16 KiB – Rust-HTTP/IPC-Code
    ├── kernel/                      880 KiB – HIXX-Kernelmodul + Buildreste
    └── target/                      generierte Rust-Build-Artefakte
```

**Wichtig:** Beide Projekte enthalten aktuell ein Modul namens
`tri_ai_worker` und beanspruchen `/dev/tri_ai_worker`; siehe Abschnitt 8.

## 2. triAI-Engine – Top-Level-Baum

```text
triAI-Engine/
├── Cargo.toml                       Rust-Paket und Abhängigkeiten
├── Cargo.lock                       gelockte Abhängigkeitsversionen
├── PROJECT.md                       Projektziele und Status
├── README.md                        Einstieg
├── CLAUDE.md                        projektbezogene Agent-/Arbeitsnotizen
├── project-manifest.toml            Projektmetadaten
├── triAI-Engine.code-workspace      VS-Code-Arbeitsbereich
├── Dockerfile                       Container-Build
├── hardening.sh                     Härtungsskript (lokale Änderung)
├── stress_test.sh                   Stresstest-Einstieg
├── OPTIMIERUNGSPLAN_V2.md           Optimierungsplanung
│
├── src/                             produktiver Rust-Quellbaum
├── tests/                           Integration-/Cache-/Warmup-Tests
├── benches/                         Criterion-/Benchmark-Einstiege
├── config/                          Beispielkonfiguration
├── deploy/                          systemd-Service-Definition
├── docs/                            Architektur, Betrieb, Evidenz, Releases
├── scripts/                         Qualitätsgates, Benchmarks, Abläufe
├── skills/                          TRI-spezifische Skill-Anleitung
├── data/                            kleine persistente Wissensdaten
├── models/                          große lokale Modell- und Pack-Daten
├── kernel/                          triAI-Kernelmodul-Experiment
├── .github/workflows/ci.yml         CI-Workflow
├── .claude/skills/                  lokale Skill-Kopien
├── .vscode/                         Editoraufgaben/-empfehlungen
├── .worktrees/                      separater Git-Worktree, keine Primärquelle
└── target/                          Build-Cache und Binaries, regenerierbar
```

## 3. triAI-Engine – Source-Tree, vollständig nach Funktion

```text
src/
├── main.rs                          Prozessstart, Config, Eventlog, HTTP-Bind
├── lib.rs                           öffentliche Modulgrenze
├── engine.rs                        Engine-Zustandsautomat und Model-Slot
├── error.rs                         gemeinsame Fehlertypen
├── config.rs                        TOML/ENV-Konfiguration und Validierung
├── http.rs                          HTTP/1.1, Auth, Routing, Worker-Proxy
├── api.rs                           API-Vertrag und Antworttypen
├── openai.rs                        OpenAI-Request/Response-Normalisierung
├── observability.rs                 JSONL-Ereignislog und Redaction
├── resources.rs                     Host-/RAM-/SSD-Ressourcensnapshot
├── planner.rs                       Modell-/Offload-/KV-Planung
├── performance.rs                   Performance-Regeln und Guardrails
├── benchmark.rs                     Benchmark-Logik
├── prompt.rs                        Prompt-Einstieg
│   ├── caps.rs                      harte Token-/Aufruf-Limits
│   ├── compress.rs                  deterministische Prompt-Kompression
│   ├── estimate.rs                  konservative Token-Schätzung
│   └── hooks.rs                     Secret-/Cloud-Schutz und Hooks
├── supervisor.rs                    Workerprozess-Lebenszyklus
│   ├── expert_tracker.rs            Expert-/Worker-Eventtracking
│   ├── fallback.rs                  kontrollierter Fallback
│   ├── manifest.rs                  Manifest- und Pfadvalidierung
│   ├── primary.rs                   Primary-Worker-Beratung
│   └── slot.rs                      exklusiver Modellslot
├── chunk.rs                         Chunk-Einstieg
│   ├── classify.rs                  Tensor-/Chunk-Klassifikation
│   ├── gguf.rs                      GGUF-Lesen und Metadaten
│   ├── index.rs                     Chunkindex
│   ├── loader.rs                    Laden/Cache
│   ├── packer.rs                    zstd-Chunk-Packen
│   ├── rollback.rs                  read-only Fallback auf Original-GGUF
│   └── warmup.rs                    Warmup/Eager-Tensors
├── analysis.rs                      Analyse-Einstieg
│   ├── gates.rs                     Promotion-/Qualitätsgates
│   ├── job.rs                       Analysejob
│   ├── policy.rs                    versionierte Policy-Vorschläge
│   └── store.rs                     Analysepersistenz
├── evidence.rs                      Evidence-Einstieg
│   ├── jsonl.rs                     JSONL-Speicher
│   ├── rotation.rs                  Rotation
│   └── types.rs                     Evidenztypen
├── monitor.rs                       Trigger-Einstieg
│   ├── hysteresis.rs                Hysterese/Cooldown
│   ├── snapshot.rs                  Monitoring-Snapshot
│   └── trigger.rs                   Trigger-Regeln
├── assistant.rs                     Assistentenlogik
├── attachments.rs                   Anhangsverarbeitung
├── coding_tools.rs                  eingeschränkte Coding-Tools
├── tool_registry.rs                 Tool-Allowlist/Dispatch
├── mcp.rs                           MCP-Subprozessbrücke
├── download.rs                      Modelldownload/Resume/Checksums
├── model_catalog.rs                 Modellkatalog
├── model_registry.rs                Modellregistrierung
├── model_sources.rs                 lokale Modellquellen/Scan
├── gguf_registry.rs                 GGUF-Registry
├── staging.rs                       atomare Staging-/Commit-Pipeline
├── storage.rs                       Speicher-/Kompressionsplanung
├── persistence.rs                   Sitzungs-/Dateipersistenz
├── learning.rs                      kuratierte Lerndaten
├── kernel_ffi.rs                    FFI zum triAI-Kernelmodul
├── kernel_worker.rs                 Userspace-Speicher-/VRAM-Simulation
├── moe/mod.rs                       MoE-Unterstützung
├── speculative/mod.rs               Speculative-Decoding-Logik
└── bin/
    ├── stress_parallel.rs           Parallelstress-Programm
    ├── tri_chunk_bench.rs           Chunk-Benchmark
    ├── tri_model_pack.rs            Modellpacker
    └── tri_quality_gates.rs         Qualitätsgate-Programm
```

## 4. triAI-Engine – persistente Daten

```text
data/
├── knowledge/
│   └── tri-routing-knowledge.json       35.623 B
│       Kuratierte Routing-Fakten; keine Rohprompts als Wahrheit behandeln.
└── learning/
    └── model-learning-database.json      6.190 B
        Lerndaten/Ergebnisse; kein Ort für Secrets oder freie Sessions.

models/
├── manifest.json                         1.119 B
│   Modellübersicht
├── primary-mini-ingried-qwen3-8b-q4_k_m.gguf
│                                           5.027.784.352 B
│   Primäres lokales Modell
├── fallback-dolphin3-llama31-8b-q4_0.gguf
│                                           4.661.223.296 B
│   Fallback-Modell
├── test-stress-model-20gb.gguf
│                                          21.474.836.504 B
│   Last-/Stresstestmodell; nicht als Standardmodell laden
├── packed-fallback/
│   ├── manifest.json                     31.660 B
│   ├── tensor_index.json                 87.593 B
│   └── chunks/chunk_0000.zst … chunk_0066.zst
│       zstd-gepackte Fallback-Tensor-Chunks
└── packed-v2/
    ├── manifest.json                     38.811 B
    ├── tensor_index.json                119.001 B
    └── chunks/chunk_0000.zst … chunk_0074.zst
        zstd-gepackte Primärmodell-Tensor-Chunks
```

Die Modellhierarchie ist die bei weitem größte Datenklasse (rund **39 GiB**).
Original-GGUF, Pack-Manifest und Tensorindex müssen stets als zusammengehörige
Kette behandelt werden. Chunk-Dateien dürfen nicht einzeln gelöscht oder
verschoben werden, solange ein Manifest darauf zeigt.

## 5. triAI-Engine – Betriebs- und Metadaten

```text
config/
└── example.toml                     Beispiel für Server, Worker, Pfade, Reserven

deploy/
└── tri-ai-engine.service            systemd-Unit als Installationsvorlage

docs/
├── ARCHITECTURE.md                  Sollarchitektur
├── API.md                           API-Dokumentation
├── OPERATION.md                     Betriebshinweise
├── USAGE.md                         Nutzungsabläufe
├── RELEASE_CHECKLIST.md             Release-Gate
├── STATUS_AND_PLAN.md               Status/Plan
├── COMPLETION-PLAN.md               Abschlussplan
├── GATE-ANALYSIS.md                 Gateanalyse
├── OPTIMIZATION-STRATEGY.md         Optimierungsstrategie
├── ABSCHLUSSBERICHT.md              Abschlussbericht
└── evidence/phase-0 … phase-5       Phasenbezogene Evidenz

tests/
├── integration.rs                   Prozess-/HTTP-Integration
├── warmup_integration.rs            Warmup-Integration
└── chunk_cache_verification.rs      echte Archivtests; ohne Archiv übersprungen

benches/
├── startup.rs
├── inference.rs
├── chunk_load.rs
└── warmup.rs
```

### Flüchtige bzw. regenerierbare Engine-Daten

```text
target/                              Cargo-Buildcache und Binaries
staging/                             temporäre Download-/Commit-Daten (per Config)
tri-ai-events.jsonl                 Laufzeit-Evidenzlog (per Config)
.worktrees/triAI-engine-full/       separater Arbeitsbaum/Kopie
kernel/*.o, *.ko, *.mod*, modules.order, Module.symvers
                                     Kbuild-Ausgaben
```

## 6. HIXX-native – vollständiger Datenbaum

```text
hixx-native/
├── Cargo.toml                       Rust-Paket hixx-server
├── Cargo.lock                       gelockte Abhängigkeiten
├── Makefile                         build/load/run/status
├── hixx.pid                         historischer PID-Hinweis, flüchtig
├── hixx_daemon.log                  historisches Daemonlog, flüchtig
├── build/                           leerer reservierter Buildordner
├── src/
│   ├── main.rs                      Tokio-/Hyper-Prozess und HTTP-Bindung
│   ├── api.rs                       /health, /status, /v1/chat/completions
│   ├── ipc.rs                       ioctl/mmap-Client zum HIXX-Modul
│   └── ipc/structs.rs               reservierte Shared-Struct-Definitionen
├── kernel/
│   ├── Makefile                     Kbuild-Einstieg
│   ├── tri_ai_worker.c              gebauter HIXX-Shared-Memory-Treiber
│   ├── hixx_worker.c                nicht eingebundener Legacy-/Alternativtreiber
│   ├── tri_ai_worker.ko             geladenes/ladbares Modul-Artefakt
│   ├── tri_ai_worker.o              Objektdatei
│   ├── tri_ai_worker.mod*           Kbuild-Modulmetadaten
│   ├── Module.symvers               Symbolmetadaten
│   └── modules.order                Modulreihenfolge
├── scripts/
│   └── hixx_tune_system.sh          Tuning-Skript
└── target/                          Cargo-Buildcache und hixx-server-Binary
```

### HIXX-Laufzeitdaten außerhalb des Projektbaums

```text
/dev/tri_ai_worker                  Character Device des aktuell geladenen Moduls
                                    Berechtigung nach make load: root:users, 0660
/tmp/hixx-server.log                manueller Prozesslogpfad aus dem Smoke-Test
Kernel-Log (journalctl -k/dmesg)    Modul- und mmap-Diagnostik
```

`hixx.pid`, `hixx_daemon.log` und `/tmp/hixx-server.log` sind keine
verlässliche Prozessverwaltung. Für Dauerbetrieb braucht HIXX eine eigene
systemd-Unit mit `RuntimeDirectory`, `Restart=on-failure` und einem
deterministischen Logziel.

## 7. Datenflüsse

```text
Original-GGUF
   ├──> Modellkatalog / Manifest
   ├──> Chunk-Packer ──> packed-*/manifest + tensor_index + *.zst
   └──> llama-server (über WorkerSupervisor)

HTTP-Request
   ├──> triAI HTTP/Auth/Prompt-Hooks
   ├──> exklusiver Engine-/Worker-Slot
   ├──> llama-server auf Loopback
   └──> OpenAI-normalisierte Antwort + redigierte Evidence

HIXX HTTP-Request
   ├──> hixx-server/api.rs
   ├──> hixx-server/ipc.rs
   ├──> /dev/tri_ai_worker
   └──> 4-MiB-vmalloc_user-Puffer (derzeit ohne Nutzdatenprotokoll)
```

## 8. Kritische Integrationsgrenze

```text
triAI-Engine/kernel/tri_ai_worker.c
    └── Magic 't' (0x74), Stats/Hugepage/CPU/Prefetch/Evict

hixx-native/kernel/tri_ai_worker.c
    └── Magic 'H' (0x48), Init/Submit/Status/mmap
```

Beide implementieren denselben Namen für Modul und Gerät, aber verschiedene
APIs. Sie dürfen nicht gleichzeitig geladen oder als austauschbar behandelt
werden. Der Datenbaum enthält daher zwei getrennte Kernelzweige, keine
gemeinsame ABI-Schicht.

## 9. Backup-/Aufbewahrungsklassen

| Klasse | Beispiele | Behandlung |
| --- | --- | --- |
| Unersetzliche/quellrelevante Daten | `src/`, `config/`, `deploy/`, `docs/`, `data/`, Manifeste, Skripte | Versionieren und sichern. |
| Große, reproduzierbare Nutzdaten | GGUFs und `packed-*` | Prüfsummen/Manifest sichern; mindestens eine vollständige Kopie. |
| Temporäre, wiederherstellbare Daten | `target/`, `*.o`, `*.ko`, `*.mod*`, `staging/` | Nicht regulär sichern; sauber neu bauen. |
| Betriebsnachweise | JSONL-Eventlog, systemd-/Kernellogs | Rotieren, Zugriffsrechte beachten, Aufbewahrungszeit festlegen. |
| Sensible Konfiguration | tatsächliche `TRI_AI_AUTH_TOKEN`-Quelle/Environment | Nie in Beispielkonfiguration, Logs oder unverschlüsselte Backups schreiben. |

## 10. Nicht Teil dieses Baums

`/home/carlos/PROJEKTE/Der WEGWEISER/` ist ein unabhängiges Projekt mit
eigenem Repository und sensitiven Credentials. Es gehört weder funktional noch
datenmäßig zu triAI oder HIXX und wurde hier nicht inventarisiert.
