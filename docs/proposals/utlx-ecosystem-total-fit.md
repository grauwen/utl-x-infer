# UTL-X Ecosystem — Total Fit Across 1.1, 2.0 (Infer) and MIL

## Status: Design Document

| Field | Value |
|---|---|
| Document | utlx-ecosystem-total-fit |
| Status | Analysis — synthesis across repos |
| Created | 2026-09-29 |
| Scope | How the UTL-X expansion directions (1.1 semantic validation, 2.0 Infer, MIL/defence) fit together as one platform |
| Sources | `utl-x` v1.3.1 codebase; `utl-x-infer/docs/proposals/*`; `utlx-mil/docs/*` |
| Related | utlx-language-versioning-validation.md, utlx-repository-strategy.md, UTLX-2-tabular-foundation-models.md, utlx-1_1-semantic-validation.md, utlx-mil/docs/UTL-X-MIL-Possibilities.md |

> This document synthesises three independently-authored expansion directions and
> assesses whether they form a coherent whole. It is analysis, not a commitment to
> build any particular item. Effort figures are order-of-magnitude and taken from
> the source documents.

---

## 1. The unifying model — one axis decides everything

UTL-X 1.0 is defined by four guarantees (from
`utlx-language-versioning-validation.md`):

| Guarantee | Definition |
|---|---|
| **Pure** | Same input always produces same output |
| **Stateless** | No external dependencies, no side effects, no model weights |
| **Deterministic** | No probability, no uncertainty, no randomness |
| **Single-pass** | One input (or N named inputs from one correlation chain) → one output |

These are what make a 1.0 script safe to run **inline on the connection arrow**,
in-process, without a pod or GPU. They are the language's defining contract, not
incidental properties.

**The central finding of this analysis:** whether an extension is a *minor*, a
*major*, or a *separate closed edition* is decided by **which guarantee it
touches**. Every expansion direction — semantic validation, probabilistic
inference, and military data links — maps cleanly onto this one axis. That is what
makes them a platform rather than a pile of features.

---

## 2. Fit map — each direction placed on the contract axis

| Direction | What it adds | Guarantee touched | Correct home | Version |
|---|---|---|---|---|
| **1.1 semantic validation** (`validate.*`) | `ValidationResult` node + cross-field assertions | **none** — stays pure/deterministic | `utl-x` core | `%utlx 1.1` (minor) |
| **MIL BINF** (bit-level binary format) | a new *format* (peer of XML/JSON/CSV) | **none** — a reader/serializer is additive | `utl-x` core | 1.x |
| **MIL streaming** | document → per-message execution | single-pass (execution model, not per-message semantics) | core infra | 1.x infra |
| **MIL transport** (UDP/TCP/JREAP-C) | message I/O with protocol headers | **not the language at all** | host / adapter layer | n/a |
| **2.0 Infer** (`ai.*`) | probabilistic cells, model inference | **purity + determinism** — broken | `utl-x-infer` (separate repo) | `%utlx 2.0` (major) |
| **MIL stateful gateway** | cross-message state (track mapping, correlation) | **statelessness** — broken | closed defence edition, *not* core | edition-only |

**Why the fit is strong and not coincidental:** the MIL possibilities document
independently arrived at the same rule the versioning document formalises —
"keep the core stateless and pure; put state only in the closed edition"
(MIL §5.5). Two documents written for different purposes converged on the same
contract-preservation principle. That convergence is the best evidence the
platform has a real design spine.

---

## 3. Grounding — claims checked against the v1.3.1 codebase

| Claim | Verified in code |
|---|---|
| UDM already has a raw-bytes type | ✅ `modules/core/.../udm/udm_core.kt` — `data class Binary(val data: ByteArray) : UDM()` |
| No bit-level binary *format* module exists yet (BINF is a real gap) | ✅ no BitReader/bitfield/`bit_order` module anywhere |
| Execution is document-oriented, not per-message streaming | ✅ confirmed |
| A transport abstraction already exists to extend | ⚠️ `modules/engine/.../transport/GrpcTransport.kt` exists (daemon); no UDP/TCP/multicast yet — so the transport layer is *partly* seeded, slightly helping the MIL estimate |

The four "gaps" that drive the MIL roadmap and the "new UDM node type" mechanism
that drives 1.1/2.0 are all real and correctly identified by their source docs.

---

## 4. Where the directions reinforce each other

1. **`mode: component` is the missing unifier for MIL.** The Infer proposal
   formalises "non-pure / non-deterministic scripts run in a dedicated pod, never
   inline" as `mode: component`. MIL separately invents the same idea (streaming
   runtime §5.3, gateway extensions §5.5) but does not name it. They are the same
   concept — **out-of-line, resource-bearing execution.** MIL should reuse the
   Infer execution-mode framework rather than build a parallel runtime. This is
   the single biggest cross-project fit opportunity.

2. **BINF closes a gap that is not military.** UDM has `Binary` but no bit-level
   reader. BINF is a generic 1.x *format* consumed by civil (AIS/ASTERIX/ADS-B),
   MIL (J-series), and even Infer (binary tabular feeds). It belongs in **`utl-x`
   core**, exactly as the MIL doc's "open core first" says — not as a military
   feature.

3. **The open-core / edition split is identical across both expansions.** Infer
   keeps ML dependencies (DJL/ONNX/GPU) out of core via a separate repo; MIL keeps
   defence packs + transport + statefulness in a closed edition. Same pattern,
   same motivation (dependency + governance isolation). The repo-strategy
   rationale generalises cleanly to a third repo `utlx-mil`.

4. **One authoring surface serves all of it.** The Theia IDE (Message-Contract
   mode, schema/USDL tooling, `.utlxp` project bundles) authors 1.0, 1.1, BINF and
   2.0 mappings equally. Nothing in the IDE forks per-direction.

---

## 5. Tensions to resolve (this is where total fit needs work)

1. **Core roadmap contention.** BINF (MIL Phase 1) and `validate.*` (1.1) are both
   "the next additive thing in `utl-x` core," competing for the same conservative
   release slot and the same developer. No technical conflict — a contention for
   attention. Needs an explicit ordering (§7).

2. **Duplicate execution-mode concepts.** Unify MIL streaming/gateway under Infer's
   `mode: component` (or a shared parent concept) **before** either is built, or
   the platform ends up with two out-of-line runtimes.

3. **Metadata sigil: `^` vs `._` — reconcile before 2.0 stabilises.** Core UTL-X
   already has a **metadata accessor, the caret `^`** (verified in
   `parser_impl.kt`: `@` = attribute, `^` = metadata, `.` = data), backed by the
   UDM per-node `metadata` map (format-fidelity; dropped on plain output, preserved
   for round-trip). The **Infer 2.0** proposal independently introduced a *different*
   convention — `._confidence`, `._source`, `._model_ref` — for probabilistic
   metadata, and the **MIL BINF** design wants the same channel for decode
   provenance (matched message, CRC status, raw bytes, bit offsets, sentinels).
   These are the same concept ("metadata about a value") under two syntaxes. `^`
   looks like the **one unifying metadata sigil** across BINF provenance *and* Infer
   annotations (e.g. `$row.unitPrice^confidence`). Decide this **before** `%utlx 2.0`
   locks its stdlib, or the platform ships two metadata conventions. *Sub-point:* the
   metadata map is currently `Map<String,String>` — typed values (confidence float,
   CRC bool) need string coercion or a widened metadata type. See
   `utlx-mil/docs/BINF-bit-level-binary-format.md` §1b.

4. **Repo-strategy doc bug — FIXED (Oct 2026).** `utlx-repository-strategy.md` had stated
   utl-x-infer imports utl-x "as a **Go module** dependency"; corrected to **Kotlin/Gradle**
   (`implementation("com.github.grauwen:utl-x:1.3.0")`) throughout. The same doc now also
   carries the repo-split decision (§1a: split by dependency/identity/cadence, not version;
   1.1 stays on the pure `utl-x` line, `infer` is 2.0-only). (This and the `^`/`._` split in
   (3) shared a root cause: docs authored independently and not cross-checked.)

5. **Cross-doc drift.** stdlib count appears as 652 (MIL), 635 (Infer README),
   "652" (code claim); conformance as "465+". Cosmetic, but a sign the docs were
   authored in isolation and not cross-checked.

6. **Focus, not fit, is the real risk.** Fit is genuinely good; the danger is a
   small team carrying four expansion vectors plus the IDE plus Open-M. "Do they
   fit?" → yes. "Can they be sequenced without starving each other?" → only with
   discipline.

---

## 6. Platform architecture (the whole picture)

```
+-----------------------------------------------------------------+
|  Authoring:  UTL-X Theia IDE (MC mode, USDL, .utlxp bundles)     |
+-----------------------------------------------------------------+
|  Editions:   utl-x-infer (2.0, ai.*, DJL/ONNX)                  |
|              utlx-mil    (defence packs, JREAP-C, stateful gw)  |
+-----------------------------------------------------------------+
|  Core:       utl-x  1.0  → 1.1 (validate.*)  → BINF + streaming |
|              language · UDM · core stdlib · XML/JSON/CSV/YAML   |
+-----------------------------------------------------------------+
|  Host:       Open-M (inline arrow = pure 1.x; Mode 3 pod =      |
|              mode: component for infer / streaming / gateway)   |
+-----------------------------------------------------------------+
```

- **Inline path** (Open-M connection arrow): only pure/deterministic `%utlx 1.0`
  and `%utlx 1.1`.
- **Component path** (Open-M Mode 3 pod): everything that breaks a guarantee —
  `ai.*` (2.0), MIL streaming/gateway — via one shared `mode: component` contract.

---

## 7. Sequencing that respects the fit

The dependency graph practically dictates the order:

1. **1.1 `validate.*` first.** Smallest, purest, same repo, no new heavy deps,
   immediately useful; exercises the "additive UDM node + version gate" machinery
   that both BINF and 2.0 reuse. Lowest risk, highest leverage.
2. **BINF + streaming in core** (MIL Phase 1 / open core). Standalone civil value
   (AIS/ASTERIX/CoT), no restrictions, shared substrate for MIL and binary Infer
   feeds. Build as a **core** capability, not a MIL one.
3. **Unify `mode: component`** as the one out-of-line execution contract — before
   Infer or MIL-gateway implementation.
4. **2.0 Infer** — after 1.1 proves the version-gate / UDM-extension pattern; it is
   the heaviest dependency stack and most experimental.
5. **MIL defence edition** — gated entirely on non-engineering preconditions
   (export-control advice + legitimate standards access), as
   `UTL-X-MIL-Possibilities.md` §6 and §8 correctly insist. No defence-specific
   code until Phase 0 clears.

---

## 8. Effort context (from source docs)

| Direction | Source estimate | Notes |
|---|---|---|
| 1.1 `validate.*` | (not separately estimated) | Additive stdlib namespace + one UDM node type |
| MIL Phase 1 (open core: BINF, streaming, transport, AIS/ASTERIX/CoT) | ~19–31 pw (≈4–7 months, 1 dev) | Low risk; standalone civil value |
| MIL Phase 2 (defence MVP) | ~22–42 pw + SME (≈6–12 months) | Gated on Phase 0; SME time is the real bottleneck |
| 2.0 Infer | (status: design phase) | DJL + ONNX + TabPFN; heaviest deps; separate repo |

Not included anywhere: formal interoperability certification, security
accreditation, full J-series coverage, operational-grade stateful gateway.

---

## 9. Bottom line

The three directions fit unusually well because they are not three ideas — they
are three points on one deliberately-designed axis (*which guarantee does this
touch?*), and two independently-written documents converged on the same
contract-preservation rule.

The open questions are **not about fit** — they are about **discipline**:

- unify the execution-mode concept (`mode: component` ⊇ MIL streaming/gateway),
- unify the **metadata sigil** — one `^` for BINF provenance *and* Infer annotations, not a separate `._`,
- order the core roadmap (**1.1 → BINF/streaming → 2.0**),
- fix the repo-strategy doc's Go/Kotlin slip,
- and resist running all vectors at once with one team.

The architecture is coherent. The risk is bandwidth, not design.
