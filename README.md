# UTL-X Infer

**Probabilistic mapping extension for UTL-X — inference-capable values, `ai.*` stdlib, and `%utlx 2.0` engine**

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL_v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![Kotlin](https://img.shields.io/badge/Kotlin-1.9+-7F52FF.svg)](https://kotlinlang.org)
[![JVM](https://img.shields.io/badge/JVM-17+-orange.svg)](https://adoptium.net)
[![UTL-X](https://img.shields.io/badge/UTL--X-%E2%89%A5_v1.3.0-amber.svg)](https://github.com/grauwen/utl-x)
[![Status](https://img.shields.io/badge/Status-Experimental-orange.svg)]()

---

UTL-X Infer extends [UTL-X](https://github.com/grauwen/utl-x) with
**probabilistic value semantics** — the ability to work with values that carry
a confidence score, a source annotation, and a model provenance reference
alongside the value itself.

Where UTL-X 1.x transforms observed, deterministic data, UTL-X Infer adds a
second class of value: the **inferred value** — produced by a Tabular
Foundation Model, a statistical imputer, a classifier, or any probabilistic
inference engine registered in the Open-M Model Registry.

Scripts declare `%utlx 2.0` and gain access to the `ai.*` standard library
namespace. All `%utlx 1.0` and `%utlx 1.1` scripts run unchanged on the UTL-X
Infer engine — full backward compatibility is enforced by the imported UTL-X
conformance suite, not just promised in a README.

UTL-X Infer is written in **Kotlin** — the same language as UTL-X. It imports
`utl-x` as a standard Gradle dependency and builds directly on its parser, UDM,
and evaluator. No cross-language bridge. No duplicated code.

> **This project is experimental.** The `%utlx 2.0` specification and `ai.*`
> stdlib are under active design. Breaking changes will occur before the first
> stable release. See [Status](#status).

---

## Contents

- [What UTL-X Infer adds](#what-utl-x-infer-adds)
- [Relationship to UTL-X](#relationship-to-utl-x)
- [Quick example](#quick-example)
- [The ai.* stdlib](#the-ai-stdlib)
- [Probabilistic UDM nodes](#probabilistic-udm-nodes)
- [Script header — %utlx 2.0](#script-header---utlx-20)
- [Installation](#installation)
- [Usage](#usage)
- [Model Registry](#model-registry)
- [Open-M integration](#open-m-integration)
- [Conformance](#conformance)
- [Status](#status)
- [Contributing](#contributing)
- [License](#license)

---

## What UTL-X Infer adds

UTL-X 1.x is **pure and deterministic** — same input always produces same
output. This is the right contract for field mapping, format conversion, and
structural transformation. It is what the language is built for.

Some integration scenarios require more. A supplier CSV feed where 15% of
`unitPrice` fields are missing. An inbound EDIFACT batch where row type must
be inferred because no explicit type field exists. A high-volume data pipeline
where statistical outliers need flagging before data reaches the ERP.

These scenarios need values to be **inferred**, not just mapped. UTL-X Infer
provides this through:

| Capability | What it does |
|---|---|
| **Probabilistic cell values** | Scalar values carrying `._confidence`, `._source`, and `._model_ref` metadata alongside the value itself |
| **`ai.*` stdlib namespace** | Functions for imputation, classification, quality scoring, anomaly detection, and schema inference |
| **`mode: component` execution** | Scripts declaring `mode: component` may be non-pure and non-deterministic — they run in a dedicated pod, never inline in the receiving wrapper |
| **Model Registry client** | Resolves model references (`namespace.models.name:version`) at runtime |
| **UDM 2.0 node types** | `ProbabilisticCell`, `ColumnAnnotation`, `TableMeta` — extensions to the Universal Data Model |

---

## Relationship to UTL-X

UTL-X Infer is **not a fork**. It is a Kotlin library that depends on
`github.com/grauwen/utl-x` and extends it. The parser, UDM 1.0 node types,
core stdlib, and `validate.*` namespace are imported as standard Gradle
dependencies — not duplicated.

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.github.grauwen:utl-x:1.3.0")              // parser, UDM 1.0, core stdlib
    implementation("ai.djl:api:0.26.0")                            // Deep Java Library
    implementation("ai.djl.onnxruntime:onnxruntime-engine:0.26.0") // ONNX runtime for TFM models
}
```

**What lives where:**

| Component | Repo |
|---|---|
| Parser, tokeniser, lexer | `utl-x` — imported |
| UDM 1.0 node types | `utl-x` — imported |
| Core stdlib (635 functions) | `utl-x` — imported |
| `validate.*` stdlib (1.1) | `utl-x` — imported |
| 1.0 / 1.1 evaluator | `utl-x` — imported |
| UDM 2.0 node types | `utl-x-infer` |
| `ai.*` stdlib | `utl-x-infer` |
| 2.0 evaluator | `utl-x-infer` |
| Model Registry client | `utl-x-infer` |
| TFM runtime (DJL + ONNX) | `utl-x-infer` |

A project that depends on `utl-x` gets zero probabilistic or TFM dependencies.
Only projects that explicitly depend on `utl-x-infer` pull in the inference
stack.

---

## Quick example

A supplier CSV feed with missing `unitPrice`, `taxClass`, and `shipCountry`
fields. Impute missing values, score data quality, filter low-confidence rows,
and map to the downstream schema:

```
%utlx 2.0
input:  rows csv
output: json
mode:   component
model:  logistics.models.tabpfn-v2:2.1.0
---
$rows
  |> ai.impute(
       columns: ["unitPrice", "taxClass", "shipCountry"],
       model:   "logistics.models.tabpfn-v2:2.1.0"
     )
  |> ai.score(
       model:     "logistics.models.quality-scorer-v1:1.0.0",
       threshold: 0.85
     )
  |> filter(row =>
       row.unitPrice._confidence >= 0.90 &&
       row._anomaly == false
     )
  |> map(row => {
       sku:          row.productCode,
       price:        row.unitPrice,
       priceConf:    row.unitPrice._confidence,
       priceSource:  row.unitPrice._source,
       taxClass:     row.taxClass,
       shipCountry:  row.shipCountry,
       qualityScore: $rows._meta.quality_score
     })
```

The same pipeline without inference — using UTL-X 1.0, no special engine
required:

```
%utlx 1.0
input: current json
output json
---
{
  sku:         $current.productCode,
  price:       $current.unitPrice,
  taxClass:    $current.taxClass,
  shipCountry: $current.shipCountry
}
```

The 1.0 script runs on the base UTL-X engine and maps what is present.
The 2.0 script runs on UTL-X Infer and fills what is missing before mapping.
Both are correct — the choice depends on whether inference is needed.

---

## The `ai.*` stdlib

All `ai.*` functions operate on UDM array-of-objects (table) nodes and return
modified versions of the same table with probabilistic cells where values were
inferred or transformed.

`ai.*` functions are a **parse-time error** in `%utlx 1.0` and `%utlx 1.1`
scripts, and in any `%utlx 2.0` script that does not declare `mode: component`.
Misconfigured scripts are caught before execution begins — not at runtime.

### `ai.impute`

Fill missing or null values using a registered TFM model. Only null cells
are filled — observed values are never overwritten.

```
$rows |> ai.impute(
  columns: ["unitPrice", "taxClass"],
  model:   "logistics.models.tabpfn-v2:2.1.0"
)
```

### `ai.classify`

Add a classification column derived by a model.

```
$rows |> ai.classify(
  target_column: "recordType",
  model:         "logistics.models.record-classifier-v1:1.0.0"
)
```

### `ai.score`

Score overall data quality. Sets `$rows._meta.quality_score` (0.0–1.0).

```
$rows |> ai.score(
  model:     "logistics.models.quality-scorer-v1:1.0.0",
  threshold: 0.85
)
```

### `ai.anomaly`

Flag anomalous rows. Sets `row._anomaly` and `$rows._meta.anomaly_count`.

```
$rows |> ai.anomaly(
  model:       "logistics.models.anomaly-detector-v1:1.0.0",
  sensitivity: 0.95
)
```

### `ai.infer_schema`

Populate `ColumnAnnotation` semantic fields — what each column means, not just
its data type. Useful for unknown partner data with no schema documentation.

```
$rows |> ai.infer_schema(
  model: "open-m.models.schema-inferrer-v1:1.0.0"
)
```

### `ai.clamp_outliers`

Replace statistical outliers with model-imputed values.

```
$rows |> ai.clamp_outliers(
  columns: ["unitPrice"],
  model:   "logistics.models.tabpfn-v2:2.1.0",
  sigma:   3.0
)
```

---

## Probabilistic UDM nodes

UTL-X Infer extends the Universal Data Model with three new node types.

### ProbabilisticCell

A scalar value annotated with inference metadata. In most expressions it
behaves as its `.value` — backward compatible with 1.x expressions. Metadata
is accessible via the `._` prefix convention:

```
$row.unitPrice                  // resolves to .value
$row.unitPrice._confidence      // 0.0–1.0
$row.unitPrice._source          // "observed" | "imputed" | "inferred" | "classified"
$row.unitPrice._model_ref       // "logistics.models.tabpfn-v2:2.1.0"
```

### ColumnAnnotation

Semantic type per column — what the column means, not just its structural type.
Accessible as `$input._schema.columns`:

```
$input._schema.columns
  |> filter(col => col.semantic == "currency_amount")
  |> map(col => col.name)
// → ["unitPrice", "lineTotal", "taxAmount"]
```

### TableMeta

Table-level metadata set by `ai.*` functions. Accessible as `$input._meta`:

```
$input._meta.quality_score    // 0.0–1.0 — set by ai.score()
$input._meta.row_count        // total rows in the table
$input._meta.anomaly_count    // count of flagged rows — set by ai.anomaly()
$input._meta.model_versions   // audit trail of all models applied
```

---

## Script header — `%utlx 2.0`

```
%utlx 2.0
input:  rows csv
output: json
mode:   component
model:  logistics.models.tabpfn-v2:2.1.0
---
```

| Field | Values | Notes |
|---|---|---|
| `%utlx` | `2.0` | Required. Engine rejects version mismatches immediately. |
| `input:` | `alias format[, alias format]*` | Unchanged from 1.x. |
| `output` | format token | Unchanged from 1.x. |
| `mode:` | `inline` \| `ref` \| `component` | Default `inline`. `ai.*` requires `component`. |
| `model:` | `namespace.models.name:version` | Required when `ai.*` functions are used. |

### mode: component

Scripts declaring `mode: component` are not required to be pure or
deterministic. They run in a dedicated Open-M mapping component pod — never
inline in the receiving wrapper. `ai.*` functions are a parse-time error
outside `mode: component`.

### Backward compatibility

All `%utlx 1.0` and `%utlx 1.1` scripts run unchanged on the UTL-X Infer
engine. The version gate is enforced by the parser before any execution:

| Script | utl-x engine | utl-x-infer engine |
|---|---|---|
| `%utlx 1.0` | ✓ runs | ✓ runs — full 1.0 contract |
| `%utlx 1.1` | ✓ runs | ✓ runs — full 1.1 contract |
| `%utlx 2.0` | ✗ rejected with informative error | ✓ runs |

---

## Installation

### Requirements

- JVM 17 or later
- Kotlin 1.9 or later
- `utl-x` v1.3.0 or later (pulled automatically via Gradle)
- Open-M Model Registry (for `ai.*` function execution at runtime)
- NVIDIA GPU optional — most TFM models run on CPU with reduced throughput

### Gradle

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.github.grauwen:utl-x-infer:1.0.0-SNAPSHOT")
}
```

### From source

```bash
git clone https://github.com/grauwen/utl-x-infer.git
cd utl-x-infer
./gradlew build
./gradlew test
```

---

## Usage

### CLI

```bash
# Validate a %utlx 2.0 script
utlx-infer validate my-mapping.utlx

# Run a script against input data
utlx-infer run my-mapping.utlx \
  --input    supplier-feed.csv \
  --schema   logistics.schemas.supplier-feed:2.0.0 \
  --registry http://model-registry.open-m.svc:8080

# Run the conformance suite
utlx-infer test ./conformance/

# Format a script
utlx-infer format my-mapping.utlx
```

### Kotlin API

```kotlin
import com.github.grauwen.utlxinfer.UtlxInferEngine
import com.github.grauwen.utlxinfer.registry.ModelRegistry

// Connect to the Model Registry
val registry = ModelRegistry.connect("http://model-registry.open-m.svc:8080")

// Create the 2.0 engine
val engine = UtlxInferEngine.builder()
    .modelRegistry(registry)
    .build()

// Compile a mapping script
val mapping = engine.compile("""
    %utlx 2.0
    input:  rows csv
    output: json
    mode:   component
    model:  logistics.models.tabpfn-v2:2.1.0
    ---
    ${'$'}rows
      |> ai.impute(columns: ["unitPrice"], model: "logistics.models.tabpfn-v2:2.1.0")
      |> filter(row => row.unitPrice._confidence >= 0.90)
      |> map(row => { sku: row.productCode, price: row.unitPrice })
""".trimIndent())

// Execute against input data
val result = mapping.execute(
    UtlxInferInput(
        alias  = "rows",
        format = "csv",
        data   = csvBytes,
        schema = "logistics.schemas.supplier-feed:2.0.0"
    )
)
```

### Java API

UTL-X Infer is fully interoperable with Java:

```java
ModelRegistry registry = ModelRegistry.connect("http://model-registry.open-m.svc:8080");

UtlxInferEngine engine = UtlxInferEngine.builder()
    .modelRegistry(registry)
    .build();

CompiledMapping mapping = engine.compile(script);
UtlxInferResult result  = mapping.execute(input);
```

---

## Model Registry

The Model Registry is the authoritative catalogue of TFM model references
resolved by `ai.*` functions at runtime. It is part of the Open-M platform.

### Model ref format

```
{namespace}.models.{name}:{version}

# Examples
logistics.models.tabpfn-v2:2.1.0
logistics.models.quality-scorer-v1:1.0.0
open-m.models.tabpfn-prior-labs:2.1.0     # platform-bundled model
```

### Registering a model

```bash
open-m model register \
  --ref     logistics.models.tabpfn-v2:2.1.0 \
  --file    ./models/tabpfn-v2-registry.yaml \
  --weights s3://open-m-models/tabpfn-v2/2.1.0/weights.pt \
  --env     production
```

### Testing a model locally

```bash
utlx-infer model test \
  --ref    logistics.models.tabpfn-v2:2.1.0 \
  --input  ./test-data/supplier-feed-sample.csv \
  --schema logistics.schemas.supplier-feed:2.0.0
```

---

## Open-M integration

In Open-M pipelines, UTL-X Infer scripts run as **Mode 3 mapping components**
— dedicated pods with their own resource allocation, error port, retry
configuration, and log topic. They never run inline in the receiving wrapper.

The Open-M wrapper is written in Go and imports the Go FEEL engine for
routing. It does **not** import `utl-x-infer`. Only the dedicated inference
mapping component pod — a separate JVM process — imports and runs the UTL-X
Infer engine.

```yaml
# Pipeline YAML — Mode 3 inference mapping component
components:
  - id:  tabular-quality-gate
    ref:  open-m.components.utlx-infer-mapper:1.0.0
    config:
      mapping_ref:  logistics.mappings.supplier-quality-gate:1.0.0
      utlx_version: "2.0"
      mode:         component
    ports:
      input:
        schema_ref: logistics.schemas.supplier-feed:2.0.0
        format:     CSV
      output:
        schema_ref: logistics.schemas.supplier-feed-clean:1.0.0
        format:     JSON
      error:
        schema_ref: logistics.schemas.supplier-feed-rejected:1.0.0
    resources:
      requests:
        memory:          "8Gi"
        cpu:             "4"
      limits:
        nvidia.com/gpu:  "1"
    placement:
      cluster-ref:   k8s-prod-eu-west
      node_selector:
        accelerator: nvidia-t4
    logging:
      topic: logistics.orders.supplier-pipeline.tabular-quality-gate.log
```

---

## Conformance

### Layer 1 — UTL-X 1.x backward compatibility

The full UTL-X conformance suite is imported and executed against the UTL-X
Infer engine on every build. Backward compatibility is enforced as code:

```kotlin
// conformance/src/test/kotlin/BackwardCompatTest.kt
import com.github.grauwen.utlx.conformance.UtlxConformanceSuite

class BackwardCompatTest {
    @Test
    fun `utl-x-infer engine passes all 1x conformance tests`() {
        val engine = UtlxInferEngine.builder().build()
        UtlxConformanceSuite.runAll(engine)   // all 465+ tests must pass
    }
}
```

```bash
./gradlew test --tests "BackwardCompatTest"
```

### Layer 2 — UTL-X 2.0 coverage

Tests covering `%utlx 2.0` scripts — `ai.*` functions, `ProbabilisticCell`
node semantics, `mode: component` enforcement, and Model Registry integration:

```bash
./gradlew test --tests "conformance.v2.*"
```

### Full suite

```bash
./gradlew test
```

All tests in both layers must pass before a release is tagged.

---

## Status

| Component | Status |
|---|---|
| `%utlx 2.0` language specification | 🟡 Design phase |
| UDM 2.0 node types | 🟡 Design phase |
| `ai.*` stdlib function signatures | 🟡 Design phase |
| Parser extension for 2.0 header fields | 🔴 Not started |
| `ProbabilisticCell` evaluator | 🔴 Not started |
| TabPFN v2 integration via DJL + ONNX | 🔴 Not started |
| Model Registry client | 🔴 Not started |
| Backward compat conformance harness | 🔴 Not started |
| 2.0 conformance suite | 🔴 Not started |
| LSP extension for `%utlx 2.0` | 🔴 Not started |
| CLI (`utlx-infer`) | 🔴 Not started |
| Open-M mapper component | 🔴 Not started |

**Current phase: design stabilisation.**

Design documents in [`docs/proposals/`](docs/proposals/) define the intended
behaviour. Implementation begins once the specification is stable and reviewed.
Do not take a production dependency on this module.

### Prerequisites before v1.0.0

- [ ] UDM 2.0 node types reviewed and finalised
- [ ] `ai.*` stdlib function signatures stable — no breaking changes after this point
- [ ] Model Registry client interface designed
- [ ] TabPFN v2 proof of concept running end-to-end via DJL + ONNX
- [ ] `%utlx 2.0` parser passing basic test scripts
- [ ] Backward compatibility suite passing against `utl-x` v1.3.0
- [ ] Minimum one conformance test per `ai.*` function

---

## Contributing

The most valuable contributions during the design phase are:

- **Design review** — read [`docs/proposals/`](docs/proposals/) and open an
  issue with questions, corrections, or alternative approaches
- **TFM expertise** — experience with TabPFN, DJL, ONNX Runtime, or other
  inference runtimes — open an issue describing what the integration should
  look like from an API and runtime perspective
- **Use case documentation** — real-world scenarios where probabilistic mapping
  would replace brittle hand-coded validation rules in production pipelines

When implementation begins, contributions follow the same process as
[UTL-X](https://github.com/grauwen/utl-x/blob/main/CONTRIBUTING.md):

1. Open an issue before starting significant work
2. Fork the repo and create a branch from `main`
3. Write tests — every new `ai.*` function needs conformance tests in both
   the function-level and the backward compatibility layer
4. Run `./gradlew test` — all tests including the imported 1.x suite must pass
5. Submit a pull request with a clear description of what changes and why

### Design documents

All proposals live in [`docs/proposals/`](docs/proposals/):

| Document | Description |
|---|---|
| [`utlx-2-tabular-foundation-models.md`](docs/proposals/utlx-2-tabular-foundation-models.md) | Full 2.0 language specification — UDM extensions, `ai.*` stdlib, execution model |
| [`utlx-language-versioning-validation.md`](docs/proposals/utlx-language-versioning-validation.md) | Version model, `validate.*` namespace, engine compatibility matrix |
| [`utlx-repository-strategy.md`](docs/proposals/utlx-repository-strategy.md) | Why two repos, naming rationale, shared component strategy |

---

## License

UTL-X Infer is licensed under the **GNU Affero General Public License v3.0
(AGPL-3.0)**.

```
Copyright (C) 2026 grauwen

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Affero General Public License as published
by the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU Affero General Public License for more details.
```

Full licence text: [`LICENSE`](LICENSE).

### What AGPL v3 means in practice

| Use case | Requirement |
|---|---|
| Internal use in your own infrastructure | Free — no conditions |
| Distributing software that includes UTL-X Infer | Source must be available under AGPL v3 |
| Offering UTL-X Infer as a hosted / SaaS service | Source must be available to your users under AGPL v3 |
| Embedding in a commercial product without source disclosure | Commercial licence required |

For commercial licensing enquiries contact [grauwen](https://github.com/grauwen).

AGPL v3 is the same licence used by [UTL-X](https://github.com/grauwen/utl-x)
and [Open-M](https://github.com/grauwen/open-m). The licence is consistent
across the entire platform.

---

## Related projects

| Project | Description |
|---|---|
| [UTL-X](https://github.com/grauwen/utl-x) | Pure functional transformation language — the Kotlin foundation this project builds on |
| [Open-M](https://github.com/grauwen/open-m) | Enterprise middleware platform — UTL-X Infer runs as Open-M Mode 3 mapping components |
| [Prior Labs TabPFN](https://github.com/PriorLabs/TabPFN) | Primary TFM integration target — zero-shot tabular imputation and classification |
| [Deep Java Library](https://github.com/deepjavalibrary/djl) | JVM inference runtime — loads and runs ONNX and PyTorch models from Kotlin/Java |
