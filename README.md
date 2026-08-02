# UTL-X Infer

**Probabilistic mapping extension for UTL-X — inference-capable values, `ai.*` stdlib, and `%utlx 2.0` engine**

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL_v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![Go](https://img.shields.io/badge/Go-1.22+-00ADD8.svg)](https://go.dev)
[![UTL-X](https://img.shields.io/badge/UTL--X-%E2%89%A5_v1.3.0-amber.svg)](https://github.com/grauwen/utl-x)
[![Status](https://img.shields.io/badge/Status-Experimental-orange.svg)]()

---

UTL-X Infer extends [UTL-X](https://github.com/grauwen/utl-x) with
**probabilistic value semantics** — the ability to work with values that
carry a confidence score, a source annotation, and a model provenance reference
alongside the value itself.

Where UTL-X 1.x transforms observed, deterministic data, UTL-X Infer adds
a second class of value: the **inferred value** — produced by a Tabular
Foundation Model, a statistical imputer, a classifier, or any probabilistic
inference engine registered in the Open-M Model Registry.

Scripts declare `%utlx 2.0` and gain access to the `ai.*` standard library
namespace. All `%utlx 1.0` and `%utlx 1.1` scripts run unchanged on the
UTL-X Infer engine — full backward compatibility is enforced by the imported
UTL-X conformance suite.

> **This project is experimental.** The `%utlx 2.0` language specification
> and the `ai.*` stdlib are under active design. Breaking changes will occur
> before the first stable release. See [Status](#status) below.

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
structural transformation.

Some integration scenarios require more. A supplier CSV feed where 15% of
`unitPrice` fields are missing. An inbound EDIFACT batch where row type must
be inferred because no explicit type field exists. A high-volume data pipeline
where statistical outliers need flagging before the data reaches the ERP.

These scenarios need values to be **inferred**, not just mapped. UTL-X Infer
provides this through:

| Capability | What it does |
|---|---|
| **Probabilistic cell values** | Scalar values that carry `._confidence`, `._source`, and `._model_ref` metadata alongside the value itself |
| **`ai.*` stdlib namespace** | Functions for imputation, classification, quality scoring, anomaly detection, and schema inference |
| **`mode: component` execution** | Scripts that declare `mode: component` are allowed to be non-pure and non-deterministic — they run in a dedicated pod, never inline in the wrapper |
| **Model Registry client** | Resolves model references (`namespace.models.name:version`) against the Open-M Model Registry |
| **UDM 2.0 node types** | `ProbabilisticCell`, `ColumnAnnotation`, `TableMeta` — extensions to the Universal Data Model |

---

## Relationship to UTL-X

UTL-X Infer is **not a fork**. It is a Go module that depends on
`github.com/grauwen/utl-x` and extends it. The parser, UDM 1.0 node types,
core stdlib, and `validate.*` namespace are imported from UTL-X — not
duplicated.

```
github.com/grauwen/utl-x-infer
    └── requires github.com/grauwen/utl-x ≥ v1.3.0
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
| TFM runtime bindings | `utl-x-infer` |

A project that imports `github.com/grauwen/utl-x` gets zero probabilistic
or TFM dependencies. Only projects that explicitly import
`github.com/grauwen/utl-x-infer` pull in the inference stack.

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

Compare the same pipeline without inference — using UTL-X 1.0, no special
engine required:

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

The 1.0 script runs on the base UTL-X engine and maps only what is present.
The 2.0 script runs on UTL-X Infer and fills what is missing before mapping.
Both are valid — the right choice depends on whether inference is needed.

---

## The `ai.*` stdlib

All `ai.*` functions operate on UDM array-of-objects (table) nodes. They
return modified versions of the same table with probabilistic cells where
values were inferred or transformed.

`ai.*` functions are a **parse-time error** in `%utlx 1.0` and `%utlx 1.1`
scripts, and in any `%utlx 2.0` script that does not declare `mode: component`.
This is enforced by the parser — not the runtime. A misconfigured script is
caught before execution begins.

### `ai.impute`

Fill missing values using a registered TFM model.

```
$rows |> ai.impute(
  columns: ["unitPrice", "taxClass"],
  model:   "logistics.models.tabpfn-v2:2.1.0"
)
```

Returns the table with affected cells replaced by `ProbabilisticCell` nodes.
The original value is preserved if present — only null/missing cells are filled.

### `ai.classify`

Add a classification column derived by a model.

```
$rows |> ai.classify(
  target_column: "recordType",
  model:         "logistics.models.record-classifier-v1:1.0.0"
)
```

Adds a new `recordType` column to every row. Each cell is a `ProbabilisticCell`
with the classified label and its confidence score.

### `ai.score`

Score the overall data quality of the table. Sets `$rows._meta.quality_score`.

```
$rows |> ai.score(
  model:     "logistics.models.quality-scorer-v1:1.0.0",
  threshold: 0.85
)
```

### `ai.anomaly`

Flag anomalous rows. Sets `row._anomaly = true` on outliers and
`$rows._meta.anomaly_count`.

```
$rows |> ai.anomaly(
  model:       "logistics.models.anomaly-detector-v1:1.0.0",
  sensitivity: 0.95
)
```

### `ai.infer_schema`

Populate `ColumnAnnotation` semantic fields — what each column means, not just
its type. Useful for schema inference on unknown partner data.

```
$rows |> ai.infer_schema(
  model: "open-m.models.schema-inferrer-v1:1.0.0"
)
```

### `ai.clamp_outliers`

Replace statistical outliers with imputed values.

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
behaves as its `.value` — backward compatible with 1.x expressions that do
not inspect metadata. Metadata is accessible explicitly via `._` prefix:

```
$row.unitPrice                  // resolves to .value — the inferred number
$row.unitPrice._confidence      // 0.0–1.0 — model confidence
$row.unitPrice._source          // "observed" | "imputed" | "inferred" | "classified"
$row.unitPrice._model_ref       // "logistics.models.tabpfn-v2:2.1.0"
```

### ColumnAnnotation

Semantic type metadata per column — what the column means, not just its type.
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
$input._meta.row_count        // total rows
$input._meta.anomaly_count    // set by ai.anomaly()
$input._meta.model_versions   // audit trail of models applied
```

---

## Script header — `%utlx 2.0`

Every UTL-X Infer script begins with the 2.0 header. New fields beyond 1.x:

```
%utlx 2.0
input:  rows csv                                        ← unchanged from 1.x
output: json                                            ← unchanged from 1.x
mode:   component                                       ← required for ai.*
model:  logistics.models.tabpfn-v2:2.1.0               ← required when ai.* used
---
```

| Field | Values | Notes |
|---|---|---|
| `%utlx` | `2.0` | Required. Engine rejects scripts with version mismatch. |
| `input:` | `alias format[, alias format]*` | Unchanged from 1.x. |
| `output` | format token | Unchanged from 1.x. |
| `mode:` | `inline` \| `ref` \| `component` | Default `inline`. `ai.*` requires `component`. |
| `model:` | `namespace.models.name:version` | Required when `ai.*` functions are used. |

### mode: component

Scripts declaring `mode: component` are **not required to be pure or
deterministic**. They run in a dedicated Open-M mapping component pod — never
inline in the receiving wrapper. The pod has its own error port, retry
configuration, resource allocation (memory, GPU), and log topic.

`ai.*` functions are a **parse-time error** outside `mode: component`.

### Backward compatibility

All `%utlx 1.0` and `%utlx 1.1` scripts run unchanged on the UTL-X Infer
engine. The version declaration in the header is the gate — the engine applies
the 1.x contract strictly to 1.x scripts.

```
%utlx 1.0 script on utl-x-infer engine  →  runs under full 1.0 contract
%utlx 1.1 script on utl-x-infer engine  →  runs under full 1.1 contract
%utlx 2.0 script on utl-x engine        →  rejected: informative error message
```

---

## Installation

### Requirements

- Go 1.22 or later
- `github.com/grauwen/utl-x` v1.3.0 or later
- Access to an Open-M Model Registry (for `ai.*` function execution)
- NVIDIA GPU optional — most TFM models run on CPU with reduced throughput

### Go module

```bash
go get github.com/grauwen/utl-x-infer
```

### CLI binary

```bash
go install github.com/grauwen/utl-x-infer/cmd/utlx-infer@latest
```

### From source

```bash
git clone https://github.com/grauwen/utl-x-infer.git
cd utl-x-infer
go build ./...
go test ./...
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

### Go API

```go
import (
    utlxinfer "github.com/grauwen/utl-x-infer"
    "github.com/grauwen/utl-x-infer/registry"
)

// Connect to the Model Registry
reg := registry.New("http://model-registry.open-m.svc:8080")

// Create a 2.0 engine
engine := utlxinfer.NewEngine(
    utlxinfer.WithModelRegistry(reg),
)

// Pre-compile a mapping script
mapping, err := engine.Compile(`
    %utlx 2.0
    input:  rows csv
    output: json
    mode:   component
    model:  logistics.models.tabpfn-v2:2.1.0
    ---
    $rows |> ai.impute(columns: ["unitPrice"], model: "logistics.models.tabpfn-v2:2.1.0")
          |> filter(row => row.unitPrice._confidence >= 0.90)
          |> map(row => { sku: row.productCode, price: row.unitPrice })
`)
if err != nil {
    log.Fatal(err)
}

// Execute against input
result, err := mapping.Execute(utlxinfer.Input{
    Alias:  "rows",
    Format: "csv",
    Data:   csvBytes,
    Schema: "logistics.schemas.supplier-feed:2.0.0",
})
```

---

## Model Registry

The Model Registry is the authoritative catalogue of TFM model references used
by `ai.*` stdlib functions. It is part of the Open-M platform — UTL-X Infer
connects to it at runtime to resolve model refs and load model weights.

### Model ref format

```
{namespace}.models.{name}:{version}

# Examples
logistics.models.tabpfn-v2:2.1.0
logistics.models.quality-scorer-v1:1.0.0
open-m.models.tabpfn-prior-labs:2.1.0     # platform-bundled model
```

### Registering a model (Open-M CLI)

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

```yaml
# pipeline YAML — Mode 3 inference mapping component
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
```

UTL-X 1.x Mode 2 inline mappings — the pure, stateless transforms on
connection arrows — use `github.com/grauwen/utl-x` directly. The Open-M
wrapper never imports `utl-x-infer`. Only the dedicated inference mapping
component pod does.

---

## Conformance

The UTL-X Infer conformance suite has two layers:

### Layer 1 — UTL-X 1.x backward compatibility

The full UTL-X conformance suite (465+ tests, all `%utlx 1.0` and `%utlx 1.1`)
is imported and run against the UTL-X Infer engine. Backward compatibility is
not a README promise — it is an enforced test requirement on every release:

```bash
go test ./conformance/backward_compat/...
```

### Layer 2 — UTL-X 2.0 coverage

New tests covering `%utlx 2.0` scripts — `ai.*` functions, `ProbabilisticCell`
node semantics, `mode: component` enforcement, and Model Registry integration:

```bash
go test ./conformance/v2/...
```

### Running the full suite

```bash
go test ./conformance/...
```

All tests in both layers must pass before a release is tagged.

---

## Status

| Component | Status |
|---|---|
| `%utlx 2.0` language specification | 🟡 Design phase |
| UDM 2.0 node types | 🟡 Design phase |
| `ai.*` stdlib function signatures | 🟡 Design phase |
| Parser extension for 2.0 header | 🔴 Not started |
| `ProbabilisticCell` evaluator | 🔴 Not started |
| TabPFN v2 integration | 🔴 Not started |
| Model Registry client | 🔴 Not started |
| Backward compat conformance suite | 🔴 Not started |
| 2.0 conformance suite | 🔴 Not started |
| LSP extension | 🔴 Not started |
| CLI (`utlx-infer`) | 🔴 Not started |
| Open-M mapper component | 🔴 Not started |

**Current phase: design stabilisation.**

The design documents in [`docs/proposals/`](docs/proposals/) define the
intended behaviour. Implementation begins once the specification is stable
and reviewed. Do not take a production dependency on this module yet.

### Prerequisites before v1.0.0

- [ ] UDM 2.0 node types reviewed and finalised
- [ ] `ai.*` stdlib function signatures stable
- [ ] Model Registry client interface designed
- [ ] TabPFN v2 proof of concept running end-to-end
- [ ] `%utlx 2.0` parser passing basic test scripts
- [ ] Backward compatibility suite passing against `utl-x` v1.3.0
- [ ] At least one conformance test per `ai.*` function

---

## Contributing

UTL-X Infer is in early design phase. The most valuable contributions right
now are:

- **Design review** — read the proposals in [`docs/proposals/`](docs/proposals/)
  and open an issue with questions, corrections, or alternative approaches
- **TFM expertise** — experience with TabPFN, ONNX, or other inference runtimes
  — open an issue describing your experience and what the integration should
  look like
- **Use case documentation** — real-world scenarios where probabilistic mapping
  would replace brittle hand-coded validation rules

When implementation begins, contributions follow the same process as
[UTL-X](https://github.com/grauwen/utl-x/blob/main/CONTRIBUTING.md):

1. Open an issue before starting significant work
2. Fork the repo and create a branch from `main`
3. Write tests — new `ai.*` functions need conformance tests
4. Ensure `go test ./conformance/...` passes — including the backward
   compatibility layer
5. Submit a pull request with a clear description of what changes and why

### Design documents

All design proposals live in [`docs/proposals/`](docs/proposals/). Current
proposals:

- [`utlx-2-tabular-foundation-models.md`](docs/proposals/utlx-2-tabular-foundation-models.md) — full 2.0 language spec
- [`utlx-language-versioning-validation.md`](docs/proposals/utlx-language-versioning-validation.md) — version model and validate.* namespace
- [`utlx-repository-strategy.md`](docs/proposals/utlx-repository-strategy.md) — why two repos, naming rationale

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

The full licence text is in [`LICENSE`](LICENSE).

### What AGPL v3 means in practice

- **Internal use:** Free. Use UTL-X Infer in your own infrastructure,
  pipelines, and integrations without restriction.
- **Distribution:** If you distribute software that includes UTL-X Infer,
  you must make the full source available under AGPL v3.
- **SaaS / hosted service:** If you offer UTL-X Infer as a service over a
  network, you must make the source available to your users under AGPL v3.
- **Commercial licence:** If AGPL v3 does not fit your use case, contact
  [grauwen](https://github.com/grauwen) to discuss a commercial licence.

AGPL v3 is the same licence used by [UTL-X](https://github.com/grauwen/utl-x)
and [Open-M](https://github.com/grauwen/open-m). The licence family is
consistent across the entire platform.

---

## Related projects

| Project | Description |
|---|---|
| [UTL-X](https://github.com/grauwen/utl-x) | Pure functional transformation language — the foundation this project builds on |
| [Open-M](https://github.com/grauwen/open-m) | Enterprise middleware platform — UTL-X Infer runs as Open-M Mode 3 mapping components |
| [Prior Labs TabPFN](https://github.com/PriorLabs/TabPFN) | The primary TFM integration target — zero-shot tabular imputation and classification |
