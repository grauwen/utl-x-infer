# UTL-X — Repository Strategy and Versioning

## Status: Design Document

| Field | Value |
|---|---|
| Document | utlx-repository-strategy |
| Status | Active design |
| Scope | UTL-X repo structure, versioning model, language version vs release version |
| Related | utlx-language-versioning-validation.md, UTLX-2-tabular-foundation-models.md |

---

## 1. The two projects

UTL-X is not one project — it is two, with a clear dependency relationship
between them.

| | UTL-X | UTL-X Infer |
|---|---|---|
| **Repo** | `github.com/grauwen/utl-x` | `github.com/grauwen/utl-x-infer` |
| **Language versions** | `%utlx 1.0`, `%utlx 1.1` | `%utlx 2.0` |
| **Current release** | v1.3.0 | Does not exist yet |
| **Identity** | Pure functional transformation language | Probabilistic inference-capable extension |
| **Primary audience** | Integration developers, data engineers | Data engineers, MLOps, integration developers needing inference |
| **Dependencies** | Zero heavyweight deps | Requires `utl-x ≥ v1.3.0` + TFM runtime |
| **Contract** | Pure, stateless, deterministic — guaranteed | Probabilistic in `mode: component` |
| **Stability** | Mature, conservative releases | Experimental until 2.0 stabilises |
| **Conformance suite** | 465+ tests, all `%utlx 1.0` | Imports full 1.x suite + adds 2.0 tests |
| **Your book** | ✓ — the book's subject | ✗ — not the book's subject |

The relationship is a **dependency, not a fork**. `utl-x-infer` builds on top of
`utl-x`. The parser, UDM 1.0 node types, and core stdlib are imported from `utl-x` as a
**Kotlin/Gradle (Maven) artifact dependency** — not duplicated.

```
com.github.grauwen:utl-x-infer
    └── implementation("com.github.grauwen:utl-x:1.3.0")   // JitPack / Maven artifact
```

---

## 1a. Decision — split by dependency/identity/cadence, NOT by version number

**A question arose: should `%utlx 1.1` be split from `%utlx 2.0` into its own repo, and
should 1.1 live in `utl-x-infer`? Decision: no to both.** The guiding rule — the same one
that justified separating `infer` from core (§2) — is that **repos split on dependency
weight, identity, and release cadence, not on version number.** Applied to 1.1 vs 2.0:

| Criterion | 1.1 (`validate.*`) | 2.0 (`ai.*`) |
|---|---|---|
| Dependencies | **zero heavyweight** — pure Kotlin | DJL + ONNX + TabPFN + GPU |
| Contract | **pure, deterministic** (minor, backward-compat) | breaks purity/determinism (major) |
| Audience | **same as core** | MLOps / inference |
| Cadence | conservative, like core | experimental |

Every row puts **1.1 on the same side as 1.0** — because 1.1 *is* the next minor of the
**pure language line** (1.0 → 1.1 → 1.2 …), which is one evolving codebase. Therefore:

1. **1.1 lives with the pure line in `utl-x`** (where `validate.*` already belongs per §6)
   — **not** in `utl-x-infer`, and **not** in a new dedicated 1.1 repo. Putting 1.1 in
   `infer` would couple a zero-dependency feature to the ML stack and mislabel it;
   a per-minor repo would be sprawl and add a permanent link to the dependency chain
   (`core ← 1.1 ← 2.0`) to version-coordinate forever.
2. **"Keep 1.0 as-is" is a git concern, not a repo concern.** Freeze 1.0 as a **release
   tag / branch** (`v1.0.x`, the book's citable subject) and develop **1.1 on `main`/a
   `next` branch of `utl-x`**. Git versioning — not a repo split — gives a pristine 1.0
   *and* a natural home for 1.1 (how CPython et al. handle minors).
3. **`utl-x-infer` stays 2.0-only.** All the §2 arguments (ML deps, identity, the book)
   apply to 2.0, not to 1.1.
4. **Interim note:** a draft of 1.1 / post-1.0 material was parked in `utl-x-infer`
   during exploration. Target state is 1.1 on the `utl-x` pure line per (1)–(2); if
   touching `utl-x` must wait, the least-bad interim is a single `utl-x-next` staging
   repo that **folds back** into `utl-x` at release — still never merged with `infer`.

### Recommended end-state

```
utl-x        pure language — 1.0 (frozen tag/release) · 1.1 (main/next) · 1.2 … ; validate.* here
utl-x-infer  2.0 probabilistic only        → implementation("com.github.grauwen:utl-x:x.y.z")
utlx-mil     defence edition (BINF, guard) → implementation("com.github.grauwen:utl-x:x.y.z")
```

### Can repos link to each other?

Yes — and the cost of each link is why repo count should stay minimal:

- **Code → versioned artifact dependency (preferred):** publish `utl-x` to Maven/JitPack
  (`com.github.grauwen:utl-x:x.y.z`); `infer` and `mil` depend on a pinned version. The
  "dependency, not fork" model. *Cost:* each boundary = a publish + a version to bump/pin.
- **Docs → full GitHub URLs** for cross-repo links (relative paths only resolve when
  repos are checked out side-by-side).
- **Git submodules** work but are painful — prefer artifacts. A **monorepo with Gradle
  modules** is the opposite option (no cross-repo overhead), rejected here on
  perception/identity grounds (§2) but technically valid.

---

## 2. Why two repos, not one

### The standalone identity argument

UTL-X (`utl-x`) is a standalone open-source language project that exists
independently of Open-M. It has its own conformance suite, its own LSP daemon,
its own CLI. People use UTL-X without Open-M. The project has a public identity
that is not "the mapping component of Open-M" — it is a language.

UTL-X Infer (`utl-x-infer`) is a different thing with a different audience,
different dependencies, and a different stability profile. It deserves its own
identity and its own community space.

### The book argument

There is a published book on AI-optimized mapping strategies for UTL-X 1.0.
That book is about `utl-x` — the pure, deterministic transformation language.
Naming the 2.0 repo `utl-x-ai` would create direct confusion:

- Is the book about `utl-x` or `utl-x-ai`?
- Do I need `utl-x-ai` to use the AI strategies from the book?
- Is UTL-X 1.0 the AI version?

The name `utl-x-infer` is deliberately distinct from the book's subject. The
book covers AI-optimized *mapping* (a 1.0 concern). The repo covers
*probabilistic inference* (a 2.0 concern). No overlap.

### The dependency pollution argument

UTL-X 2.0's `ai.*` stdlib requires TFM model loading, tensor operations, and
optionally GPU interaction. These are megabytes of transitive dependencies that
have no place in a 1.0 or 1.1 deployment.

In a monorepo with two Gradle modules, the dependency boundary exists technically
but not perceptually. Contributors still work in one repo. New users reading the
repo see both and have to understand the distinction before writing a single
script.

With two repos:

- A user who needs pure field mapping reads `github.com/grauwen/utl-x` — the
  README describes a pure transformation language, full stop.
- A data engineer evaluating TFM integration reads `github.com/grauwen/utl-x-infer`
  — the README describes probabilistic mapping, requires model registry,
  different audience.

### The release cadence argument

UTL-X 1.x is at v1.3.0 with a mature conformance suite. Future releases are
conservative — new stdlib functions, bug fixes, the `validate.*` namespace.

UTL-X Infer is experimental. The `ai.*` namespace, `ProbabilisticCell` UDM
nodes, Model Registry integration — these are early-stage design proposals.
Rapid breaking changes will happen before stabilisation. Mixing a mature
conservative project with an experimental fast-moving one in the same repo
creates noise for both audiences.

---

## 3. Why "infer" not "ai" or "probabilistic"

The naming decision matters. The 2.0 repo name must describe what is
**architecturally new at the language level** — not what use case motivates it.

What is actually new in 2.0, stripped of the AI framing:

- **Probabilistic cell values** — a value carrying a confidence score and
  source annotation alongside the value itself
- **Non-deterministic evaluation** — same input can produce different output
  depending on model state
- **Stateful execution** — model weights are state, held across evaluations
- **Inferred values** — values not observed in the input but derived by a model

The common thread is **inference** — deriving values probabilistically from
models or rules. A rule-based imputation engine, a Bayesian classifier, and
a statistical outlier detector would all benefit from the 2.0 UDM extensions,
even without a neural network.

### Candidate names evaluated

| Name | Verdict | Reason |
|---|---|---|
| `utl-x-ai` | ✗ Rejected | Conflicts with book. AI is already a 1.0 story. |
| `utl-x-probabilistic` | ✗ Too long | Hard to say. Unlikely to become community shorthand. |
| `utl-x-stochastic` | ✗ Too academic | Most developers do not search for "stochastic". |
| `utl-x-extended` | ✗ Too generic | Every v2 of everything is "extended". No signal. |
| `utl-x-plus` | ✗ Too generic | Same problem. |
| `utl-x-p` | ✗ Too cryptic | "What does the p mean?" is a bad first question. |
| `utl-x-prob` | ~ Functional | Reasonable but not elegant. |
| `utl-x-inference` | ~ Good | Accurate but longer than needed. |
| `utl-x-infer` | ✓ **Chosen** | Short, speakable, technically honest, future-proof. |

### Why `utl-x-infer` wins

- **Accurate** — inference is what the 2.0 execution model adds. Values are
  inferred, not just observed and transformed.
- **Distinct from the book** — the book covers mapping strategies, not inference
  semantics. Zero overlap.
- **Short and speakable** — "I'm using utl-x-infer" is natural in conversation.
- **Technically honest** — covers TFM imputation, classification, anomaly
  detection, and any future probabilistic capability without being locked to
  neural networks or "AI" as a buzzword.
- **Future-proof** — if 2.0 gains capabilities beyond TFM inference (Bayesian
  networks, Monte Carlo simulation, fuzzy logic), the name still fits.

---

## 4. The two-track version model

UTL-X has two independent versioning concepts that must not be conflated:

| Version | What it is | Example |
|---|---|---|
| **Script header version** (`%utlx 1.0`) | Language feature set declaration — tells the engine which capabilities and contracts apply | `%utlx 1.0`, `%utlx 1.1`, `%utlx 2.0` |
| **Engine release version** (`v1.3.0`) | Software release — bug fixes, performance improvements, new stdlib functions that don't change the language contract | `v1.0.0`, `v1.1.0`, `v1.2.0`, `v1.3.0` |

These are different things. A script declaring `%utlx 1.0` means: **I use the
1.0 language contract — pure, stateless, deterministic**. It says nothing about
which engine release version runs it.

The engine at `v1.3.0` runs `%utlx 1.0` scripts. So will the engine at
`v1.5.0`, `v1.8.0`, and `v2.0.0`. The header version is a **minimum language
version** the script requires — not the engine release it was written against.

### Version history — `github.com/grauwen/utl-x`

```
Engine release    Script header      What changed
────────────────────────────────────────────────────────────────────
v1.0.0            %utlx 1.0          Initial release
v1.1.0            %utlx 1.0          New stdlib functions, bug fixes
v1.2.0            %utlx 1.0          More stdlib, performance
v1.3.0  ← now    %utlx 1.0          More stdlib, bug fixes
  │
  ▼ future
v1.x.0            %utlx 1.1          validate.* namespace added
                                      Scripts using validate.* declare %utlx 1.1
                                      Scripts declaring %utlx 1.0 unchanged
  │
  ▼ future
v2.0.0            %utlx 1.0 + 1.1    Engine still supports all 1.x scripts
                                      Does NOT add 2.0 — that is utl-x-infer
```

### Version history — `github.com/grauwen/utl-x-infer`

```
Engine release    Script header      What changed
────────────────────────────────────────────────────────────────────
(does not exist yet)
  │
  ▼ future
v1.0.0            %utlx 2.0          First stable release of 2.0 engine
                                      ai.* stdlib, ProbabilisticCell UDM
                                      Requires utl-x ≥ v1.3.0
```

Note: `utl-x-infer` has its **own release version sequence** starting at
`v1.0.0`. It does not start at `v2.0.0`. The `%utlx 2.0` script header version
is a language version, not the repo's release version. These are independent
sequences.

### The version gate — how it works

The first line of every UTL-X script is the version declaration:

```
%utlx 1.0    ← runs on utl-x engine ≥ v1.0.0
%utlx 1.1    ← runs on utl-x engine ≥ v1.x.0 (when validate.* ships)
%utlx 2.0    ← runs on utl-x-infer engine ≥ v1.0.0
```

Engine behaviour:

| Script header | utl-x engine | utl-x-infer engine |
|---|---|---|
| `%utlx 1.0` | ✓ runs | ✓ runs (backward compat) |
| `%utlx 1.1` | ✓ runs (if ≥ v1.x.0) | ✓ runs (backward compat) |
| `%utlx 2.0` | ✗ rejected — informative error | ✓ runs |

A `utl-x` engine encountering a `%utlx 2.0` script rejects it immediately with:

```
UTL-X engine v1.3.0 cannot execute %utlx 2.0 scripts.
%utlx 2.0 requires the UTL-X Infer engine (github.com/grauwen/utl-x-infer).
```

---

## 5. What scripts declare today and going forward

**All existing UTL-X scripts declare `%utlx 1.0`.** They run on engine v1.3.0.
They will run on every future 1.x engine release. They will run on `utl-x-infer`
when it ships. The header `%utlx 1.0` is stable and will not need changing.

```
%utlx 1.0       ← declare this for all pure transformation scripts
                   Works on: utl-x v1.0.0 through current and future 1.x
                   Works on: utl-x-infer v1.0.0+ (backward compat)

%utlx 1.1       ← declare this when using validate.* stdlib
                   Works on: utl-x ≥ v1.x.0 (when validate.* ships)
                   Works on: utl-x-infer v1.0.0+

%utlx 2.0       ← declare this when using ai.* stdlib
                   Requires: mode: component in header
                   Works on: utl-x-infer v1.0.0+ only
```

---

## 6. Shared components — where they live

`utl-x-infer` reuses from `utl-x` via **Kotlin/Gradle (Maven) artifact import** — not
duplication:

| Component | Lives in | Imported by |
|---|---|---|
| Parser / tokeniser / lexer | `utl-x` | `utl-x-infer` via Gradle dep |
| UDM 1.0 node types | `utl-x` | `utl-x-infer` via Gradle dep |
| Core stdlib (1.0 functions) | `utl-x` | `utl-x-infer` via Gradle dep |
| validate.* stdlib (1.1) | `utl-x` | `utl-x-infer` via Gradle dep |
| 1.0 / 1.1 evaluator | `utl-x` | `utl-x-infer` via Gradle dep |
| UDM 2.0 node types | `utl-x-infer` | — |
| ai.* stdlib | `utl-x-infer` | — |
| 2.0 evaluator | `utl-x-infer` | — |
| TFM runtime dependencies | `utl-x-infer` | — |
| Model Registry client | `utl-x-infer` | — |

```kotlin
// utl-x-infer — build.gradle.kts
dependencies {
    implementation("com.github.grauwen:utl-x:1.3.0")               // parser, UDM 1.0, core stdlib, validate.* (1.1)
    implementation("ai.djl:api:0.26.0")                            // TFM runtime — only here, never in utl-x
    implementation("ai.djl.onnxruntime:onnxruntime-engine:0.26.0")
}
```

A project depending on `com.github.grauwen:utl-x` gets zero TFM dependencies.
A project depending on `com.github.grauwen:utl-x-infer` gets both.

---

## 7. Conformance suite

### `utl-x` conformance suite

Lives in `github.com/grauwen/utl-x/conformance`. Covers all `%utlx 1.0` and
`%utlx 1.1` scripts. Currently 465+ passing tests. Every engine release must
pass the full suite before tagging.

```bash
# Run from utl-x repo
go test ./conformance/...
```

### `utl-x-infer` conformance suite

Lives in `github.com/grauwen/utl-x-infer/conformance`. Covers `%utlx 2.0`
scripts. **Imports and runs the full 1.x conformance suite** to assert backward
compatibility — this is not a convention, it is an enforced test requirement:

```go
// utl-x-infer/conformance/backward_compat_test.go
import utlxconformance "github.com/grauwen/utl-x/conformance"

func TestBackwardCompatibility(t *testing.T) {
    // The 2.0 engine must pass every 1.x test
    engine := utlxinfer.NewEngine()
    utlxconformance.RunAll(t, engine)
}
```

Backward compatibility is not a README promise — it is an imported test suite
that must pass on every release.

---

## 8. LSP daemon

### `utl-x` LSP daemon

Exists today in `github.com/grauwen/utl-x`. Provides:

- `%utlx 1.0` and `%utlx 1.1` autocomplete
- Core stdlib function signatures and documentation
- validate.* completion (when 1.1 ships)
- Schema-aware field tree rendering from TSCH/JSCH

The LSP daemon never knows `utl-x-infer` exists. It does not need to.

### `utl-x-infer` LSP extension

`utl-x-infer` ships a separate LSP extension that registers additional
completions for `%utlx 2.0` scripts:

- `ai.*` function signatures and documentation
- `mode:` header completion
- `model:` header completion with Model Registry lookup
- `ProbabilisticCell` metadata property completion (`._confidence`,
  `._source`, `._model_ref`)

The VS Code extension loads both the 1.x LSP daemon and the 2.0 LSP extension
when both are installed. Standard LSP extension composition — neither knows
about the other.

---

## 9. CLI

### `utl-x` CLI

The existing `utlx` binary handles 1.x scripts:

```bash
utlx run     my-mapping.utlx
utlx validate my-mapping.utlx
utlx test    ./conformance/
utlx format  my-mapping.utlx
```

### `utl-x-infer` CLI

`utl-x-infer` ships a companion binary `utlx-infer`:

```bash
utlx-infer run      my-ai-mapping.utlx --model-registry http://registry:8080
utlx-infer validate my-ai-mapping.utlx
utlx-infer test     ./conformance/
```

Alternatively — if the `utlx` CLI supports plugins in the future:

```bash
utlx plugin install utlx-infer
utlx run my-ai-mapping.utlx   # dispatches to utlx-infer plugin for %utlx 2.0
```

The plugin model is optional and can be added later. Start with two binaries.

---

## 10. Open-M integration — which repo does the wrapper use?

The Open-M wrapper imports exactly what it needs and nothing more:

| Component | Imports |
|---|---|
| Open-M wrapper (Mode 2 inline, Mode 2 ref) | `github.com/grauwen/utl-x` only |
| Open-M UTL-X mapping component (Mode 3, pure) | `github.com/grauwen/utl-x` only |
| Open-M UTL-X mapping component (Mode 3, `%utlx 2.0`) | `github.com/grauwen/utl-x-infer` |

The Mode 3 mapping component that executes `%utlx 2.0` scripts is a
**separate Open-M component image** — it has different resource requirements
(GPU nodeSelector, model registry access, higher memory limits) and is
deployed independently from Mode 3 components running `%utlx 1.x` scripts.

```yaml
# Pipeline YAML — Mode 3 pure mapping component (utl-x)
components:
  - id:  field-mapper
    ref:  open-m.components.utlx-mapper:1.0.0      # uses utl-x

# Pipeline YAML — Mode 3 inference mapping component (utl-x-infer)
  - id:  quality-gate
    ref:  open-m.components.utlx-infer-mapper:1.0.0  # uses utl-x-infer
    resources:
      limits:
        nvidia.com/gpu: "1"
```

---

## 11. When to create `utl-x-infer`

The repo does not exist yet and should not be created until there is
production-quality code to put in it. The design is specified. The
implementation is future work.

**Prerequisites before creating the repo:**

- [ ] UDM 2.0 node types designed and reviewed (`ProbabilisticCell`,
  `ColumnAnnotation`, `TableMeta`)
- [ ] `ai.*` stdlib function signatures finalised
- [ ] Model Registry client interface designed
- [ ] At least one TFM integration (TabPFN v2) working in a proof of concept
- [ ] `%utlx 2.0` parser extension implemented and passing basic tests
- [ ] Backward compatibility test harness written and passing against `utl-x`
  conformance suite

Until these are met, design work lives in the `utl-x` repo under
`docs/proposals/utlx-2-infer/` — tracked as a proposal, not a released
capability.

---

## 12. Summary

```
github.com/grauwen/utl-x             github.com/grauwen/utl-x-infer
────────────────────────             ──────────────────────────────
Language: %utlx 1.0, 1.1            Language: %utlx 2.0
Current:  v1.3.0                    Current:  does not exist yet
Contract: pure, stateless,          Contract: probabilistic in
          deterministic                        mode: component
Deps:     zero heavyweight           Deps:     utl-x + TFM runtime
Audience: integration developers    Audience: data engineers + MLOps
Book:     ✓ the book's subject       Book:     ✗ not the book's subject
LSP:      full 1.x daemon            LSP:      2.0 extension, loads alongside
CLI:      utlx binary                CLI:      utlx-infer binary
Tests:    465+ conformance tests     Tests:    imports 1.x suite + 2.0 tests

Relationship: utl-x-infer depends on utl-x
              Parser, UDM 1.0, core stdlib imported — not duplicated
              Backward compat enforced by imported conformance suite
```

The script header version (`%utlx 1.0`, `%utlx 1.1`, `%utlx 2.0`) is a
**language feature gate** — not a release version label. The engine release
version (`v1.3.0`, `v1.4.0`) is a **software artifact version**. They evolve
on separate tracks. The header version changes rarely — only when a new
capability requires a different engine contract. The release version changes
with every bug fix, performance improvement, and stdlib addition.

All existing `%utlx 1.0` scripts run unchanged on every future engine release
of both `utl-x` and `utl-x-infer`. The header declaration is stable.
