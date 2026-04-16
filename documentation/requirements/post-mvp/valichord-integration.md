# ValiChord Integration: Blind Peer Validation for the NDO Lifecycle

**Status**: Post-MVP Design Document
**Created**: 2026-04-16
**Authors**: ValiChord project (topeuph-ai), in dialogue with Nondominium
**Relates to**: `ndo_prima_materia.md` (§5.1 LifecycleStage, §5.3 Prototype→Stable transition, REQ-NDO-LC-02/03), `flowsta-integration.md`, `unyt-integration.md`

---

## Table of Contents

1. [Purpose and scope](#1-purpose-and-scope)
2. [The gap ValiChord fills](#2-the-gap-valichord-fills)
3. [What ValiChord provides](#3-what-valichord-provides)
4. [Integration architecture](#4-integration-architecture)
5. [Integration path — five design decisions](#5-integration-path--five-design-decisions)
6. [Requirements traceability](#6-requirements-traceability)
7. [Current state](#7-current-state)

---

## 1. Purpose and scope

This document is the **dedicated integration stub** for wiring **ValiChord** — a Holochain blind
commit-reveal peer validation protocol — into the Nondominium lifecycle.

ValiChord solves a specific problem: how can an independent party provide cryptographically
ungameable confirmation that a resource does what it claims to do, with no possibility of
adaptive reveals or post-hoc verdict changes? It answers the reproducibility question —
*can an independent party arrive at the same result as the originator?* — not the correctness
question. A resource can be reproducible and scientifically wrong. ValiChord only attests to
the former.

**Repo**: https://github.com/topeuph-ai/ValiChord  
**Integration docs** (ValiChord side): `nondominium_integration/` in the ValiChord repo

---

## 2. The gap ValiChord fills

### 2.1 The commented-out call

In `zome_resource`'s `create_economic_resource()`, a cross-zome call to
`zome_gouvernance::validate_new_resource` is currently commented out:

```rust
// TEMPORARILY COMMENTED OUT - Call governance zome to initiate resource validation
// This implements REQ-GOV-02: Resource Validation
// TODO: Re-enable once cross-zome call issues are resolved
```

When uncommented, this initiates validation for a newly created `EconomicResource` in
`ResourceState::PendingValidation`. ValiChord is the validation protocol that fires here.

### 2.2 The Prototype → Stable transition

`ndo_prima_materia.md` §5.3 defines the `Prototype → Stable` lifecycle transition as requiring
**multi-agent peer validation (configurable N-of-M)**:

```
| Prototype → Stable | Multi-agent peer validation (configurable N-of-M) |
```

This transition also generates a triggering `EconomicEvent` (REQ-NDO-LC-03), and must be
driven by the governance zome acting as operator (REQ-NDO-LC-02). ValiChord's `HarmonyRecord`
is that event — a cryptographically locked majority consensus result that the governance zome
can use to authorise the transition.

### 2.3 Two layers, two integration hooks

Resources now have a two-level structure: `NondominiumIdentity` (permanent Layer 0 anchor) +
`EconomicResource` (operational instance). ValiChord drives both:

| Layer | Entry | ValiChord drives |
|---|---|---|
| Layer 0 | `NondominiumIdentity.lifecycle_stage` | `Prototype → Stable` via `update_lifecycle_stage()` + `NdoToTransitionEvent` link pointing to the `HarmonyRecord` |
| Operational | `EconomicResource.state` | `PendingValidation → Active` via the `validate_new_resource` cross-zome call (once re-enabled) |

---

## 3. What ValiChord provides

### 3.1 The blind commit-reveal protocol

ValiChord uses a four-DNA Holochain architecture:

- **DNA 1 (Researcher Repository)** — researcher's private source chain; stores their expected result commitment before any validator sees it
- **DNA 2 (Validator Workspace)** — each validator's private source chain; stores their sealed verdict before reveal
- **DNA 3 (Attestation)** — shared DHT; coordination space for commitment hashes and reveals
- **DNA 4 (Governance)** — shared DHT; permanent `HarmonyRecord` once consensus is reached

The protocol sequence:
1. Researcher locks expected results: `SHA-256(msgpack(metrics) || nonce)` on their private chain; hash only on the shared DHT
2. Each validator independently seals their findings: `SHA-256(msgpack(attestation) || nonce)` — hash only on DHT, findings stay private
3. Once all commitment hashes are present, `PhaseMarker::RevealOpen` fires
4. Researcher reveals; ValiChord verifies the hash matches the original commitment
5. Each validator reveals; ValiChord verifies hash matches `CommitmentAnchor`
6. `check_and_create_harmony_record()` derives majority consensus → writes immutable `HarmonyRecord`

**The guarantee:** the assessment written to the DHT is provably identical to what was sealed before unblinding. No adaptive reveals are possible.

### 3.2 Outputs

| ValiChord output | Nondominium use |
|---|---|
| `HarmonyRecord` | Triggering event for lifecycle transitions; linked via `NdoToTransitionEvent` |
| `ReproducibilityBadge` (Gold/Silver/Bronze/FailedReproduction) | Discoverable credential on the resource |
| `ValidatorReputation` | Per-validator agreement rate and tier (Provisional/Certified/Senior) |
| Per-validator `ValidationAttestation` | Source for `zome_gouvernance::create_validation_receipt()` |
| Shareable public URL (`GET /record/<hash>`) | No-auth browser verification of the full audit chain |

### 3.3 Version alignment

Both projects target the same Holochain release:

| Dependency | ValiChord | Nondominium |
|---|---|---|
| `hdk` | `0.6` | `0.6` |
| `hdi` | `0.7` | `0.7` |

No version upgrades required on either side to begin integration work.

---

## 4. Integration architecture

A validation round proceeds as follows. **[NDO]** = Nondominium zome call. **[VC]** = ValiChord zome call.

**Step 1 — Resource registered**
Researcher creates `EconomicResource` in NDO **[NDO]** `zome_resource::create_economic_resource()` → `PendingValidation`
ValiChord `validate_new_resource` cross-zome call fires **[VC]** → opens `ValidationRequest`
Researcher locks expected results **[VC]** `researcher_repository::lock_researcher_result()`

**Step 2 — Validators enrolled**
Each validator promoted to accountable agent **[NDO]** `zome_person::promote_agent_to_accountable()`
Each validator publishes profile in ValiChord **[VC]** `attestation::publish_validator_profile()`

**Step 3 — Commit phase (blind)**
Each validator seals findings locally **[VC]** `validator_workspace::seal_private_attestation()`
Commitment hash published to DHT — no content **[VC]** `attestation::notify_commitment_sealed()`

**Step 4 — Reveal phase**
Each validator reveals **[VC]** `attestation::submit_attestation()` — SHA-256 verified on-chain
Researcher reveals **[VC]** `researcher_repository::reveal_researcher_result()`

**Step 5 — HarmonyRecord → NDO**
ValiChord governance DNA produces consensus outcome **[VC]** `governance::check_and_create_harmony_record()`
Per-validator receipts written into Nondominium **[NDO]** `zome_gouvernance::create_validation_receipt()`
Layer 0 lifecycle advanced **[NDO]** `zome_resource::update_lifecycle_stage()` e.g. `Prototype → Stable`, with `NdoToTransitionEvent` link referencing `HarmonyRecord` action hash
Operational state transitioned **[NDO]** `zome_resource::update_resource_state()` → `Active`

**Step 6 — Validator attribution**
For each validator **[NDO]** `zome_gouvernance::log_economic_event(VfAction::Work)` → PPRs issued
Validators receive NDO reputation credit for completing a ValiChord round.

---

## 5. Integration path — five design decisions

These are open questions requiring agreement before integration code is written.

### Decision 1 — Ownership of validation state

Nondominium has `ResourceValidation` with its own `required_validators` / `current_validators` /
`status`. ValiChord has `ValidationRequest` with the same.

**Option A:** ValiChord runs autonomously; on completion, writes outcome into NDO's `ResourceValidation.status`. NDO is authoritative state; ValiChord feeds it.  
**Option B:** `HarmonyRecord` is the authoritative record. NDO's governance rules check for a linked `HarmonyRecord` rather than maintaining their own consensus tracking.

Option A is simpler. Option B avoids duplication but requires NDO governance rules to understand ValiChord entry types.

### Decision 2 — Membrane proofs and NDO roles

ValiChord validators hold Ed25519 institutional credentials (membrane proofs) for DNA 3 access.
NDO validators hold `AccountableAgent` roles via `zome_person::promote_agent_to_accountable()`.

Should a valid ValiChord credential auto-trigger NDO role promotion? Or should each system
manage its own enrollment independently?

### Decision 3 — Who creates the NDO resource?

**Option A:** Researcher creates `EconomicResource` + `NondominiumIdentity` in NDO first, gives hashes to ValiChord when opening a round.  
**Option B:** ValiChord creates them via cross-app call as part of study registration.

Option A preserves NDO as the canonical resource registry. Option B gives researchers a
unified workflow but tightly couples the systems at creation time.

### Decision 4 — Flowsta as shared identity layer

ValiChord knows validators by ValiChord device key; NDO knows them by NDO agent key. These may differ.

**Option A:** Flowsta `IsSamePersonEntry` required for cross-system validators — automated key resolution.  
**Option B:** Optional, with manual key-mapping fallback.

See `flowsta-integration.md` for the full Flowsta architecture.

### Decision 5 — Custodian constraint on `update_resource_state()`

As of the current implementation, `update_resource_state()` is custodian-gated. ValiChord's
governance DNA is not the custodian and cannot call it directly.

**Option A:** Add a governance-authorised `validate_and_activate_resource()` function accepting a `HarmonyRecord` action hash instead of requiring custodianship.  
**Option B:** Keep the custodian gate. After ValiChord produces the `HarmonyRecord`, the researcher (custodian) is notified and calls `update_resource_state()` themselves, providing the `HarmonyRecord` hash as the triggering event reference.

Option A is a tighter integration. Option B keeps NDO's custodian model intact and avoids
coupling the two DNAs at the Rust level.

---

## 6. Requirements traceability

| NDO REQ ID | Summary | ValiChord role |
|---|---|---|
| **REQ-NDO-LC-02** | Lifecycle transitions validated by governance zome as operator | ValiChord's `HarmonyRecord` is the trigger event the governance zome validates |
| **REQ-NDO-LC-03** | Each lifecycle transition generates a triggering `EconomicEvent` | `HarmonyRecord` action hash linked via `NdoToTransitionEvent` |
| **REQ-NDO-LC-05** | `Prototype → Stable` requires multi-agent peer validation (N-of-M) | ValiChord provides the N-of-M blind commit-reveal round |
| **REQ-NDO-OS-02** | `OperationalState` transitions managed by governance zome | ValiChord governance DNA calls `update_resource_state()` after consensus (subject to Decision 5) |
| **REQ-GOV-02** | Resource validation (the commented-out call) | ValiChord is the validation protocol this call initiates |

New REQ-NDO-VAL-* identifiers for ValiChord-specific integration requirements should be added
to `ndo_prima_materia.md` once the five design decisions above are resolved.

---

## 7. Current state

**ValiChord protocol**: production-grade as of v0.3.0 (April 2026). The full 3-validator blind
commit-reveal round — AI researcher + 3 AI validators — runs end-to-end on a live Holochain
conductor on Oracle. The `HarmonyRecord` is queryable at a permanent public URL
(`GET /record/<hash>`) with no authentication required. A Feynman AI research agent integration
is also live (ValiChord skill in Feynman 0.2.16).

**Integration code**: not yet written. The five design decisions in Section 5 are open.

**How to engage**: raise an issue in https://github.com/topeuph-ai/ValiChord or comment on
the integration design documents in `nondominium_integration/` in that repo.
