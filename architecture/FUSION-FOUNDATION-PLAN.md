# CarloONE Fusion Foundation Plan

Status: draft for CTO review and architecture approval
Date: 2026-09-21

## 1. Purpose and scope

This plan consolidates the useful technical source material from the current CarloONE and HIXX workstreams, filters it against what is actually verified on disk and in runtime, and defines the minimum foundation required before product work, premium layers, or UI integration are expanded.

The goal is not to build the final product in one pass. The goal is to establish a correct backend architecture and a clear set of decision gates so that all later work is anchored in something that can actually be built, tested, and supported.

This document intentionally does not treat the legacy CTO fusion plan as source truth. It is a strategic intent document, but it must be checked against the verified baseline below.

## 2. Verified source inputs

The following source documents are treated as the technical baseline for this plan:

- `architecture/TECHNICAL-BASELINE.md`
- `architecture/TRIAI-ENGINE-ARCHITECTURE.md`
- `operations/HIXX-SERVER-ARCHITECTURE-AND-OPERATIONS.md`
- `operations/TRI-HIXX-DATA-TREE.md`
- `guides/CTO-FUSION-PLAN.md` (legacy strategic intent only; filtered and not accepted as fact without validation)

## 3. What is verified and useful

### 3.1 triAI-Engine is the primary productive backend path

The triAI architecture documents and source tree show a clear userspace orchestration model:

- worker lifecycle and readiness management
- exclusive model slot and state transitions
- model catalog, manifest, GGUF handling, chunking, staging, and rollback
- resources, prompt guardrails, attachments, tool registry, observability, and evidence
- HTTP-normalized API and request handling
- explicit state boundaries for local inference orchestration

This is the strongest and most coherent source direction currently present. It is the correct default backend path.

### 3.2 HIXX is a functional IPC prototype, not a real inference engine

The HIXX source and operations documentation confirm that it is currently a working HTTP-to-kernel-IPC prototype:

- loopback-only HTTP endpoints
- Rust ioctl/mmap client
- Linux character device interaction
- shared-memory mapping with explicit validation
- init/submit/status semantics
- error propagation to HTTP responses

It does not yet provide a real work queue, worker result path, semantic prompt execution, or inference output. The current documentation itself explicitly states this limitation. HIXX is therefore best treated as an isolated technical experiment or sidecar component until a formal ABI and operational boundary is decided.

### 3.3 The structural issue is not only technical but architectural

The most important finding across the checked source documents is the shared naming collision:

- triAI uses a module/device naming pattern based on `tri_ai_worker`
- HIXX also uses a module/dev naming pattern based on `tri_ai_worker`
- the two implementations are not the same ABI
- they therefore cannot coexist without collision or silent runtime confusion

This is the first decision gate that must be resolved before any deeper integration is attempted.

### 3.4 The legacy strategic plan remains useful only as intent, not as execution truth

The legacy CTO fusion plan is useful for expressing the intended product direction, but it is not a reliable implementation baseline because it assumes:

- static embedding without a verified target machine or runtime constraint check
- a single converged backend stack before the boundary models are confirmed
- premium scope before the open backend is stable
- a fixed timeline and staffing model that is not grounded in actual repository or environment state
- a level of integration maturity that the source documents do not support

The new plan below therefore separates intent from verified fact.

## 4. Architecture governing principles

1. The triAI userspace backend is the operating model to preserve.
2. HIXX remains an isolated IPC or sidecar prototype unless an explicit ABI decision is made.
3. The platform must not commit to premium or UI layers before a stable backend boundary exists.
4. Kernel access remains optional and must not be a required startup dependency.
5. LAN or external exposure is prohibited until auth, rate limits, logging, and TLS/or a proper reverse proxy layer are defined.
6. The codebase must not rely on global warning suppression or broad lint exemptions.
7. Every integration decision must be traceable to a source document and a clear acceptance criterion.

## 5. Recommended architecture model

### 5.1 Productive track: triAI backend

The triAI backend should be treated as the primary productive implementation path. It should remain the root path for:

- model lifecycle and worker supervision
- GGUF management and staging
- prompt guardrail enforcement
- evidence, observability, and audits
- request normalization and structured runtime states
- local inference orchestration and model slot handling

The problem is not that triAI is complete; the problem is that it is the coherent and real path already in source. The plan should therefore build upward from it without pretending it is already a finished product.

### 5.2 Experimental track: HIXX backend

HIXX should remain clearly labeled as a research and transport-layer prototype until the following are explicitly resolved:

- device and module naming
- ABI compatibility and versioning
- queue protocol and ownership model
- result path and worker semantics
- lifecycle management and restart behavior
- operational separation from the triAI runtime

This track should be allowed to continue only under clearly documented boundaries. It should never be treated as a production inference path by default.

### 5.3 Frontend and premium positioning

Tauri, canvas, premium modules, and cloud workflows should be planned only after all of the following are true:

- backend contract is defined
- target machine or execution profile is decided
- the runtime model is stable
- ownership and deployment boundaries are clear

This is the key strategic correction to the older plan.

## 6. Required decision gates before deeper integration

### Gate 1: target machine and runtime boundary

The project must explicitly decide:

- local workstation backend only
- dedicated GPU machine with heavier model orchestration
- hybrid local + remote model execution

Without this decision, architecture assumptions will continue to drift.

### Gate 2: kernel integration boundary

The project must choose one of two categories:

- isolate HIXX as its own module/device namespace; or
- define one shared ABI for both and merge them under a controlled versioning regime

Recommended: isolate HIXX as a separate and clearly labeled prototype until the architecture matures.

### Gate 3: product / premium split

Open-core and premium must be separated structurally and legally before broad product work begins. The premium repository should not be treated as part of the same execution baseline as the public core until there is a stable contract and a real licensing decision in place.

### Gate 4: exposure and operations boundary

Before LAN or remote exposure, the following must be required:

- authentication model
- rate limiting and request caps
- logs and audit trails
- TLS or reverse-proxy placement
- service lifecycle and restart policy
- separate operational ownership for backend and UI

## 7. Execution plan

### Phase 0 — Architecture and environment decision

Objective

Establish the actual architecture boundary and target machine before product work expands.

Files and repos

- `carlo-one-docs/architecture/TECHNICAL-BASELINE.md`
- `carlo-one-docs/architecture/TRIAI-ENGINE-ARCHITECTURE.md`
- `carlo-one-docs/operations/HIXX-SERVER-ARCHITECTURE-AND-OPERATIONS.md`
- the team repository `carlo-one-core`, which now carries both imports: `triAI-Engine/` and `hixx-native/` (source of truth for HIXX is `carlo-one-core/hixx-native` @ abefbc1, per `BACKEND-SOURCE-INVENTORY.md`; the owner's local `/home/carlos/PROJEKTE/...` folders are superseded as reference locations)

Prerequisites

- none beyond access to the verified source set

Implementation tasks

- confirm target machine and execution model
- confirm whether HIXX remains isolated or gets merged under a shared ABI
- define the no-go boundary for static embedding and premium expansion
- record final architecture ownership

Verification criteria

- architecture decision memo approved by owner/CTO
- no unresolved naming or device collision remains
- no product layer starts before backend authority is decided

Risks

- parallel work continuing under incorrect assumptions
- backend drift due to unsupported product scope

Non-goals

- UI work
- premium feature delivery
- end-user deployment

### Phase 1 — triAI backend hardening and stabilization

Objective

Convert the triAI userspace backend into the explicit, verified product backend for the next delivery window.

Files and repos

- `carlo-one-core/triAI-Engine/...`
- `carlo-one-core/README` and project documentation
- `carlo-one-docs/architecture/TRIAI-ENGINE-ARCHITECTURE.md`

Prerequisites

- Phase 0 decision and target environment clarified

Implementation tasks

- remove or enforce `--no-network` behavior properly
- remove global warning suppression and reduce lint exemptions to justified local cases
- define a canonical health and readiness state model
- establish a clean worker and slot lifecycle contract
- tighten request validation and operational safety
- confirm evidence, observability, and configuration ownership

Verification criteria

- `cargo test` passes in the current triAI checkout
- `cargo clippy -- -D warnings` is meaningful and passes after justified exceptions are narrowed
- worker lifecycle and failure modes are explicit
- no hidden runtime dependency on a privileged kernel path is assumed

Risks

- overestimating triAI maturity
- underestimating the need for explicit safety and deployment rules

Non-goals

- premium features
- canvas or design-specific logic
- external cloud sync

### Phase 2 — HIXX boundary and protocol definition

Objective

Keep HIXX from drifting into an accidental production dependency while preserving the useful IPC research.

Files and repos

- `carlo-one-core/hixx-native/` @ abefbc1 — the team repository is the source of truth (covers `hixx-native/src/*` and `hixx-native/kernel/*`); line numbers cited below refer to this commit
- `carlo-one-docs/operations/HIXX-SERVER-ARCHITECTURE-AND-OPERATIONS.md` (behavioral reference only; its `/home/carlos/PROJEKTE/hixx-native` path references are superseded by the team repo)

Prerequisites

- Phase 0 boundary and naming decision

Implementation tasks

- rename HIXX module/device to a unique namespace if kept (candidate: `hixx_ipc_worker` / `/dev/hixx_ipc_worker`). Verified rename scope @ abefbc1: the `tri_ai` string occurs in exactly 4 files — `src/ipc.rs` (DEVICE_PATH), `kernel/tri_ai_worker.c` (DEVICE_NAME), `kernel/Makefile` (obj-m target), root `Makefile` (rmmod/insmod/chgrp/chmod/status targets); the ABI values (magic 0x48, commands 0x00004800/0x00004801/0x80044802) are unaffected by a pure name rename.
- extend the Rust ABI test (`src/ipc.rs`, `ioctl_encodings_match_linux_layout` — exists and pins the three hex values, but only against the Rust-side derivation) to also cover the C-side encoding: the values are duplicated, not shared — Rust derives them via a const fn from magic 0x48 (`src/ipc.rs`), C via `_IO`/`_IOR` macros from the same magic (`kernel/tri_ai_worker.c`); there is no shared header. Establish a single source of truth (checked-in ABI header or a golden-value test covering magic, direction, and size encoding) as part of this task.
- define a versioned shared-memory contract and explicit queue ownership. Verified constraints @ abefbc1: mmap is fixed to offset 0 and exactly 4 MiB (`kernel/tri_ai_worker.c` mmap check, `src/ipc.rs` RING_BUFFER_SIZE); the mapping exposes no head/tail and no slots — SUBMIT only increments a kernel-side counter and the 4 MiB buffer stays zeroed (`kernel/tri_ai_worker.c`); no struct in the contract carries a version or reserved field, so future ring/queue layouts cannot evolve in place and must be defined fresh.
- add a result path and job lifecycle state
- separate experimental mode from real production mode. Verified today: POST `/v1/chat/completions` never reads the request body and returns a canned 202 with a fabricated `chat.completion` object (`src/api.rs`); HTTP 503 is mapped only for IPC submit/status ioctl failures (`src/api.rs`, chat and status handlers); `/health` reports ok without probing the kernel (`src/api.rs`). An honest 202 must return an accepted-task envelope (job id plus status pointer), not a fake completion; the request structs in `src/ipc/structs.rs` are currently unused by the HTTP path.
- archive the legacy alternative driver. Verified disposition @ abefbc1: `kernel/hixx_worker.c` is not referenced by any Makefile, speaks a different ABI (magic 'h'/0x68 with LOAD_MODEL/RUN_INFERENCE/GET_METRICS/RINGPTR ioctls), has no `.mmap` op, uses the pre-6.4 `class_create(THIS_MODULE, ...)` signature (cannot build against the same kernel headers as the operative module), and calls `virt_to_phys()` on vmalloc memory. Recommended: move it to `kernel/attic/` with a note marking it design reference for the versioned queue contract — keep as reference, do not maintain as a module, do not delete.

Verification criteria

- the kernel and userspace contract are explicit and tested: `grep -rn "tri_ai" carlo-one-core/hixx-native/` returns no matches (rename complete), and the extended Rust ABI test passes against the shared/mapped hex values (magic 0x48, 0x00004800, 0x00004801, 0x80044802)
- no collision exists with triAI's existing module and device path (`triAI-Engine/kernel/tri_ai_worker.c` registers the same module/device name today)
- status, queue depth, worker state, and error propagation are deterministic
- machine split for this phase: userspace code work (ABI test extension, `cargo test` on the hixx crate in dev profile) runs on the team box; kernel build (`kernel/Makefile` requires clang/ld.lld and kernel headers), `make load` (root, insmod), the 4 MiB mmap smoke, and `scripts/hixx_tune_system.sh` (allocates 2048 hugepages ≈ 4 GiB — exceeds team-box RAM) run only on the owner's GPU machine. Note the root `make rust`/`make all` targets do `cargo build --release` with LTO — release/LTO builds are owner-machine territory; team-box verification is `cargo test`, not `make rust`.

Risks

- if not controlled, HIXX can silently poison the triAI runtime and create ABI ambiguity

Non-goals

- inference or model orchestration as a default path
- product exposure before the contract is stable

### Phase 3 — deployment and operations hardening

Objective

Create a safe service and runtime model before any remote or LAN exposure.

Files and repos

- systemd or launcher definitions in the local project sources
- `carlo-one-docs/operations/TRI-HIXX-DATA-TREE.md`

Prerequisites

- stable backend contract

Implementation tasks

- define separate service ownership for triAI and HIXX
- ensure proper logs, restart policy, and runtime directories
- establish data retention and model artifact boundaries
- enforce explicit permissions and non-root access where possible
- apply security defaults before any external binding

Verification criteria

- service starts cleanly and logs deterministically
- model files and runtime logs are separated from source and ephemeral state
- no secrets are stored in example config or runtime output

Risks

- production service drift
- operational confusion between source state and runtime state

Non-goals

- product user-facing deployment work
- premium onboarding

### Phase 4 — UI and product layer activation

Objective

Allow Tauri or other frontend work only after backend paths are stable and bounded.

Prerequisites

- Phase 0–3 decisions complete

Implementation tasks

- create the UI/backend contract
- connect the UI to a stable backend API surface
- keep premium and product features behind explicit capability checks

Verification criteria

- UI calls are routed to a known backend contract
- no UI layer introduces real runtime assumptions about unavailable kernel or model services

Risks

- UI development creates hidden dependency chains
- teams build against assumptions rather than contracts

Non-goals

- premature premium expansion
- canvas or cloud features without a stable backend

### Phase 5 — premium gating and product expansion

Objective

Keep premium logic structurally separate and only enabled when the public backend has real stability and clear ownership.

Prerequisites

- product layer and backend contract stable
- legal and licensing decisions complete

Implementation tasks

- define premium capability boundaries
- keep premium in a separate repo or strict internal layer
- limit premium features to explicit capability checks

Verification criteria

- open-core functionality works independently
- premium entry points do not break core execution

Risks

- feature leakage between open and premium paths
- legal or licensing mismatch

Non-goals

- premium work before the stable backend exists

## 8. Delivery order recommendation

The practical order should be:

1. decide target machine and backend ownership
2. fix naming/ABI boundary between triAI and HIXX
3. stabilize triAI backend and enforced quality gates
4. isolate and define HIXX as a sidecar or research layer
5. fix runtime hardening and service lifecycle
6. then begin UI, product, and premium work

This is the order that best matches the actual source evidence and avoids building large product claims on unstable core assumptions.

## 9. Risks and blockers

- unresolved triAI/HIXX kernel collision
- overselling HIXX as an inference engine
- prematurely building premium functionality
- UI work depending on an unfinalized backend contract
- no explicit target-host decision
- global warning suppression and broad lint exemptions masking real quality issues
- lack of auth, rate limiting, and TLS before any external exposure

## 10. Non-goals

- building a full static-embedded monolith by default
- treating premium as part of the open-core baseline
- assuming the old CTO plan is execution truth
- claiming the HIXX prototype is complete or production-ready
- exposing the runtime before the service boundary is hardened

## 11. CTO decision list

The following items require explicit approval before implementation proceeds beyond the foundation phase:

1. Target machine and runtime model
2. Whether HIXX remains isolated or requires a shared ABI
3. Whether static embedding remains a future option or is rejected for now
4. Whether UI integration begins before the backend contract is finalized
5. Whether premium delivery is gated behind a stable open-core backend
6. Whether any external or LAN exposure may begin before auth, rate limits, and lifecycle protections are completed

## 12. Recommended next action

The next action should be a short CTO decision memo that approves:

- triAI as the dominant productive backend
- HIXX as a controlled prototype or sidecar until the ABI model is agreed
- backend and machine boundary decisions before any product layer is expanded
- the phased delivery order above

This gives the project a realistic foundation without pretending that the current state is more mature than the source documents support.
