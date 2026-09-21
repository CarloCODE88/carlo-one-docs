# CTO-Freigabe: Fusion Foundation – Architektur- und Backend-Entscheidungsfreeze

**Status:** Freigabe mit Bedingungen
**Datum:** 2026-09-21
**Genehmigt durch:** CTO (Owner)
**Archiviert:** aus dem Team-Chat übernommen; Überschriften-Nummerierung aus dem Chat-Format normalisiert (2–9), Inhalt wortgetreu.

---

## 1. Ziel der Freigabe

Diese Freigabe legt die Architektur- und Entscheidungsgrundlage für die CarloONE Fusion fest. Sie dient als verbindlicher Rahmen für die nächste Umsetzung und setzt die bisherige, überzogene CTO-Strategie aus dem legacy-Fusionplan aus dem operativen Kontext heraus.

Die Freigabe basiert auf den verifizierten Source-Baselines und technisch belastbaren Dokumenten:

- `carlo-one-docs/architecture/TECHNICAL-BASELINE.md`
- `carlo-one-docs/architecture/TRIAI-ENGINE-ARCHITECTURE.md`
- `carlo-one-docs/operations/HIXX-SERVER-ARCHITECTURE-AND-OPERATIONS.md`
- `carlo-one-docs/operations/TRI-HIXX-DATA-TREE.md`

Die legacy-CTO-Fusion-Planung bleibt strategischer Hintergrund, aber nicht die operative Wahrheit. Sie ist nur Intent, keine technische Freigabe.

## 2. Verbindlich freigegebene Architektur

### 2.1 Produktiver Backend-Pfad: triAI-Engine

Der productive Backend-Pfad ist die triAI-Engine im Userspace. Sie ist der einzige Pfad, der in den vorhandenen Source-Basen als konsistente und belastbare Implementierung identifiziert wurde.

Dazu gehören:

- Worker-Lifecycle und Readiness
- Exklusiver Model-Slot / State-Handling
- Ressourcenplanung
- GGUF-/Manifest-/Chunk-Handling
- Prompt-Guardrails
- Evidence/Observability
- lokale Orchestrationslogik

Die triAI-Engine ist die Grundlage für die nächste Umsetzung. Ihr Status ist „grundsätzlich tragfähig, aber nicht als fertiger Produkt-Stack zu behandeln“.

### 2.2 Isolierter Forschungs-/IPC-Pfad: HIXX

HIXX bleibt ein eigener, klar abgegrenzter Prototyp und darf nicht als produktive Inference-Engine oder als Teil des produktiven Backend-Stacks behandelt werden.

HIXX bleibt nur dann weiterverfolgt, wenn:

- ein definierter Modul-/Gerätenamensraum gewählt wurde
- eine Versionierung des ABI festgelegt wurde
- ein Queue-/Result-Protokoll und Worker-Lifecycle definiert wurden
- ein klarer Betriebs- und Besitzkontext feststeht

Empfohlene Vorgabe:

- HIXX soll als eigener Namespace geführt werden (`hixx_ipc_worker` / `/dev/hixx_ipc_worker`), nicht als parallel laufende Instanz mit der triAI-Namenskonvention.

### 2.3 Lokales Backend / Orchestrationsschicht: nexus-qodex-fusion

Die bereits erbaute und getestete `nexus-qodex-fusion`-Schicht ist ein eigenständiger Backend-/Orchestrationslayer und muss ausdrücklich in die Architektur aufgenommen werden. Sie ist nicht als „nebenbei“ oder als vernachlässigbare Nebenentwicklung zu behandeln.

Die Freigabe sieht vor:

- `nexus-qodex-fusion` ist entweder
  - a) lokale Backend-/Orchestrationsschicht, die mit triAI als modellischem Worker spricht,
  - b) oder ein eigener Adapter-/Gateway-Layer,
  - c) oder ein späteres integratives Teilmodul, das in einer klaren Architektur konsistent eingebunden wird.

Nicht zulässig:

- duale, unbenannte Backend-Stapel ohne klare Verantwortlichkeiten
- parallele Backend-Implementierungen ohne Architektur- und Besitzdefinition
- Unterstellungen, dass die Daemon-Logik und die triAI-Logik identisch oder umsetzungsreif gleichzeitig wären

## 3. Entscheidungsgate Phase 0 – verbindlich

Die folgende Entscheidung gilt als verbindliches Phase-0-Gate:

### 3.1 Backend-Topologie muss entschieden werden

Vor weiterer Umsetzung muss eine der folgenden Optionen offiziell gewählt sein:

- **Option A:** `nexus-qodex-fusion` ist das lokale Backend / API / Scanner / Catalog / Orchestrations-Layer; triAI bleibt der Modellsupervisor
- **Option B:** `nexus-qodex-fusion` wird in triAI integriert oder durch einen klaren Adapter ersetzt
- **Option C:** `nexus-qodex-fusion` wird als separate Schicht verworfen und aus der Produktarchitektur entfernt

Empfehlung:

- Option A ist die technisch sauberste und in der aktuellen Lage am robustesten, weil die Daemon-Workflows bereits als lauffähig und wiederverwendbar identifiziert wurden.

### 3.2 HIXX-Namespace-Entscheidung

Vor weiterer HIXX-Weiterentwicklung muss der Modul-/Gerätenamensraum abgeschlossen sein:

- `hixx_ipc_worker` / `/dev/hixx_ipc_worker` als isolierter Pfad
- oder andere offiziell definierte Namespace-Lösung

Diese Entscheidung ist jetzt erforderlich und low-risk.

### 3.3 Maschinen-/Runtime-Entscheidung

Vor jedem weiteren ingenieurtechnischen Phase-Start muss festgelegt werden:

- Welche Phasen auf dieser Box laufen
- Welche Phasen auf der GPU-/Produktionsmaschine laufen
- Welche Phasen ausschließlich in einem Team- oder Eigentums-Kontext mit den entsprechenden System- und Kernel-Voraussetzungen laufen

Kernel- und Systemd-Arbeit auf dieser Box ist derzeit nicht als verifizierbarer Laufpfad freigegeben.

## 4. Ausführungsverbote

Die folgenden Punkte sind bis zur Entscheidungsfreigabe verboten:

- keine neue UI-/Tauri-Implementierungsphase
- keine Premium-/Cloud-Sync-/Canvas-Expansion
- keine LAN-/Remote-Exposition
- keine Annahme, HIXX sei bereits eine Inference-Engine
- keine klassische „statische Einbettung“ ohne Architekturentscheidung
- keine Weiterführung von Backend-Parallelismus ohne eindeutige Verantwortlichkeiten
- keine Produkt-Claims auf Basis von ungeprüften Zahlen

## 5. Freigegebene Phasenfolge

### 5.1 Phase 0 – Architektur- und Topologie-Entscheidung

Ziel:

- klare Backend-Topologie
- klare Namen-/ABI-Grenze
- klare Entscheidung für triAI vs HIXX vs Daemon

Erforderliche Inhalte:

- `nexus-qodex-fusion` als Backend-/Orchestrationslayer oder als klar benannter Adapter
- HIXX-Namespace freigeben
- target-machine mapping
- no-go-Liste für produktive Exposition

### 5.2 Phase 1 – triAI Backend-Härtung

Ziel:

- aktuelle triAI-Engine als produktiven Backend-Pfad stabilisieren
- Quality Gates, Lint, Validierung und Execution Contract
- keine Premium- oder UI-Weiterentwicklung

Erwartete Handlungen:

- `--no-network` korrekt definieren oder entfernen
- globale Lint-/Warnungs-Ausnahmen reduzieren
- sichere Backend-Lebenszyklus- und Zustandslogik
- reproduzierbare Test-/Clippy-Kriterien

### 5.3 Phase 2 – HIXX kontrolliert weiterführen

Ziel:

- HIXX nur als isolierter, eigener Prototyp
- Queue-/Result-Definition erst dann, wenn Namespace und ABI entschieden sind

Nicht erlaubt:

- produktive Nutzung als Inference-Backend
- parallele Nutzung als produktiver Core-Pfad

### 5.4 Phase 3 – Runtime-/Operations-Härtung

Ziel:

- Services, Logs, Restart, Retention, Auth, Rate-Limits, TLS/Reverse-Proxy, Berechtigungen

### 5.5 Phase 4 – UI/Produkt-/Premium-Integration

Ziel:

- erst nach erfolgreichem Phase-0 bis Phase-3-Freeze

## 6. Korrigierte Prioritätsreihenfolge

Die tatsächliche Reihenfolge ist:

1. Backend-Topologie entscheiden
2. triAI stabilisieren
3. HIXX isolieren / ABI- und Namespace-Entscheidung
4. Runtime/Service-Sicherheit
5. erst danach UI/Produkt/Premium

Diese Reihenfolge ist verbindlich.

## 7. Risiken und Einschränkungen

Die wichtigsten Risiken bleiben:

- unklare Backend-Topologie
- Namens-/ABI-Kollision zwischen triAI und HIXX
- fehlender Hausverstand über das laufende Daemon-Backend
- unklare Maschinenzuordnung
- produktive Exposition vor Auth/Rate-Limits/TLS
- ungetestete Claims zu Testzahlen, Reife und Timeline

Diese Risiken sind nicht als „Laborsymptome“ zu bagatellisieren, sondern als Architekturgrenzen zu behandeln.

## 8. Entscheidungs- und Freigabe-Compliance

Die Umsetzung darf nur nach der Freigabe der folgenden Entscheidungspunkte weiterlaufen:

- Backend-Topologie: Daemon / triAI / HIXX-Verhältnis
- HIXX-Namespace / ABI-Beschränkung
- Target-Machine-Mapping
- Phase-1-Kriterien und Reproduzierbarkeit
- Keine LAN- oder Produktive-Exposition vor Auth/Rate-Limits

## 9. Schlussfolgerung

Die Fusion darf nicht als einheitlicher monolithischer Produktivstack gestartet werden. Die korrekte Grundlage ist:

- triAI als produktiver Backend-Kern
- `nexus-qodex-fusion` als eigenständiger lokaler Orchestrations-/Backend-Layer
- HIXX als klar getrennte, experimentelle IPC-Komponente bis zum ABI-/Namespace-Entscheidungsfest

Diese Freigabe stellt die notwendige Stabilisierung sicher und verhindert, dass Product-, Premium- und UI-Arbeit auf unzureichend geklärter Backend- und Maschinenbasis ablaufen.

Die Freigabe ist gültig, solange die oberen Entscheidungspunkte nicht neu bewertet und abgeändert werden. Die Pause bleibt in Kraft, bis die Architektur- und Backend-Topologie formal entschieden ist.
