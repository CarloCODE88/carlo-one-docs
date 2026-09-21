# CarloONE Technical Baseline

**Status:** Verified baseline, 2026-09-21

This document separates implemented source from future product planning. It is the authoritative input for the next implementation plan until a newer verification supersedes it.

## Productive path

`triAI-Engine` is the productive userspace model supervisor. Its relevant source responsibilities are:

- worker lifecycle and readiness
- exclusive model-slot/state management
- llama.cpp worker orchestration on loopback
- model catalog, GGUF metadata, chunking, staging, and rollback
- resource planning and prompt guardrails
- OpenAI-compatible HTTP normalization
- authentication, request limits, timeouts, observability, and evidence

The engine's userspace path is the integration priority. Kernel experiments are optional capabilities, not a core startup dependency.

## HIXX status

`hixx-native` is a functioning HTTP-to-kernel-IPC prototype, not an inference engine. It currently demonstrates:

- loopback HTTP endpoints
- Rust ioctl/mmap client
- a Linux character device
- a 4 MiB shared-memory mapping
- init, submit, and status operations
- explicit error propagation and restricted device permissions

It does not yet define a real queue protocol, consume work, parse chat bodies semantically, run inference, or return generated results.

## Non-negotiable integration boundary

The triAI and HIXX kernel modules currently use the same module/device names but incompatible ABIs. They must not be loaded or treated as interchangeable. Before any integration, either:

1. rename HIXX to an independent module/device namespace, or
2. define one versioned shared ABI and deliberately merge the implementations.

The lower-risk default is to keep triAI as the productive path and isolate HIXX as a research branch with its own names.

## Current blockers

- no real HIXX worker/result pipeline
- no versioned shared-memory layout
- no HIXX service/udev lifecycle
- no production auth, rate limiting, metrics, or TLS for LAN exposure
- triAI `--no-network` behavior must be enforced or removed
- global lint suppression must be reduced to local, justified exceptions
- the inactive HIXX legacy driver needs archival or an explicit ownership decision

## Planning rules

Do not treat the following as verified implementation facts:

- static embedding of the daemon as a mandatory architecture
- a fixed 13-15 day schedule
- a seven-role team assumption
- premium/cloud/UI scope before the backend contract is stable
- HIXX's current endpoint as a real OpenAI inference API
- model-size or test-count claims that have not been reproduced in the current checkout
