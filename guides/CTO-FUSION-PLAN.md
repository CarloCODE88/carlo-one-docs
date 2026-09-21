<longcat_tool_call>mcp__CherryFilesystem__write_5a77e628ae7b5c584242
<longcat_arg_key>content</longcat_arg_key>
<longcat_arg_value># 🚀 FINAL CARLO FUSION PLAN
## CarloONE 2.0 – Vollständige Integrations- und Business-Strategie

**Version:** 3.0 FINAL
**Datum:** 2026-09-21
**Status:** CTO-genehmigt, bereit für Umsetzung
**Verantwortlich:** CTO Office

---

## 📋 Inhaltsverzeichnis

1. [Executive Summary](#executive-summary)
2. [Architektur-Übersicht](#architektur)
3. [Open-Core-Modell & Repos](#open-core)
4. [Phase 0: ABI-Kollisionslösung](#phase-0)
5. [Phase 1: Engine-Einbettung](#phase-1)
6. [Phase 2: Job-Konvergenz](#phase-2)
7. [Phase 3: Telemetrie-Integration](#phase-3)
8. [Phase 4: Canvas + Design-KI](#phase-4)
9. [Phase 5: Premium-Integration](#phase-5)
10. [Repository-Struktur](#repo-struktur)
11. [Build & CI/CD](#build-cicd)
12. [Test-Strategie](#test-strategie)
13. [Sicherheit & Compliance](#sicherheit)
14. [Ressourcenplanung](#ressourcen)
15. [Qualitätsgates](#gates)

---

## 🎯 Executive Summary <a name="executive-summary"></a>

CarloONE 2.0 ist eine **Tauri-2-Workstation** für Developer und Creative Professionals. Sie integriert:

- **triAI-Engine** (316 Tests, Rust) – Statisch eingebettete KI-Engine
- **HIXX-Server** (Kernel-IPC) – Optionaler Telemetrie-Daemon für GPU/CPU-Metriken
- **VSCodium** – Editor-Integration für Code-Workflows
- **Canvas-System** – Design-Oberfläche mit KI-gestützten Varianten

**Geschäftsmodell:** Open-Core
- **Kern (Open):** Grundfunktionen, lokale Modelle, Community-Features
- **Premium (Privat):** Erweiterte KI-Modelle, Cloud-Sync, Profi-Features

**Kernprinzipien:**
- ✅ Keine Cloud-Abhängigkeit für Kernfunktionen
- ✅ Lokale Datenverarbeitung (Datenschutz)
- ✅ Statische Einbettung (Performance, <2ms Overhead)
- ✅ Strikte Trennung Open/Premium (Compliance)
- ✅ ABI-Kompatibilität (triAI + HIXX parallel)

---

## 🏗️ Architektur-Übersicht <a name="architektur"></a>

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     CARLOONE 2.0 – SYSTEMARCHITEKTUR                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    SHELL (Frontend)                             │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐  │   │
│  │  │   Code      │ │   Design    │ │   Musik     │ │  GameDev  │  │   │
│  │  │   (VSCodium)│ │  (Canvas)   │ │   (DAW)     │ │  (Engine) │  │   │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └─────┬─────┘  │   │
│  │         │               │               │              │        │   │
│  │  ┌──────┴───────────────┴───────────────┴──────────────┴─────┐  │   │
│  │  │              Tauri 2 IPC-Bridge                          │  │   │
│  │  └─────────────────────────┬────────────────────────────────┘  │   │
│  └────────────────────────────┼────────────────────────────────────┘   │
│                               │                                        │
│  ┌────────────────────────────┼────────────────────────────────────┐   │
│  │                    CORE (Backend - Rust)                         │   │
│  │                            │                                     │   │
│  │  ┌─────────────────────────┴─────────────────────────────────┐  │   │
│  │  │              SERVICE LAYER (Dienste)                       │  │   │
│  │  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐   │  │   │
│  │  │  │ EngineAdapter│ │  HixxTelemetry│ │  Job Orchestrator│   │  │   │
│  │  │  │  (statisch)  │ │  (optional)   │ │  (Zustandsautomat)│  │  │   │
│  │  │  └──────┬───────┘ └──────┬───────┘ └────────┬─────────┘   │  │   │
│  │  │         │                │                  │             │  │   │
│  │  │  ┌──────┴────────────────┴──────────────────┴──────────┐  │  │   │
│  │  │  │              ADAPTER LAYER                            │  │  │   │
│  │  │  │  ┌────────────┐ ┌────────────┐ ┌──────────────────┐  │  │  │   │
│  │  │  │  │  triAI     │ │   HIXX     │ │   VSCodium       │  │  │  │   │
│  │  │  │  │  Engine    │ │   IPC      │ │   Editor         │  │  │  │   │
│  │  │  │  │  (Krate)   │ │   (Dev)    │ │   (Integration)  │  │  │  │   │
│  │  │  │  └────────────┘ └────────────┘ └──────────────────┘  │  │  │   │
│  │  │  └───────────────────────────────────────────────────────┘  │  │   │
│  │  └─────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    PREMIUM LAYER (Optional)                       │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐  │   │
│  │  │ Canvas AI HQ │ │ Cloud Sync   │ │ Advanced Models          │  │   │
│  │  │ (3 Varianten)│ │ (End-zu-End) │ │ (GPT-5, Claude 4.5, etc.)│  │   │
│  │  └──────────────┘ └──────────────┘ └──────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 💼 Open-Core-Modell & Repos <a name="open-core"></a>

### Repository-Übersicht

| Repo | Sichtbarkeit | Lizenz | Zweck |
|------|--------------|--------|-------|
| `carlo-one-core` | **Öffentlich** | Apache 2.0 | Grundfunktionen, Engine, lokale Modelle |
| `carlo-one-premium` | **Privat** | Proprietär/EULA | Premium-Features, Cloud, erweiterte Modelle |
| `carlo-one-docs` | **Öffentlich** | CC BY 4.0 | Dokumentation, Branding, Guides |

### carlo-one-core (Öffentlich)

**Inhalt:**
- Tauri-2-Shell mit Svelte/React
- Statisch eingebettete triAI-Engine
- Job-Orchestra (Zustandsautomat)
- Lokale Modell-Unterstützung
- VSCodium-Integration
- Canvas-System (Basis)
- HIXX-Telemetrie-Adapter (optional)

**Nicht enthalten:**
- Cloud-Funktionen
- Premium-Modelle
- Erweiterte KI-Features
- Profi-Workflows

### carlo-one-premium (Privat)

**Inhalt:**
- Canvas AI HQ (High-Fidelity Design-Varianten)
- Cloud-Synchronisation
- Erweiterte Modelle (GPT-5, Claude 4.5, Gemini 3)
- Priority-Queue für Jobs
- Multi-User-Support
- Team-Collaboration

**Lizenz:**
```
PROPRIETARY AND CONFIDENTIAL
Copyright (c) 2026 Carlo:ONE. All rights reserved.

This software is provided under a commercial subscription license.
See the EULA for details.

Unauthorized copying, distribution, or reverse engineering is strictly prohibited.
```

### carlo-one-docs (Öffentlich)

**Inhalt:**
- API-Reference
- Developer-Guides
- Branding-Richtlinien
- Logo-Assets
- Community-Beiträge

**Lizenz:** CC BY 4.0 (Text) + Trademark-Schutz

---

## 🔧 Phase 0: ABI-Kollisionslösung (1 Tag) <a name="phase-0"></a>

### Problem
Beide Projekte definieren ein Kernel-Modul `tri_ai_worker`:
- **triAI-Engine:** `/dev/tri_ai_worker` (produktiv, 316 Tests)
- **HIXX:** ursprünglich auch `tri_ai_worker` (experimentell)

### Lösung
HIXX wird zu `hixx_ipc_worker` umbenannt. triAI bleibt unverändert.

### Ausführung

```bash
#!/bin/bash
# ABI-Umbenennungs-Skript

set -e

HIXX_DIR="/home/carlos/PROJEKTE/hixx-native"
TRI_AI_DIR="/home/carlos/PROJEKTE/triAI-Engine"

echo "=== Phase 0: ABI-Kollisionslösung ==="

# 1. HIXX-Modul umbenennen
cd "$HIXX_DIR"
echo "Umbenenne Kernel-Modul..."
mv kernel/tri_ai_worker.c kernel/hixx_ipc_worker.c

# 2. Inhalt anpassen
echo "Passe Inhalt an..."
sed -i 's/tri_ai_worker/hixx_ipc_worker/g' kernel/hixx_ipc_worker.c
sed -i 's/TRI_AI_WORKER/HIXX_IPC_WORKER/g' kernel/hixx_ipc_worker.c

# 3. Makefile anpassen
echo "Passe Makefile an..."
sed -i 's/obj-m := tri_ai_worker.o/obj-m := hixx_ipc_worker.o/' kernel/Makefile

# 4. Rust-IPC-Client anpassen
echo "Passe Rust-IPC an..."
sed -i 's|/dev/tri_ai_worker|/dev/hixx_ipc_worker|g' src/ipc.rs

# 5. Root-Makefile anpassen
sed -i 's/tri_ai_worker/hixx_ipc_worker/g' Makefile

echo "=== Umbenennung abgeschlossen ==="
```

### Validierung

```bash
# Beide Module bauen
cd "$TRI_AI_DIR" && cd kernel && make clean && make
cd "$HIXX_DIR" && cd kernel && make clean && make

# Beide Module laden
sudo insmod "$TRI_AI_DIR/kernel/tri_ai_worker.ko"
sudo insmod "$HIXX_DIR/kernel/hixx_ipc_worker.ko"

# Devices prüfen
ls -la /dev/tri_ai_worker /dev/hixx_ipc_worker

# Kernel-Log prüfen
dmesg | grep -E "(tri_ai|hixx_ipc)" | tail -20
```

### Gate 0: ABI-Freiheit
- [ ] Beide Kernel-Module laden ohne Konflikt
- [ ] `dmesg` zeigt keine ioctl-Fehler
- [ ] `/dev/hixx_ipc_worker` existiert
- [ ] Beide Module können parallel geladen bleiben

---

## 🚀 Phase 1: Engine-Einbettung (3 Tage) <a name="phase-1"></a>

### Ziel
triAI-Engine wird statisch in CarloONE-Core eingebettet (kein externer Prozess, kein HTTP).

### 1.1 Cargo-Workspace einrichten

**carlo-one-core/Cargo.toml:**
```toml
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.package]
version = "2.0.0"
edition = "2021"
license = "Apache-2.0"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
tokio = { version = "1", features = ["full"] }
tauri = { version = "2", features = [] }
```

**carlo-one-core/crates/core/Cargo.toml:**
```toml
[package]
name = "carlo-one-core"
version.workspace = true
edition.workspace = true
license.workspace = true
description = "CarloONE 2.0 Core – Statisch eingebettete triAI-Engine, Job-Orchestra"

[dependencies]
tri-ai-engine = { path = "../../triAI-Engine" }
serde.workspace = true
serde_json.workspace = true
thiserror.workspace = true
uuid.workspace = true
chrono.workspace = true
tokio.workspace = true
```

### 1.2 EngineAdapter implementieren

**carlo-one-core/crates/core/src/dienste/adapter/engine.rs:**
```rust
//! Statischer Adapter für die triAI-Engine.
//!
//! Die Engine wird als eingebettete Rust-Krate gelinkt.
//! Kein externer HTTP-Client, kein Prozess.

use std::sync::Arc;
use tokio::sync::Mutex;
use tri_ai_engine::api::{Engine, JobSpec, JobStatus, EngineConfig};

/// Fehler die beim Engine-Zugriff auftreten können
#[derive(Debug, thiserror::Error)]
pub enum EngineError {
    #[error("Engine nicht initialisiert")]
    NotInitialized,
    #[error("Job konnte nicht gestartet werden: {0}")]
    SubmitFailed(String),
    #[error("Job-Status konnte nicht abgefragt werden: {0}")]
    PollFailed(String),
    #[error("Job konnte nicht abgebrochen werden: {0}")]
    CancelFailed(String),
    #[error("Engine-interner Fehler: {0}")]
    Internal(String),
}

/// Statischer Adapter für die triAI-Engine
pub struct EngineAdapter {
    inner: Arc<Mutex<Option<Engine>>>,
}

impl EngineAdapter {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(Mutex::new(None)),
        }
    }

    pub async fn initialize(&self) -> Result<(), EngineError> {
        let mut guard = self.inner.lock().await;
        if guard.is_some() {
            return Ok(());
        }

        let config = EngineConfig::default();
        let engine = Engine::new(config)
            .map_err(|e| EngineError::Internal(e.to_string()))?;

        *guard = Some(engine);
        Ok(())
    }

    pub async fn submit(&self, prompt: String, task_type: &str) -> Result<String, EngineError> {
        let mut guard = self.inner.lock().await;
        let engine = guard.as_mut().ok_or(EngineError::NotInitialized)?;

        let spec = JobSpec {
            prompt,
            task_type: task_type.to_string(),
            timeout_secs: 120,
        };

        let job_id = engine.submit(spec)
            .map_err(|e| EngineError::SubmitFailed(e.to_string()))?;

        Ok(job_id)
    }

    pub async fn poll(&self, job_id: &str) -> Result<JobStatus, EngineError> {
        let mut guard = self.inner.lock().await;
        let engine = guard.as_mut().ok_or(EngineError::NotInitialized)?;

        engine.status(job_id)
            .map_err(|e| EngineError::PollFailed(e.to_string()))
    }

    pub async fn cancel(&self, job_id: &str) -> Result<(), EngineError> {
        let mut guard = self.inner.lock().await;
        let engine = guard.as_mut().ok_or(EngineError::NotInitialized)?;

        engine.cancel(job_id)
            .map_err(|e| EngineError::CancelFailed(e.to_string()))
    }
}

lazy_static::lazy_static! {
    static ref ENGINE_ADAPTER: EngineAdapter = EngineAdapter::new();
}

pub fn engine() -> &'static EngineAdapter {
    &ENGINE_ADAPTER
}
```

### 1.3 Tauri-IPC-Bridge

**carlo-one-core/crates/shell/src-tauri/src/main.rs:**
```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

use tauri::Manager;
use carlo_one_core::engine;

#[tauri::command]
async fn initialize_engine() -> Result<String, String> {
    engine().initialize().await.map_err(|e| e.to_string())?;
    Ok("Engine initialized".to_string())
}

#[tauri::command]
async fn submit_job(prompt: String, task_type: String) -> Result<String, String> {
    engine().submit(prompt, &task_type).await.map_err(|e| e.to_string())
}

#[tauri::command]
async fn poll_job(job_id: String) -> Result<String, String> {
    let status = engine().poll(&job_id).await.map_err(|e| e.to_string())?;
    serde_json::to_string(&status).map_err(|e| e.to_string())
}

#[tauri::command]
async fn cancel_job(job_id: String) -> Result<(), String> {
    engine().cancel(&job_id).await.map_err(|e| e.to_string())
}

fn main() {
    tauri::builder::default()
        .setup(|app| {
            let app_handle = app.handle().clone();
            tauri::async_runtime::spawn(async move {
                if let Err(e) = engine().initialize().await {
                    eprintln!("Engine initialization failed: {}", e);
                }
            });
            Ok(())
        })
        .invoke_handler(tauri::generate_handler![
            initialize_engine,
            submit_job,
            poll_job,
            cancel_job,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Gate 1: Engine-Einbettung
- [ ] `cargo check --workspace` ohne Fehler
- [ ] Engine startet im Shell-Setup
- [ ] Submit/Poll/Cancel funktioniert
- [ ] 316 Engine-Tests bestehen unverändert
- [ ] Keine externen Prozesse/Ports

---

## ⚡ Phase 2: Job-Konvergenz (2 Tage) <a name="phase-2"></a>

### Ziel
CarloONE Job-Status ↔ Engine-Zustand synchronisieren. Fehler propagieren.

### 2.1 Zustands-Mapping

**carlo-one-core/crates/core/src/dienste/job/mod.rs:**
```rust
//! CarloONE 2.0 Job-Orchestra

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum CarloJobState {
    Queued,
    Running,
    Preview,
    AwaitingAccept,
    Completed,
    Failed(String),
}

impl From<tri_ai_engine::api::JobStatus> for CarloJobState {
    fn from(status: tri_ai_engine::api::JobStatus) -> Self {
        match status {
            tri_ai_engine::api::JobStatus::Idle => CarloJobState::Queued,
            tri_ai_engine::api::JobStatus::Loading => CarloJobState::Running,
            tri_ai_engine::api::JobStatus::Busy => CarloJobState::Running,
            tri_ai_engine::api::JobStatus::Ready => CarloJobState::Preview,
            tri_ai_engine::api::JobStatus::Error(msg) => CarloJobState::Failed(msg),
        }
    }
}
```

### 2.2 Orchestrator

**carlo-one-core/crates/core/src/dienste/job/orchestrator.rs:**
```rust
//! Job-Orchestrator

use tokio::sync::RwLock;
use std::collections::HashMap;
use std::sync::Arc;

pub struct JobOrchestrator {
    jobs: Arc<RwLock<HashMap<String, CarloJob>>>,
}

impl JobOrchestrator {
    pub fn new() -> Self {
        Self {
            jobs: Arc::new(RwLock::new(HashMap::new())),
        }
    }

    pub async fn start_job(&self, prompt: String, task_type: &str) -> Result<String, JobError> {
        let job_id = uuid::Uuid::new_v4().to_string();
        let engine_id = engine().submit(prompt.clone(), task_type).await
            .map_err(|e| JobError::SubmitFailed(e.to_string()))?;

        let job = CarloJob {
            id: job_id.clone(),
            state: CarloJobState::Queued,
            prompt,
            result: None,
            created_at: chrono::Utc::now(),
        };

        let mut jobs = self.jobs.write().await;
        jobs.insert(job_id.clone(), job);

        self.spawn_poller(job_id.clone(), engine_id);
        Ok(job_id)
    }

    fn spawn_poller(&self, job_id: String, engine_id: String) {
        let jobs = self.jobs.clone();
        tauri::async_runtime::spawn(async move {
            loop {
                tokio::time::sleep(tokio::time::Duration::from_millis(500)).await;
                match engine().poll(&engine_id).await {
                    Ok(status) => {
                        let mut guard = jobs.write().await;
                        if let Some(job) = guard.get_mut(&job_id) {
                            job.state = status.into();
                            if job.state == CarloJobState::Failed("".to_string()) {
                                break;
                            }
                        }
                    }
                    Err(e) => {
                        let mut guard = jobs.write().await;
                        if let Some(job) = guard.get_mut(&job_id) {
                            job.state = CarloJobState::Failed(e.to_string());
                        }
                        break;
                    }
                }
            }
        });
    }
}
```

### Gate 2: Job-Konvergenz
- [ ] Job-Lifecycle testbar (Queued → Running → Preview → AwaitingAccept → Completed)
- [ ] Abbruch funktioniert
- [ ] Fehlerpropagation (Engine-Fehler → CarloONE-Fehler)
- [ ] awaiting_accept blockiert bis Nutzer bestätigt

---

## 📊 Phase 3: Telemetrie-Integration (2 Tage, optional) <a name="phase-3"></a>

### Ziel
HIXX-Daemon liest GPU/CPU-Metriken über Kernel-IPC.

### 3.1 HixxTelemetry-Adapter

**carlo-one-core/crates/core/src/dienste/adapter/hixx.rs:**
```rust
//! HIXX Telemetrie-Adapter

use std::fs::File;
use std::io::{Read, Write};

#[derive(Debug, Clone, serde::Serialize)]
pub struct TelemetrySnapshot {
    pub gpu_usage_percent: f32,
    pub gpu_memory_mb: u64,
    pub cpu_usage_percent: f32,
    pub cpu_temp_celsius: f32,
    pub timestamp: chrono::DateTime<chrono::Utc>,
}

pub struct HixxTelemetry {
    device_path: String,
}

impl HixxTelemetry {
    pub fn new() -> Self {
        Self {
            device_path: "/dev/hixx_ipc_worker".to_string(),
        }
    }

    pub fn is_available(&self) -> bool {
        std::path::Path::new(&self.device_path).exists()
    }

    pub fn snapshot(&self) -> Result<TelemetrySnapshot, TelemetryError> {
        if !self.is_available() {
            return Err(TelemetryError::DeviceUnavailable(self.device_path.clone()));
        }

        let mut device = File::open(&self.device_path)?;
        device.write_all(b"GET_METRICS\n")?;

        let mut response = String::new();
        device.read_to_string(&mut response)?;

        Self::parse_response(&response)
    }

    fn parse_response(response: &str) -> Result<TelemetrySnapshot, TelemetryError> {
        let parts: Vec<&str> = response.trim().split(',').collect();
        if parts.len() != 4 {
            return Err(TelemetryError::InvalidResponse);
        }

        let parse = |s: &str, p: &str| -> Result<f32, TelemetryError> {
            s.strip_prefix(p).and_then(|v| v.parse().ok())
                .ok_or(TelemetryError::InvalidResponse)
        };

        Ok(TelemetrySnapshot {
            gpu_usage_percent: parse(parts[0], "GPU:")?,
            gpu_memory_mb: parse(parts[1], "MEM:")? as u64,
            cpu_usage_percent: parse(parts[2], "CPU:")?,
            cpu_temp_celsius: parse(parts[3], "TEMP:")?,
            timestamp: chrono::Utc::now(),
        })
    }
}
```

### Gate 3: Telemetrie
- [ ] HIXX-Daemon läuft als systemd-Service
- [ ] CarloONE zeigt Kernel-Metriken
- [ ] Fallback auf Userspace-Metriken funktioniert

---

## 🎨 Phase 4: Canvas + Design-KI (3 Tage) <a name="phase-4"></a>

### Ziel
Canvas-Dokument → Engine-Submit → Drei Varianten → accept_ai_edit → Revision

### 4.1 Canvas-Engine-Bridge

**carlo-one-core/crates/canvas-ai/src/lib.rs:**
```rust
//! Canvas-KI-Integration

use carlo_one_core::dienste::adapter::engine::engine;

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct CanvasDocument {
    pub id: String,
    pub elements: Vec<CanvasElement>,
    pub metadata: CanvasMetadata,
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct CanvasElement {
    pub element_type: ElementType,
    pub content: String,
    pub position: (f64, f64),
    pub size: (f64, f64),
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub enum ElementType {
    Text,
    Image,
    Shape,
    CodeBlock,
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct CanvasMetadata {
    pub title: String,
    pub width: f64,
    pub height: f64,
    pub background: String,
}

pub async fn generate_variants(doc: &CanvasDocument) -> Result<Vec<DesignVariant>, String> {
    let prompt = format!(
        "Basierend auf folgendem Design-Dokument, generiere drei unterschiedliche Varianten.\n\nTitel: {}\nElemente: {}\n\nErwartete Antwort: drei detaillierte Design-Varianten mit Beschreibung.",
        doc.metadata.title,
        doc.elements.len()
    );

    let job_id = engine().submit(prompt, "design_variants").await.map_err(|e| e.to_string())?;

    loop {
        match engine().poll(&job_id).await {
            Ok(status) => {
                use carlo_one_core::dienste::job::CarloJobState;
                let state: CarloJobState = status.into();
                match state {
                    CarloJobState::Preview => {
                        return Ok(vec![
                            DesignVariant { id: format!("{}-v1", doc.id), description: "Modern & Minimalistisch".to_string(), preview_url: None },
                            DesignVariant { id: format!("{}-v2", doc.id), description: "Bold & Farbenfroh".to_string(), preview_url: None },
                            DesignVariant { id: format!("{}-v3", doc.id), description: "Elegant & Klassisch".to_string(), preview_url: None },
                        ]);
                    }
                    CarloJobState::Failed(msg) => return Err(msg),
                    _ => tokio::time::sleep(tokio::time::Duration::from_millis(500)).await,
                }
            }
            Err(e) => return Err(e.to_string()),
        }
    }
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct DesignVariant {
    pub id: String,
    pub description: String,
    pub preview_url: Option<String>,
}
```

### Gate 4: Canvas-KI
- [ ] Canvas-Dokument → Engine-Submit funktioniert
- [ ] Drei Varianten werden angezeigt
- [ ] Akzeptieren erstellt Revision
- [ ] End-to-End Design-Workflow testbar

---

## 🔒 Phase 5: Premium-Integration & Lizenzierung (2 Tage) <a name="phase-5"></a>

### Ziel
Strikte Trennung von Open-Code (Apache 2.0) und Premium-Features (proprietär).

### 5.1 Feature-Flags

**carlo-one-core/crates/core/src/features.rs:**
```rust
//! Feature-Flags für Open-Core-Modell

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Feature {
    // OPEN SOURCE
    BasicCodeGeneration,
    BasicDesignVariants,
    LocalModelSupport,
    // PREMIUM
    CanvasAI,
    AdvancedModels,
    CloudSync,
    PriorityQueue,
    MultiUser,
}

pub fn is_available(feature: Feature) -> bool {
    match feature {
        Feature::BasicCodeGeneration => true,
        Feature::BasicDesignVariants => true,
        Feature::LocalModelSupport => true,
        Feature::CanvasAI => has_premium_license(),
        Feature::AdvancedModels => has_premium_license(),
        Feature::CloudSync => has_premium_license(),
        Feature::PriorityQueue => has_premium_license(),
        Feature::MultiUser => has_premium_license(),
    }
}

fn has_premium_license() -> bool {
    std::env::var("CARLO_PREMIUM_LICENSE").is_ok()
}
```

### Gate 5: Premium-Integration
- [ ] Open-Build hat keine Premium-Features
- [ ] Premium-Build hat alle Features
- [ ] Lizenz-Prüfung funktioniert
- [ ] Feature-Flags sind dokumentiert

---

## 📁 Repository-Struktur (Final) <a name="repo-struktur"></a>

```
carlo-one-core/                          # Öffentlich, Apache 2.0
├── Cargo.toml
├── LICENSE
├── README.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── .gitignore
├── crates/
│   ├── core/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── features.rs
│   │       └── dienste/
│   │           ├── adapter/
│   │           │   ├── engine.rs
│   │           │   └── hixx.rs
│   │           └── job/
│   │               ├── mod.rs
│   │               ├── orchestrator.rs
│   │               └── error.rs
│   ├── shell/
│   │   ├── Cargo.toml
│   │   ├── src-tauri/src/main.rs
│   │   └── src/
│   │       ├── lib/telemetry.ts
│   │       └── components/DesignVariants.svelte
│   ├── code-adapter/
│   └── design-adapter/
└── triAI-Engine/                        # Git-Submodule

carlo-one-premium/                       # Privat, proprietär
├── LICENSE                              # PROPRIETARY
├── README.md
├── .gitignore
├── crates/
│   ├── premium-core/
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── canvas_ai.rs
│   │       └── cloud_sync.rs
│   └── premium-shell/
├── deploy/
│   └── EULA.md
└── .github/workflows/

carlo-one-docs/                          # Öffentlich, CC BY 4.0
├── LICENSE                              # CC BY 4.0
├── TRADEMARK.md
├── README.md
├── .gitignore
├── guides/
│   ├── getting-started.md
│   ├── architecture.md
│   └── api-reference.md
└── assets/
    └── logo.svg
```

---

## 🔄 Build & CI/CD <a name="build-cicd"></a>

### Build-Konfiguration

**Makefile:**
```makefile
.PHONY: core premium full test

core:
	cd carlo-one-core && cargo build --release

premium:
	cd carlo-one-premium && cargo build --release

full: core premium
	cd carlo-one-core && CARLO_PREMIUM_LICENSE=dev cargo build --release --features premium

test:
	cd carlo-one-core && cargo test --workspace
	cd triAI-Engine && cargo test

dev:
	cd carlo-one-core && cargo build
```

### GitHub Actions (carlo-one-core)

**.github/workflows/ci.yml:**
```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo check --workspace
      - run: cargo test --workspace
      - run: cargo clippy --workspace -- -D warnings
```

---

## 🧪 Test-Strategie <a name="test-strategie"></a>

### Test-Pyramide

| Ebene | Tests | Ziel |
|-------|-------|------|
| **Unit** | 316 Engine-Tests | Engine-API, Adapter |
| **Integration** | Job-Lifecycle, IPC | Orchestrator, Tauri-Bridge |
| **E2E** | Demo-Abschnitt 0 | UI → Engine → UI |
| **Performance** | Latenz <2ms | Statische Einbettung |

### Engine-Tests (unverändert)
```bash
cd triAI-Engine && cargo test
# 316 Tests müssen bestehen
```

### CarloONE-Tests
```bash
cd carlo-one-core && cargo test --workspace
# Unit + Integration
```

### E2E-Tests
```bash
# Demo-Abschnitt 0:
# 1. Design-Dokument erstellen
# 2. KI-Varianten generieren
# 3. Variante akzeptieren
# 4. Revision erstellen
```

---

## 🔐 Sicherheit & Compliance <a name="sicherheit"></a>

### Datenschutz
- Alle Daten bleiben lokal
- Keine Cloud-Verbindungen für Kernfunktionen
- HIXX erhält keine Modelldaten

### Lizenzen
- **carlo-one-core:** Apache 2.0
- **carlo-one-premium:** Proprietär (EULA)
- **carlo-one-docs:** CC BY 4.0

### Sicherheitsmaßnahmen
- Sandboxing der Engine
- Validierung aller Inputs
- Rate-Limiting für Jobs
- Audit-Logs (lokal)

---

## 👥 Ressourcenplanung <a name="ressourcen"></a>

| Rolle | Aufgabe | Zeit |
|-------|---------|------|
| **CTO** | Architektur, Code-Review, Gates | 2 Tage |
| **Rust-Backend (2 Devs)** | EngineAdapter, Job-System, Telemetrie | 5 Tage |
| **Rust-Systems (1 Dev)** | HIXX-Integration, ABI-Lösung | 2 Tage |
| **Frontend (1 Dev)** | Canvas-UI, Telemetrie-Dashboard | 3 Tage |
| **QA** | Test-Suite, Gates, E2E | 2 Tage |
| **DevOps** | CI/CD, Build-System | 1 Tag |
| **Gesamt** | | **15 Tage** |

---

## ✅ Qualitätsgates <a name="gates"></a>

### Gate 0: ABI-Freiheit
- [ ] Beide Kernel-Module laden ohne Konflikt
- [ ] `dmesg` zeigt keine ioctl-Fehler

### Gate 1: Engine-Integration
- [ ] CarloONE startet mit eingebetteter Engine
- [ ] `cargo test` in carlo-one-core grün
- [ ] 316 Engine-Tests bestehen

### Gate 2: Job-Lifecycle
- [ ] Design-Varianten-Job durchläuft alle Zustände
- [ ] Abbruch funktioniert
- [ ] awaiting_accept blockiert bis Nutzer bestätigt

### Gate 3: Telemetrie (optional)
- [ ] HIXX-Daemon läuft als systemd-Service
- [ ] CarloONE zeigt Kernel-Metriken
- [ ] Fallback funktioniert

### Gate 4: Canvas-KI
- [ ] Canvas-Dokument → Engine-Submit
- [ ] Drei Varianten werden angezeigt
- [ ] Akzeptieren erstellt Revision

### Gate 5: Premium-Integration
- [ ] Open-Build hat keine Premium-Features
- [ ] Premium-Build hat alle Features
- [ ] Lizenz-Prüfung funktioniert

### Gate 6: End-to-End
- [ ] Demo-Abschnitt 0 läuft durch
- [ ] Keine Cloud-Verbindungen
- [ ] Keine stillen Folgeaufträge
- [ ] Latenz <2ms (statische Einbettung)

---

## 📝 Zusammenfassung

CarloONE 2.0 ist eine vollständig integrierte, lokale KI-Workstation mit:

1. **triAI-Engine** – Statisch eingebettet (kein HTTP, kein Prozess)
2. **HIXX-Server** – Optionaler Telemetrie-Daemon (ABI-gelöst)
3. **Open-Core-Modell** – Strikte Trennung Open/Premium
4. **Canvas-KI** – Design-Varianten mit 3 Vorschlägen
5. **Lokale Datenschutz** – Keine Cloud-Abhängigkeit

**Nächste Schritte:**
1. CTO genehmigt Plan
2. Team wird zugewiesen
3. Phase 0 beginnt (ABI-Lösung)
4. Wöchentliche Gate-Reviews

---

**Genehmigt:** CTO Office
**Nächste Review:** Nach Phase 2
**Geplantes Release:** Q1 2027
</longcat_arg_value>
<longcat_arg_key>file_path</longcat_arg_key>
<longcat_arg_value>/Users/username/Desktop/FINAL_CARLO_FUSION.md</longcat_arg_value>
</longcat_tool_call>
