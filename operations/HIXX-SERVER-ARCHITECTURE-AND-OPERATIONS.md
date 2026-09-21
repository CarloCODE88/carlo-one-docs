# HIXX Server – Architektur, IPC-Vertrag und Betriebsdokumentation

**Stand:** 15. September 2026  
**Projektordner:** `/home/carlos/PROJEKTE/hixx-native`  
**Programm:** `hixx-server`  
**Kernelmodul:** `tri_ai_worker.ko`  
**Gerät:** `/dev/tri_ai_worker`  
**Standard-HTTP-Adresse:** `127.0.0.1:8765`

## 1. Zweck und tatsächlicher Reifegrad

HIXX ist gegenwärtig ein **funktionierender HTTP-zu-Kernel-IPC-Prototyp**. Er
zeigt den gesamten Weg von einer HTTP-Anfrage über Rust und einen Linux-
Character-Device-Treiber bis zu einem gemeinsamen, in den Prozess gemappten
Speicherbereich.

Wichtig: Er ist **noch keine Modell-Inferenz-Engine**. Ein `POST
/v1/chat/completions` validiert oder verarbeitet kein Prompt und erzeugt keinen
Text durch ein Modell. Er signalisiert dem Kernelmodul lediglich eine
angenommene Aufgabe. Der Kernel erhöht dabei einen Zähler. Die Antwort
`"Kernel task accepted"` bedeutet deshalb *angenommen*, nicht *berechnet*.

Das ist absichtlich als klare Systemgrenze zu verstehen. Die nächste
Ausbaustufe braucht einen echten Worker, ein Auftragsformat, Ergebnisübergabe
und die Anbindung an eine Inferenz-Engine (zum Beispiel llama.cpp), bevor die
OpenAI-kompatible API semantisch ehrlich als Inference API bezeichnet werden
kann.

## 2. Komponentenübersicht

```text
HTTP-Client
    |
    |  GET /health, GET /status, POST /v1/chat/completions
    v
+-----------------------------------------------------+
| Rust-Daemon: target/release/hixx-server             |
|                                                     |
| src/main.rs  - Tokio/Hyper-Server, Prozessstart     |
| src/api.rs   - HTTP-Routen und Fehlerübersetzung    |
| src/ipc.rs   - Dateideskriptor, ioctl, mmap, munmap |
+--------------------------+--------------------------+
                           |
                           | open + ioctl + MAP_SHARED mmap
                           v
                 /dev/tri_ai_worker
                           |
+--------------------------+--------------------------+
| Linux-Kernelmodul: kernel/tri_ai_worker.c           |
|                                                     |
| Character Device · ioctl ABI · 4 MiB shared buffer  |
| vmalloc_user() · remap_vmalloc_range()              |
+-----------------------------------------------------+
```

## 3. Quellstruktur und Verantwortlichkeiten

| Pfad | Aufgabe | Bemerkung |
| --- | --- | --- |
| `src/main.rs` | Startet Tokio/Hyper, öffnet IPC beim Start, bindet HTTP-Port. | Der Standard ist absichtlich Loopback-only. |
| `src/api.rs` | Implementiert `/health`, `/status`, `/v1/chat/completions`. | Übersetzt Kernel-IPC-Fehler in HTTP 503. |
| `src/ipc.rs` | Rust-Seite des binären Kernelvertrags. | Enthält die ioctl-Nummern, mmap und RAII-Aufräumen. |
| `kernel/tri_ai_worker.c` | Tatsächlich gebautes/laufendes Treibermodul. | Das ist die maßgebliche Kernelimplementierung. |
| `kernel/Makefile` | Kbuild-Einstieg für `tri_ai_worker.c`. | Baut gegen den laufenden Kernel. |
| `Makefile` | Projektbefehle `kernel`, `rust`, `load`, `run`, `status`. | Setzt Gerätezugriff auf Gruppe `users`, Modus `0660`. |
| `kernel/hixx_worker.c` | Älterer, nicht eingebundener Alternativtreiber. | **Nicht produktiv verwenden**; siehe offene Punkte. |

## 4. Start- und Laufzeitfluss

1. Das Kernelmodul wird geladen und reserviert eine Major/Minor-Nummer.
2. Beim Laden reserviert es **4 MiB** RAM mit `vmalloc_user()`.
3. Der Kernel legt `/dev/tri_ai_worker` an.
4. Der Rust-Daemon öffnet das Gerät mit Lese- **und** Schreibrechten.
5. Der Daemon sendet `INIT_BUFFER`; das Modul setzt Puffer und Zähler zurück.
6. Der Daemon mappt exakt 4 MiB mit `MAP_SHARED`, `PROT_READ | PROT_WRITE`.
7. Erst wenn ioctl und mmap erfolgreich sind, startet Hyper auf dem HTTP-Port.
8. Eine Chat-Completion führt `SUBMIT_TASK` aus.
9. `/status` ruft `GET_STATUS` auf und gibt die Kernel-Zählung aus.

Der kritische Startpunkt ist: Der HTTP-Server startet nicht halb-funktional.
Wenn `/dev/tri_ai_worker`, `INIT_BUFFER` oder `mmap` fehlschlagen, beendet sich
der Prozess mit einem konkreten Fehler.

## 5. Der ioctl-ABI-Vertrag im Detail

Linux kodiert ioctl-Requests nach diesem Schema:

```text
bits 31..30: Richtung     (_IOC_NONE=0, _IOC_READ=2, _IOC_WRITE=1)
bits 29..16: Größe        (Größe des Payloads)
bits 15..8 : Magic        (hier 0x48 / 'H')
bits 7..0  : Command-Nr.
```

HIXX nutzt folgende Requests:

| Operation | Kernel-Makro | Hexwert | Argument | Wirkung |
| --- | --- | ---: | --- | --- |
| Initialisieren | `_IO(0x48, 0)` | `0x00004800` | keines | Löscht den gemeinsamen Puffer und setzt Zähler zurück. |
| Aufgabe einreichen | `_IO(0x48, 1)` | `0x00004801` | keines | Erhöht im Prototyp die Zahl akzeptierter Aufgaben. |
| Status lesen | `_IOR(0x48, 2, int)` | `0x80044802` | `int *` | Kopiert die aktuelle Warteschlangentiefe zu Userspace. |

Die Werte sind sowohl in Rust getestet als auch in C mit den Linux-Makros
definiert. Das ist wichtig, denn frühere Versionen verschoben Command-Nummer
und Größe beide in Bit 16. Dadurch passten User- und Kernelseite nicht
zusammen. Außerdem wurde der Rückgabewert von `ioctl` ignoriert. Beides ist
behoben.

### Richtung der Daten

`_IOR` bedeutet aus Sicht des Kernels: **der Kernel schreibt in Userspace**.
Darum verwendet `GET_STATUS` `copy_to_user()`. Ein Statuswert darf nicht bloß
als Systemcall-Rückgabewert zurückgegeben werden: Negative Rückgabewerte sind
für `errno` reserviert und der Rust-Aufrufer übergibt ausdrücklich einen
Pointer.

## 6. Shared-Memory-Mapping: warum es jetzt funktioniert

### Früherer Fehler

Der ursprüngliche Treiber reservierte 4 MiB mit `kmalloc()` und übersetzte die
virtuelle Adresse per `virt_to_phys()` für `remap_pfn_range()`. Das ist für
einen so großen Puffer unzuverlässig:

- 4 MiB muss physisch zusammenhängend verfügbar sein.
- Große zusammenhängende Kernel-Allokationen scheitern oft unter Last oder
  nach längerer Laufzeit an Fragmentierung.
- `virt_to_phys()` und PFN-Remapping sind der falsche Abstraktionsgrad für
  einen normalen Software-Shared-Puffer.

### Aktuelle Lösung

Der Treiber nutzt jetzt `vmalloc_user(RING_BUFFER_SIZE)`. Diese API erzeugt
virtuell zusammenhängenden Speicher und markiert ihn explizit für das Mapping
in Userspace. Die Seiten müssen dabei physisch nicht zusammenhängend sein.

Das Mapping erfolgt mit:

```c
remap_vmalloc_range(vma, ring_buffer, 0)
```

Die Implementierung akzeptiert nur:

- Offset `0` (`vm_pgoff == 0`),
- exakt `4 * 1024 * 1024` Bytes,
- die vom Rust-Client erwartete einzige Mapping-Größe.

Das verhindert Teil- oder Offset-Mappings, die momentan keinen definierten
Protokollvertrag hätten. Das Rust-Programm öffnet das Gerät mit `O_RDWR`; das
ist für eine schreibbare, shared Mapping-Region erforderlich. Ein nur lesend
geöffneter Dateideskriptor führte zuvor zu `EACCES`, noch bevor die mmap-
Methode des Treibers erreicht wurde.

## 7. HTTP-API

### `GET /health`

Antwort bei laufendem Daemon:

```json
{"status":"ok"}
```

Dies bestätigt den HTTP-Server selbst. Es ist bewusst ein Liveness-Check und
fragt nicht den Kernel ab.

### `GET /status`

Antwort bei erfolgreicher Kernel-Kommunikation:

```json
{"status":"running","kernel":"hixx-native","queue_depth":1}
```

Bei ioctl-Fehlern liefert die Route HTTP 503 statt erfundener Statuswerte.

### `POST /v1/chat/completions`

Der Endpunkt akzeptiert derzeit den Request-Körper nicht semantisch; er reicht
noch keine Nachrichten oder Modellparameter an den Kernel weiter. Bei
erfolgreichem `SUBMIT_TASK` antwortet er mit HTTP **202 Accepted**:

```json
{
  "id": "hixx-1",
  "object": "chat.completion",
  "choices": [
    {
      "index": 0,
      "message": {"role": "assistant", "content": "Kernel task accepted"},
      "finish_reason": "stop"
    }
  ]
}
```

Die Form ist clientfreundlich, aber noch keine echte Chat-Completion. Sobald
eine Worker-Pipeline existiert, sollte die Antwort erst nach einem echten
Ergebnis oder als klar ausgewiesener asynchroner Job erfolgen.

## 8. Sicherheitsgrenzen

### HTTP

Der Server bindet standardmäßig an `127.0.0.1:8765`, nicht an alle
Netzwerkinterfaces. Das ist wichtig, da es aktuell keine Authentifizierung,
Rate-Limits oder Request-Validierung gibt.

LAN-Binding ist nur absichtlich freizugeben:

```bash
HIXX_BIND_ADDR=0.0.0.0:8765 ./target/release/hixx-server
```

Vor einer solchen Freigabe sind mindestens Authentifizierung, eine
Anfragegrößenbegrenzung, strukturierte Request-Validierung und ein Reverse
Proxy mit TLS erforderlich.

### Character Device

`make load` setzt `/dev/tri_ai_worker` auf `root:users`, Modus `0660`. Damit
können Mitglieder der Gruppe `users` den Daemon ohne `root` ausführen. Die
frühere Einstellung `0666` wurde entfernt, weil jeder lokale Benutzer das
Kernelinterface hätte steuern können.

Die Gruppe ist konfigurierbar:

```bash
make load HIXX_DEVICE_GROUP=hixx
```

Für Dauerbetrieb ist eine eigene Gruppe `hixx` plus eine passende udev-Regel
sauberer als die allgemeine Gruppe `users`.

### Kernelgrenzen

- Das Modul prüft die ioctl-Magic.
- Der Treiber serialisiert Status- und Zähleränderungen per `mutex`.
- `GET_STATUS` kopiert nur ein `int` zurück.
- mmap ist auf einen expliziten Offset und eine explizite Größe eingeschränkt.

Der gemeinsame Puffer enthält aktuell **keine** definierte Struktur und wird
vom Daemon noch nicht gelesen oder geschrieben. Er ist deshalb ein reservierter
Transportkanal, kein fertiges Queue-Protokoll.

## 9. Bauen, Laden und Starten

Alle Befehle erfolgen im Projektordner:

```bash
cd /home/carlos/PROJEKTE/hixx-native

# Rust und Modul bauen
make all

# Modul neu laden; setzt Gerätegruppe und -modus
make load

# Daemon im Vordergrund starten
./target/release/hixx-server
```

Alternativ baut und lädt `make run` alles und startet anschließend den Daemon.
`make load` entlädt ein bereits geladenes `tri_ai_worker`-Modul. Der Befehl
soll deshalb nicht ausgeführt werden, während ein produktiver Daemon das Gerät
offen hält.

### Prüfen

```bash
curl --fail http://127.0.0.1:8765/health
curl --fail http://127.0.0.1:8765/status
curl --fail --request POST http://127.0.0.1:8765/v1/chat/completions \
  --header 'content-type: application/json' \
  --data '{"model":"hixx","messages":[{"role":"user","content":"ping"}]}'
```

Ein anschließendes `/status` muss eine um eins höhere `queue_depth` melden.

## 10. Verifizierte Checks dieser Übergabe

Folgende Checks wurden nach den Reparaturen ausgeführt:

| Check | Ergebnis |
| --- | --- |
| `cargo fmt` | erfolgreich |
| `cargo test` | 2/2 Tests erfolgreich |
| `cargo clippy -- -D warnings` | erfolgreich |
| `cargo build --release` | erfolgreich |
| `make kernel` | Kernelmodul erfolgreich gebaut |
| Modulentladen/-laden | erfolgreich |
| mmap des 4-MiB-Puffers | erfolgreich |
| `/health` | HTTP 200, JSON `status: ok` |
| `/status` | HTTP 200, Kernelstatus lesbar |
| Chat-POST | HTTP 202, anschließende Tiefe erhöhte sich auf 1 |

## 11. Code-Review-Ergebnis und verbleibende Arbeit

### Behobene kritische Probleme

1. **ABI-Mismatch zwischen Rust und C:** behoben durch identische `_IO`/`_IOR`
   Kodierung und einen Rust-Test auf die drei Hexwerte.
2. **Ignorierte Systemcall-Fehler:** `ioctl` und `mmap` werden nun geprüft;
   Startfehler enthalten die fehlerhafte Operation und den OS-Fehler.
3. **Ungeeignete 4-MiB-`kmalloc`-Allokation:** ersetzt durch
   `vmalloc_user()`.
4. **Fehlerhafte Statusübergabe:** Status nutzt jetzt `copy_to_user()`.
5. **Race Conditions im simplen Treiber:** Zählerzugriffe sind per Mutex
   serialisiert.
6. **Öffentlicher Netzwerk-Default:** von `0.0.0.0` auf `127.0.0.1` geändert.
7. **Weltweit schreibbares Gerät:** von `0666` auf gruppenbasiertes `0660`
   geändert.
8. **Stille HTTP-Fehler:** Kernel-IPC-Fehler erzeugen HTTP 503.

### Noch offen: funktionale Lücken

1. **Kein Inferenzworker.** `head` wächst, `tail` wird nie von einem Consumer
   weiterbewegt. Die Queue kann daher nur eingereichte Arbeit zählen.
2. **Kein Buffer-Layout.** Header, Ring-Slots, Sequenznummern, Längen,
   Ownership und Speicherbarrieren sind nicht spezifiziert.
3. **Keine Ergebnisroute.** Es gibt weder Streaming noch Polling noch
   Auftrags-ID/Ergebnisablage.
4. **Chat-Body wird nicht geparst.** `model`, `messages`, Tokenlimits und
   Fehlereingaben müssen bei der echten API validiert werden.
5. **Kein Lebenszyklus-Management.** Kein systemd-Service, keine udev-Regel,
   kein kontrollierter Shutdown und keine automatische Wiederherstellung nach
   Modulreload.
6. **Keine Produktionssicherheit.** Fehlende Authentifizierung, Limits,
   Metriken, Audit-Logs und TLS (bei LAN-Exposition).

### Noch offen: Codehygiene

`kernel/hixx_worker.c` ist ein zweiter, abweichender und aktuell nicht gebauter
Treiber. Er darf nicht neben `tri_ai_worker.c` als scheinbar gleichwertige
Implementierung liegen bleiben. Vor der nächsten Ausbaustufe sollte er entweder
gelöscht/archiviert oder bewusst als eigenständiges, getestetes Modul
integriert werden. Andernfalls drohen Build- und ABI-Verwechslungen.

## 12. Empfohlene nächste Ausbaustufe

Die richtige Reihenfolge ist:

1. Einen versionierten Shared-Memory-Header definieren (`magic`, ABI-Version,
   Ringgröße, Producer-/Consumer-Indizes, atomare Sequenzen).
2. Auftrags- und Ergebnis-Slots mit festen Maximalgrößen oder referenzierten,
   sicher verwalteten Puffern definieren.
3. Einen Userspace-Worker implementieren, der genau einen Auftrag konsumiert,
   an die gewählte Inferenz-Engine übergibt und das Ergebnis zurückschreibt.
4. Erst danach die HTTP-API echte Prompts entgegennehmen und eine reale
   Completion bzw. Server-Sent Events ausgeben lassen.
5. Zum Schluss systemd, udev, strukturierte Logs, Metriken und Authentifizierung
   ergänzen.

Der Kernel sollte dabei möglichst ein kleiner, stabiler Transport-/Isolations-
Layer bleiben. Modellladen, Tokenisierung, Scheduling und HTTP gehören in den
Userspace; dort sind sie wartbarer, testbarer und erheblich sicherer zu
iterieren.

---

# Ergänzung: Gesamt-Review triAI-Engine und HIXX-Server

## 13. Kurzfazit

Die beiden Projekte sind gegenwärtig **nicht integrierbar**, obwohl sie beide
ein Gerät namens `/dev/tri_ai_worker` und ein Modul namens `tri_ai_worker`
verwenden. Sie implementieren zwei verschiedene und inkompatible Kernel-ABIs.
Das ist die wichtigste Feststellung des Gesamt-Reviews.

| Bereich | Urteil | Begründung |
| --- | --- | --- |
| triAI-Engine, Userspace | weit entwickelt | Modell-Supervision, OpenAI-Kompatibilität, Sicherheitsgrenzen, Staging und viele Tests sind vorhanden. |
| triAI-Engine, Kernel-Experiment | nicht produktionsreif | privilegiertes, zweites ABI; nutzt große physisch zusammenhängende Allokationen und hat keinen sauberen Bezug zum HTTP-Pfad. |
| HIXX-Server, IPC-Prototyp | funktionsfähig | Rust↔Kernel↔HTTP wurde live geprüft. |
| HIXX-Server, Inferenz | nicht implementiert | Der Kernel zählt Aufgaben, führt aber keine aus. |
| Kombination | blockiert | Gleiches Modul-/Gerätenamen, unterschiedliche ioctl-Magic und Payloads. |

## 14. Architektur der triAI-Engine

```text
Client / Franz / OpenAI-kompatibler Client
              |
              v
triAI HTTP-Schicht (std::net, Auth, Limits, Routing)
              |
              v
Engine-Zustandsautomat + einzelner Model-Slot
              |
              v
WorkerSupervisor --> llama-server auf Loopback-Port
              |
              +--> Model-Katalog, GGUF-Registry, Downloads
              +--> Chunking, Staging, Warmup, Cache
              +--> Prompt-Hooks, Attachments, Tool-Registry
              +--> Evidence/Observability, Ressourcen-/Policy-Analyse
```

Die Engine ist bewusst als Userspace-Orchestrator gebaut. Ihr wertvollster,
bereits vorhandener Kern ist nicht ein Kernelmodul, sondern die Kombination aus
Model-Supervisor, Ressourcenplanung, GGUF-/Chunk-Handling und strikt
serialisiertem Modellslot.

### Stärken der Engine

- Der Zustand `Idle → Loading → Ready → Busy → …` ist explizit modelliert.
- Der Supervisor prüft Ports und Worker-Readiness, begrenzt Stderr und räumt
  fehlgeschlagene Prozesse auf.
- Externe Bind-Adressen brauchen einen Auth-Token; der Tokenvergleich erfolgt
  zeitkonstant.
- HTTP-Lese- und Schreibtimeouts sowie Größenlimits sind vorhanden.
- Prompt-Hooks sind deterministisch und blockieren Secret-/Cloud-Marker.
- Chunk-, Staging- und Wiederherstellungslogik hat ausführliche Tests.
- Der gesamte Testlauf hat **316 aktive Tests bestanden**; drei echte
  Chunk-Cache-Integrationstests bleiben mangels realem Archiv bewusst
  übersprungen.

## 15. P0: Namens- und ABI-Kollision zwischen beiden Kernelmodulen

| Eigenschaft | triAI-Engine | HIXX-Server |
| --- | --- | --- |
| Quelldatei | `triAI-Engine/kernel/tri_ai_worker.c` | `hixx-native/kernel/tri_ai_worker.c` |
| Modulname | `tri_ai_worker` | `tri_ai_worker` |
| Gerät | `/dev/tri_ai_worker` | `/dev/tri_ai_worker` |
| ioctl Magic | `'t'` / `0x74` | `0x48` / `'H'` |
| API | Hugepage, CPU-Affinität, Prefetch, Evict, Stats | Init, Submit, Status, 4-MiB-mmap |
| Privileg | jeder ioctl verlangt `CAP_SYS_ADMIN` | Gerätedateirechte steuern Zugriff |

Ein geladenes Modul kann nur eine dieser Implementierungen bereitstellen. Läuft
das HIXX-Modul, erhält die triAI-FFI bei `GET_STATS` `ENOTTY`; läuft das
triAI-Modul, scheitert HIXX bereits bei seiner Initialisierung/mmap. Beide
dürfen deshalb niemals denselben Namen oder dasselbe Device verwenden.

**Verbindliche Entscheidung vor weiterer Integration:**

1. Entweder das HIXX-Experiment als eigenes Modul/Gerät umbenennen, z.B.
   `hixx_ipc_worker` und `/dev/hixx_ipc_worker`, oder
2. die HIXX-IPC-Funktion als klar versionierten Teil des triAI-Kernelmoduls
   aufnehmen und einen einzigen gemeinsamen C-Header als ABI-Quelle pflegen.

Die zweite Variante ist nur sinnvoll, wenn ein Kerneltransport wirklich nötig
ist. Für die aktuelle lokale llama.cpp-Architektur ist die erste Variante
risikoärmer: Die triAI-Engine bleibt der produktive Pfad, HIXX bleibt ein
isoliertes Forschungsprojekt.

## 16. triAI-Engine: konkrete Review-Funde

### P1 – Warnungen und Clippy sind global deaktiviert

`src/lib.rs` enthält `#![allow(clippy::all)]` und `#![allow(warnings)]`. Das
unterdrückt auch neue Warnungen in Produktionscode und verhindert, dass ein
zukünftiges `cargo clippy -- -D warnings` ein belastbares Qualitätsgate ist.

**Empfehlung:** Beide globalen Attribute entfernen. Unvermeidbare Ausnahmen
lokal, mit Begründung und engstem Scope annotieren. Erst dann Clippy in die
Release-Checks aufnehmen.

### P1 – `--no-network` ist derzeit nur eine Statusmeldung

Der Schalter in `src/main.rs` schreibt lediglich `air-gapped mode enabled`.
Er verändert weder die HTTP-Bindung noch Download-Quellen, Worker-Start oder
Netzwerkzugriffe. Der Name suggeriert eine Sicherheitseigenschaft, die aktuell
nicht erzwungen wird.

**Empfehlung:** Einen `NetworkPolicy::Disabled`-Wert in `Config` einführen und
damit Downloads, externe Modellquellen, MCP-Subprozesse und Worker-Zieladressen
fail-closed blockieren. Zusätzlich mit einem Integrationstest beweisen, dass
kein ausgehender Verbindungsversuch erfolgt.

### P1 – Der Kernel-FFI-Pfad verlangt CAP_SYS_ADMIN

Das triAI-Kernelmodul verweigert alle ioctls ohne `CAP_SYS_ADMIN`. Der normale
unprivilegierte Engine-Daemon kann die FFI daher nicht verwenden. Das ist
inhaltlich sicherer als ein weltbeschreibbares Device, aber die aktuelle
Integration muss den Fehler eindeutig als *optional nicht verfügbar* behandeln
und darf daraus keinen Start- oder Inferenzfehler machen.

**Empfehlung:** Kernel-Optimierungen als optionalen Capability-Adapter führen,
nicht als Kernabhängigkeit. Privilegien nicht durch einen als root laufenden
HTTP-Server "lösen".

### P2 – Kernel-Hugepage-Mechanik ist mit Dauerbetrieb unvereinbar

Der triAI-Treiber allokiert große zusammenhängende Seitenblöcke per
`alloc_pages`. Bei größeren Werten scheitert das aufgrund von `MAX_ORDER` oder
Fragmentierung; außerdem gibt es keinen User-ABI zum Freigeben einzelner
Allokationen. Die interne Markierung `TRI_VRAM` repräsentiert außerdem keinen
realen VRAM-Transfer.

**Empfehlung:** Keine Modellgewichte über dieses Modul verwalten. Den
Userspace-Planer der Engine für VRAM-Entscheidungen nutzen; echte GPU-Residency
bleibt Aufgabe von llama.cpp/CUDA.

### P2 – globaler Engine-Mutex bedeutet bewusste, aber harte Serialität

Die HTTP-Schicht hält während einer Engine-Operation den globalen Mutex. Das
schützt den Ein-Modell-Slot zuverlässig, allerdings blockiert ein langsamer
Worker damit jede weitere steuernde Anfrage. Es ist aktuell eine korrekte
Single-User-Entscheidung, aber keine Mehrnutzer-Architektur.

**Empfehlung:** Für den aktuellen Rechner beibehalten. Bei Mehrnutzerbetrieb
eine explizite Warteschlange mit Obergrenze, Rückstauantworten und Cancel-
Semantik einführen statt mehrere Worker parallel zu starten.

## 17. HIXX-Server: Review-Funde

### Behoben

Die in den Abschnitten 5 bis 11 beschriebenen ABI-, mmap-, Fehlerbehandlungs-
und Berechtigungsfehler wurden behoben und live getestet.

### Noch offen

1. Der 4-MiB-Puffer hat kein Datenformat und wird nicht genutzt.
2. `tail` wird niemals fortgeschrieben; `queue_depth` ist ein Akzeptanzzähler,
   keine echte Queue.
3. Chat-Requests werden nicht geparst und erzeugen keine Inferenz.
4. Es fehlen Service-/udev-Installation, Shutdown und Restart-Policy.
5. Der alternative Treiber `kernel/hixx_worker.c` ist fachlich abweichend und
   nicht Teil des Builds; er muss archiviert oder bewusst integriert werden.

## 18. Empfohlene Zielarchitektur

```text
                 produktiver Pfad
Client --> triAI-Engine --> llama.cpp worker --> GPU
              |      ^
              |      +-- Modelle, Planung, Sicherheit, Observability
              |
              +-- optional: klar begrenzte Kernel-Telemetrie

                 isolierter Forschungszweig
Client --> HIXX IPC-Prototyp --> eigenes Modul + eigenes Device
                                --> erst später Worker/ABI-Experiment
```

Die triAI-Engine sollte der **einzige** produktive HTTP- und Model-Supervisor
bleiben. HIXX darf experimentieren, aber nicht parallel dieselbe Kernel-
Ressource beanspruchen. Eine spätere Integration braucht zuerst einen gemeinsamen
Versionierungsplan und End-to-End-Tests, niemals nur gleiche Dateinamen.

## 19. Priorisierte Maßnahmenliste

1. **Sofort:** Modul- und Device-Namenskollision auflösen.
2. **Sofort:** `--no-network` wirklich durchsetzen oder den Schalter entfernen.
3. **Kurzfristig:** globale `allow(warnings)`/`allow(clippy::all)` ablösen.
4. **Kurzfristig:** HIXX als getrennten Forschungszweig kennzeichnen und den
   inaktiven Alternativtreiber bereinigen.
5. **Mittelfristig:** echte Inferenz ausschließlich über den triAI-Supervisor
   anbinden; HIXX erst mit definiertem Queue- und Ergebnis-ABI erweitern.
6. **Vor LAN-/Produktivbetrieb:** systemd/udev, TLS/Reverse Proxy,
   Authentifizierung, Rate Limits und belastbare End-to-End-Tests abschließen.
