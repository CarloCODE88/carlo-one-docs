# CARLO:ONE — Masterplan

**Status:** Draft v1.0 — Konsolidierung der Owner-Klarstellung vom 2026-09-21, wartet auf Owner-Ratifikation¹

**Datum:** 2026-09-21

**Quellen:** ARCHITEKTURPLAN.md, FORTSCHRITTSBERICHT.md, TUI_SPEC.md (Shared-Verzeichnis) ·
architecture/CTO-APPROVAL-FUSION-FOUNDATION.md, architecture/FUSION-FOUNDATION-PLAN.md,
architecture/TECHNICAL-BASELINE.md (Docs-Repo) · eigene Code-Reviews der Team-Sessions.

**Leseregel:** Dieses Dokument trennt drei Ebenen strikt — **verifizierter Ist-Stand** (am Code bzw. in
den Baseline-Dokumenten geprüft), **Plan** (vereinbart, noch nicht umgesetzt) und **offene
Owner-Entscheidungen**. Im Konfliktfall gilt die CTO-Freigabe „Fusion Foundation — Architektur- und
Backend-Entscheidungsfreeze“ vom 2026-09-21; dieser Masterplan widerspricht ihr an keiner Stelle.
Referenzen auf das ursprüngliche „Masterplan“-Dokument sind rekonstruiert (Fußnote ²).

---

## 1. Vision & Positionierung

CARLO:ONE ist eine **lokale KI-Entwicklungs-Workstation**: Sie findet lokale Modelle, versteht sie,
katalogisiert sie und macht sie über eigene APIs für Agenten und Oberflächen nutzbar — **ohne LM
Studio** oder einen anderen Fremd-Daemon dazwischen.

Das ursprüngliche Konzept bleibt unverändert bestehen (Owner-Klarstellung vom 2026-09-21): Der
nexus-qodex-fusion-Daemon mit Scanner, Modellkatalog, CARLO-Sicherheitsschicht, TUI, Conversations,
MCP und Multi-Agent ist das Produkt. **Nur für die lokale Modellnutzung** werden triAI-Engine und
HIXX-Server integriert — triAI als Modellsupervisor, HIXX als isolierter Forschungspfad. Es ist kein
Umbau des Produkts um triAI herum; alle anderen Produktbausteine bleiben wie ursprünglich geplant.

Die Engine ist eine **„Sicherheitsaussagen tragende“ Engine**: Sie darf Dinge tun — aber alles läuft
durch ein bewertendes Tor (CARLO-Policy), das jede privilegierte Aktion einzeln prüft. Grundhaltung
ist Verbot: Ohne explizite Freigabe startet der Daemon nichts, und nichts verlässt die erlaubten
Pfadwurzeln.

Für wen: zuerst wir selbst — ein Werkzeug für den täglichen lokalen Modell- und Agentenbetrieb. Ein
Vertriebsmodell ist nicht entschieden; es existieren kein Zahlungsweg, keine Produkte, keine Umsätze.

## 2. Produktarchitektur (Zielbild)

```
   Oberflächen          TUI (Ratatui)  →  später GUI        MCP (Agenten)
                          │  REST/WS                  │  stdio
   ┌──────────────────────▼───────────────────────────▼───────────────┐
   │  Daemon nexus-qodex-fusion  —  DAS PRODUKTZENTRUM                │
   │  REST/WS-API · GGUF-Scanner · Modellkatalog (SQLite/FTS5) ·      │
   │  CARLO-Sicherheitsschicht · Conversations · MCP-Server ·         │
   │  Multi-Agent                                                     │
   └──────────────────────────────┬───────────────────────────────────┘
                                  │  Proxy-Integration /api/engine/*
                                  │  (HTTP, Loopback, Token-Weitergabe)
   ┌──────────────────────────────▼───────────────────────────────────┐
   │  triAI-Engine  —  Modellsupervisor für lokale Modellnutzung      │
   │  Worker-Lifecycle/Readiness · exklusiver Model-Slot ·            │
   │  Ressourcenplanung · GGUF-/Manifest-/Chunk-Handling              │
   │  (Staging/Rollback) · Prompt-Guardrails · Evidence               │
   └──────────────────────────────┬───────────────────────────────────┘
                                  │  Loopback
                        llama.cpp-Worker (lokale Inferenz)

   Daneben, isoliert, NIE produktiv:
   HIXX — HTTP-zu-Kernel-IPC-Prototyp, eigener Namespace hixx_ipc_worker
```

- **Oberflächen.** Die TUI ist ein Subkommando des Daemons und spricht ausschließlich mit dessen
  API; die GUI kommt später auf denselben Vertrag. MCP gibt Agenten eine stdio-Schnittstelle
  dieselben Fähigkeiten.
- **Daemon = Produktzentrum.** REST/WebSocket-API, GGUF-Scanner mit eigenem Parser, Modellkatalog,
  CARLO-Sicherheitsschicht, Conversations, MCP-Server und Multi-Agent leben hier. Der Daemon ist die
  einzige Quelle der Wahrheit für Katalog, Konversationen und Sicherheitsentscheidungen.
- **triAI = eingebundene Komponente für lokale Modellnutzung.** Sie supervidiert die Ausführung:
  Worker-Lifecycle, exklusiver Model-Slot, Ressourcenplanung, GGUF-/Manifest-/Chunk-Handling mit
  Staging/Rollback, Prompt-Guardrails, Evidence/Observability. **Kein Ersatz für den Daemon, kein
  zweites Backend, kein Dual-Stack.** Diese Rollenverteilung entspricht der Topologie Option A der
  CTO-Freigabe; durch die Owner-Klarstellung inhaltlich entschieden, formal bestätigt werden muss
  sie noch (§ 8).
- **llama.cpp-Worker** laufen unter triAI-Orchestrierung auf Loopback und werden nie direkt
  exponiert.
- **HIXX** bleibt Forschungspfad für Kernel-IPC — eigener Namespace, eigener Queue-Vertrag, nie
  produktiver Inferenzpfad und nie Teil des Produkt-Backends.

## 3. Der unveränderte Produkt-Roadmap-Kern

Diese Reihenfolge stammt aus dem ursprünglichen Plan und bleibt unverändert. Einordnung nach CTO-
Freigabe: Die Punkte 3.1–3.3 sind bei Topologie A die **Backend-Stabilisierung innerhalb der
verbindlichen Phasenfolge** (§ 5); die Oberflächen- und Agenten-Anteile (TUI, Chats-Screen, MCP,
Multi-Agent) stehen hinter dem Backend- bzw. UI-Gate (Phase 4).

**3.1 Scanner-Härtung (F1–F3).** *Was:* globales Element-/Byte-Budget pro GGUF-Datei (statt nur pro
Array), Feldlängen-Caps für Metadaten-Strings, saturierende u64→i64-Umwandlungen; die niedrigeren
Befunde F4–F7 (TOCTOU-Fenster, ungebundene Verzeichnislisten, verschluckte Fehler) folgen danach.
*Warum:* Die unabhängige Parser-Prüfung bewertet den Parser als nicht abstürzbar, aber aushungerbar
— ein einziger übergroßer Datensatz kann Speicher aufblähen, und gefälschte Werte landen falsch im
Katalog; falsche Katalogdaten sind gefährlicher als ein Absturz, weil man sie für echt hält.
*Fertig ist:* nichts — die Befunde sind dokumentiert und stehen bewusst ganz oben.

**3.2 Security Teil 2 (Audit/API + Scanner ans Policy-Gate + Auth-Entscheid).** *Was:* append-only
Audit-Log mit Secret-Redaction (eigene Migration), /api/security/*-Endpunkte (Prüfung, Audit,
Policy-Ansicht), Anschluss des Model-Scanners an das Pfad-Gate — und die Auth-Entscheidung als Teil
der Expositions-Politik (§ 5, § 8). *Warum:* Die CARLO-Policy ist als Prüfpunkt gebaut, aber nichts
ist angeschlossen — der Scanner liest heute Pfade, ohne das Gate zu fragen; eine Sicherheitsaussage
ohne Verkehr ist keine. *Fertig ist:* Security Teil 1 — Policy-Modell mit Whitelist/Blocklist
(Blocklist-Vorrang), Pfad-Gate gegen „..“- und Symlink-Ausbruch, Stufen strict/standard/permissive,
Fail-loud-Konfiguration; Audit und Sandbox existieren als Stubs.

**3.3 KPI-Massentest.** *Was:* Scan-Test mit 1.000 synthetischen GGUF-Dateien (Scan-Zeit,
Katalogwachstum, Rescan-Idempotenz) plus Messläufe für RAM-Leerlauf und Kaltstart. *Warum:* Die
3-Sekunden-Zusage für 1.000 Modelle ist heute eine Annahme, kein Ergebnis — der bisherige
Smoke-Test deckt zwei Dateien ab, und die Freigabe verbietet Produkt-Claims auf ungeprüfte Zahlen.
*Fertig ist:* der Smoke-Test-Aufbau (Fixture-Generator, Rescan-Nachweis) als Verfahren und Team-Skill.

**3.4 TUI Stufe 1.** *Was:* Screens Models, Scan und Footer-Statusleiste als Subkommando des
Daemons, Feature-Gate `tui`, `--selftest` für TTY-freie Verifikation; die verbindliche Spec
(Keymap-Regeln, Testkatalog, Modularchitektur) liegt vor. *Warum:* Erste eigene Oberfläche für
Katalog und Scan — und die verifizierte Grundlage, auf der Chats und alle späteren Screens
aufsetzen. *Fertig ist:* nur die verbindliche Spec; TUI-Code existiert nicht.

**3.5 Conversations-API + Chats-Screen.** *Was:* Endpunkte für Konversationslisten, Nachrichten und
FTS5-Volltextsuche plus die fehlenden DB-Helfer; daran anschließend der Chats-Screen der TUI.
*Warum:* Konversationsspeicher und Suche sind seit dem ersten Datenbankschema vorgesehen — ohne
Endpunkte hätte ein Chats-Screen keinen Inhalt. *Fertig ist:* das DB-Schema (Conversations,
Messages, FTS5-Index) inklusive CRUD-Helfern im DB-Layer; die Endpunkte fehlen.

**3.6 MCP-Server (12 Tools).** *Was:* MCP-Server-Skelett mit JSON-RPC über stdio und dem
Tool-Katalog laut Masterplan § 8 (12 Tools²). *Warum:* Agenten — einschließlich der Team-Agenten —
bekommen Katalog- und Betriebsschnittstelle ohne HTTP-Umweg; das ist die Grundlage der
15+-Tools-KPI (§ 7). *Fertig ist:* Stub-Module in der Daemon-Struktur, kein MCP-Code.

**3.7 Multi-Agent-Skelett.** *Was:* Agent-Trait und Supervisor mit Agent-Auswahl und Handoff.
*Warum:* Mehrere Agenten, die dieselbe Daemon-API nutzen, sind der eigentliche Zweck der
Workstation; das Skelett definiert diese Verträge früh. *Fertig ist:* Stub-Module, kein Code.

## 4. Integration triAI + HIXX für lokale Modellnutzung

Der einzige neue Baustein gegenüber dem ursprünglichen Plan. Stand: Plan, noch nicht umgesetzt.

**Was die Proxy-Integration konkret braucht:**

- **HTTP-Client im Daemon, Loopback-only.** triAI wird nie an eine externe Schnittstelle gebunden;
  der Daemon bleibt die einzige öffentliche API.
- **Proxy-Routen `/api/engine/*`.** OpenAI-kompatible Anfragen laufen durch den Daemon zu triAI;
  triAI normalisiert sie zur Ausführung, der Daemon behält Katalog, Sicherheit und Accounting.
- **Token-Weitergabe.** Das Authentifizierungsmodell des Daemons reicht Tokens an triAI weiter —
  es gibt keinen zweiten, unabhängigen Auth-Stapel.
- **Katalog-Abstimmung.** Der Daemon-Katalog ist die Quelle der Wahrheit (welche Modelle existieren,
  Pfade, Metadaten); triAI verwaltet nur den Ausführungszustand (Slot, Lifecycle) und meldet ihn
  zurück. triAI lädt nur Modelle, deren Pfad der Katalog führt — und Modellpfade laufen künftig durch
  das Pfad-Gate (§ 3.2).
- **Betriebsverträge.** Readiness-/Health-Mapping (triAI-Zustand → Daemon-Status), Timeouts und
  Fehlerklassen; die Evidence/Observability von triAI wird über den Daemon abfragbar.

**Was sie nicht ist:**

- kein Umbau des Produkts um triAI herum und keine Übernahme der Daemon-Rollen durch triAI;
- kein Dual-Stack — die CTO-Freigabe verbietet Backend-Parallelität ohne eindeutige
  Verantwortlichkeiten ausdrücklich;
- keine statische Einbettung (dazu wäre eine eigene Architekturentscheidung nötig, die nicht
  gefallen ist);
- kein zweiter Modellkatalog in triAI.

**HIXX-Behandlung:**

- **Namespace:** Vorschlag `hixx_ipc_worker` / `/dev/hixx_ipc_worker` — formal zu bestätigen
  (§ 8). Die Umbenennung ist mechanisch begrenzt (wenige Trefferstellen) und berührt die ABI-Werte
  nicht.
- **Legacy-Treiber:** Der alternative `hixx_worker.c` ist verifiziert nicht referenziert, spricht
  ein anderes ABI und kann gegen die aktuellen Kernel-Header nicht bauen. Er wandert als
  Design-Referenz in `kernel/attic/` — behalten, nicht gepflegt, nicht gelöscht.
- **Queue-Vertrag: greenfield.** Das heutige Shared-Memory-Mapping ist fest dimensioniert und
  enthält keine Warteschlangenstruktur; ein versionierter Queue-/Result-Vertrag (Job-Lifecycle,
  Status, Fehlerpropagierung) wird neu definiert, sobald Namespace und ABI entschieden sind. Die
  ABI-Werte sind zwischen Rust- und C-Seite verdoppelt — ein gemeinsamer Test bzw. Golden-Wert ist
  Teil dieser Arbeit.
- **Maschinen-Split:** Userspace-Arbeit (ABI-Tests, Testsuite) auf der Team-Box; Kernel-Bau,
  Modul-Load, Shared-Memory-Smoke und Hugepages-Tuning ausschließlich auf der Owner-GPU-Maschine
  (§ 6).

**Modell-Läufe** finden ausschließlich auf der Owner-GPU-Maschine statt. Die großen GGUF-Dateien
(4,7–5 GB) überschreiten den Arbeitsspeicher der Team-Box; dort laufen kein Modell, keine Release-/
LTO-Builds und keine Kernel-Arbeit.

## 5. Verbindliche Gates & Verbote

Aus der CTO-Freigabe vom 2026-09-21, unverändert verbindlich:

1. Keine neue UI-/Tauri-Implementierungsphase vor dem Phase-0-bis-3-Freeze.
2. Keine Premium-/Cloud-Sync-/Canvas-Expansion.
3. Keine LAN-/Remote-Exposition in irgendeiner Form.
4. Keine Annahme, HIXX sei eine Inference-Engine; HIXX ist nie produktiver Pfad.
5. Keine „statische Einbettung“ ohne eigene Architekturentscheidung.
6. Kein Backend-Parallelismus ohne eindeutige Verantwortlichkeiten.
7. Keine Produkt-Claims auf Basis ungeprüfter Zahlen (Testzahlen, Reife, Timeline).

**Die 6 Mindestsicherheiten vor jeder Exposition** — Voraussetzung, sobald der Freeze endet:

1. Authentifizierungsmodell
2. Rate-Limiting und Request-Caps
3. Logs und Audit-Trails
4. TLS oder Reverse-Proxy-Platzierung
5. Service-Lifecycle und Restart-Policy
6. Getrennte Betriebsverantwortung für Backend und UI

**Verbindliche Phasenfolge:** 0 Topologie-Entscheidung → 1 triAI-Härtung (`--no-network` fail-closed
erzwingen oder entfernen, Lint-Ausnahmen reduzieren, reproduzierbare Test-/Clippy-Kriterien) →
2 HIXX-Isolation → 3 Runtime-/Ops-Härtung (systemd, Logs, Restart, Auth, Rate-Limits, TLS) →
4 erst dann UI/Produkt/Premium. Daemon-Backend-Stabilisierung (§ 3.1–3.3) läuft als Teil dieser
Folge.

## 6. Maschinen-Zuordnung

Vorschlag (formale Festlegung offen, § 8). Kernel- und systemd-Arbeit auf der Team-Box ist bis zur
Festlegung kein verifizierbarer Laufpfad.

| Arbeitsgebiet | Team-Box (2 Kerne, 4 GB RAM, /home ~300 MB) | Owner-GPU-Maschine | Owner-Eigentum |
|---|---|---|---|
| Code & Tests | Daemon-Entwicklung, lib-Tests, Scanner-/Policy-Arbeit, HIXX-Userspace-Tests | — | — |
| Build-Grenzen | nur dev-Profile (debug=0), keine Release-/LTO-Builds | Release-/LTO-Builds | — |
| Modell-Läufe | verboten (RAM) | alle Inferenz, 4,7–5-GB-GGUFs | — |
| Kernel & Ops | — | Kernel-Bau und -Load (clang/ld.lld, Header), insmod, Shared-Memory-Smoke, Hugepages, systemd/Logs/Restart | — |
| Betriebsentscheidungen | — | — | LAN, Ports, Lizenzfragen, Vertriebsmodell |

## 7. KPIs

Ziele aus dem Masterplan-Bestand, unverändert; ergänzt um die triAI-Testsuite. Stand 2026-09-21:
nichts davon gemessen.

| KPI | Ziel | Stand |
|---|---|---|
| RAM im Leerlauf | < 80 MB | nicht gemessen |
| Kaltstart | < 100 ms | nicht gemessen |
| Model-Scan, 1.000 Modelle | < 3 s | nicht gemessen (Smoke-Test deckt 2 Dateien) — Massentest geplant (§ 3.3) |
| WebSocket-Latenz | < 50 ms | nicht gemessen (WS heute Echo ohne Protokoll) |
| MCP-Tools | 15+ | 0 (nicht begonnen) |
| triAI-Testsuite reproduziert | belegte Zahl statt des 226/316-Widerspruchs | offen — frischer Messlauf nötig (§ 8) |

Messbar werden die KPIs mit dem Massentest (§ 3.3) bzw. mit der triAI-Härtung (§ 5, Phase 1).

## 8. Offene Entscheidungen

Die fünf Phase-0-Punkte der CTO-Freigabe:

1. **Backend-Topologie.** Durch die Owner-Klarstellung vom 2026-09-21 inhaltlich entschieden:
   Option A — der Daemon ist Kern des Produkts (Backend/API/Scanner/Katalog/Orchestration), triAI
   ist als Modellsupervisor integriert, HIXX bleibt isoliert. Optionen B (Integration/Adapter) und
   C (Verwerfen) gelten als nicht verfolgt. **Die formale Bestätigung im Phase-0-Gate steht noch
   aus.**
2. **HIXX-Namespace.** Vorschlag: `hixx_ipc_worker` / `/dev/hixx_ipc_worker`. Formale Entscheidung
   offen (laut Freigabe low-risk und sofort nachholbar).
3. **Maschinen-/Runtime-Zuordnung.** Vorschlag in § 6; formale Festlegung offen.
4. **Phase-1-Kriterien & Reproduzierbarkeit.** Frischer Messlauf für die triAI-Testsuite (belegte
   Zahl statt des 226/316-Widerspruchs), Clippy-Gate mit reduzierten globalen Lint-Ausnahmen,
   `--no-network` fail-closed erzwingen oder entfernen. Konkrete Kriterien offen.
5. **Expositions-Politik.** Das Verbot ohne die 6 Mindestsicherheiten (§ 5) ist verbindlich; die
   konkrete Ausgestaltung (Authentifizierungstyp, Token-Handling auch für die Proxy-Routen) ist
   offen und wird mit Security Teil 2 (§ 3.2) entschieden.

## 9. Status & nächste Schritte

**Verifizierter Ist-Stand (2026-09-21):**

- Daemon Phase 1 abgeschlossen: 82/82 Tests grün, 0 Warnungen @ Commit `4074b20` — Scanner
  (eigener GGUF-Parser v2/v3, idempotenter Katalog-Upsert), CARLO-Policy-Kern, REST-API,
  SQLite/FTS5-Schema.
- Fusion-Grundlage poliert und gemergt; CTO-Freigabe archiviert (Docs-Repo @ `3e75565`).
- hixx-native liegt als Quelle der Wahrheit im Team-Repo `carlo-one-core` @ `abefbc1`
  (Userspace- und Kernel-Teile).
- LICENSE im Docs-Repo vorhanden.
- Businessplan Rev 3 mit der CTO-Freigabe als verbindlichem Rahmen.

**Was unmittelbar nach der Freigabe startet:**

1. Owner-Ratifikation dieses Masterplans (der Status „Draft“ endet).
2. Formale Phase-0-Bestätigungen (§ 8): Topologie bestätigen, HIXX-Namespace festlegen,
   Maschinen-Mapping ratifizieren.
3. Erste Arbeitsschritte in der verbindlichen Reihenfolge: triAI-Härtung (Phase 1 der CTO-Folge)
   und — bei bestätigter Topologie A — Daemon-Backend-Stabilisierung beginnend mit der
   Scanner-Härtung F1–F3 (§ 3.1), gefolgt von Security Teil 2 (§ 3.2) und KPI-Massentest (§ 3.3).
4. Danach HIXX-Isolation (Phase 2) und Runtime-/Ops-Härtung (Phase 3), jeweils mit dem
   Maschinen-Split aus § 6.
5. UI/Produkt/Premium erst nach dem Phase-0-bis-3-Freeze (Phase 4): TUI Stufe 1,
   Conversations-API/Chats-Screen, MCP-Server, Multi-Agent nach § 3.4–3.7.

---

**Fußnoten**

¹ Ratifikation = ausdrückliche Bestätigung durch den Owner. Bis dahin ist dieser Plan die
Team-Hypothese auf Basis der Owner-Klarstellung, nicht eine ratifizierte Entscheidung.

² Das ursprüngliche „Masterplan“-Dokument liegt auf dieser Box nicht vor. Die Referenzen darauf —
„Masterplan § 8“ (12 MCP-Tools), „Masterplan 1.6“ (CARLO-Sicherheitsstufe) und die KPI-Tabelle —
sind aus den Arbeitsnoten ARCHITEKTURPLAN.md und FORTSCHRITTSBERICHT.md sowie aus Team-Sessions
rekonstruiert. Die konkreten 12 Tool-Definitionen aus § 8 selbst sind nicht rekonstruierbar; der
Tool-Katalog wird beim MCP-Skelett (§ 3.6) neu festgelegt.
