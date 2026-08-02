# UTL-X 2.0 — Tabular Foundation Models

## Status: Design Proposal

| Field | Value |
|---|---|
| Document | UTL-X-2-tabular-foundation-models |
| Status | Proposal — under discussion |
| UTL-X version | 2.0 (new major version) |
| Depends on | UTL-X 1.0, UDM 1.0, TSCH 1.0, Open-M MPPM Envelope |
| Related | open-m-arrow-mapping-utlx.md, open-m-utlx-ninput-stepwindow.md |

---

## 1. Motivation

Tabular Foundation Models (TFMs) — pre-trained models that operate on structured
tabular data zero-shot or few-shot — are becoming a practical tool for data
quality, anomaly detection, missing value imputation, and schema inference in
enterprise integration pipelines.

Prior Labs' TabPFN v2 is the leading example: a model that can classify, regress,
impute, and score tabular datasets without task-specific training, trained on
synthetic prior data to generalise across domains.

Enterprise integration pipelines handle tabular data constantly:

- EDIFACT and X12 EDI feeds — thousands of rows, inconsistent field population
- CSV supplier feeds — missing values, type inconsistencies, outliers
- SAP batch extracts — fixed-width or CSV, COBOL-heritage types
- Workday reports — tabular exports with mixed semantic column types

Today, handling data quality in these feeds requires hand-crafted validation
rules that are brittle, hard to maintain, and domain-specific. A TFM can replace
a large class of these rules with a single inference call — and do it better,
because the model learns from the statistical distribution of the data rather
than from explicitly coded thresholds.

The question is: **where does TFM capability belong in the UTL-X and Open-M
architecture?**

This document proposes the answer: **UTL-X 2.0** — a new major version of the
language that extends the UDM with probabilistic node types and introduces an
`ai.*` standard library namespace, gated behind a new `mode: component`
execution contract.

---

## 2. Why a new major version?

UTL-X 1.0 is defined by four guarantees:

| Guarantee | Definition |
|---|---|
| **Pure** | Same input always produces same output |
| **Stateless** | No external dependencies, no side effects |
| **Deterministic** | No probability, no uncertainty in output values |
| **Single-pass** | One input (or N named inputs) → one output |

TFM support breaks three of these four guarantees:

- **Not pure** — inference output depends on model weights, runtime environment,
  and model version, not only on the input data
- **Not stateless** — model weights are state, loaded at pod startup and held
  in memory during execution
- **Not deterministic** — probabilistic output means the same row can produce
  different imputed values on different calls, or different confidence scores
  across engine versions

These are not cosmetic changes. Purity and determinism are what make UTL-X 1.0
safe to run **inline on the connection arrow** in the receiving component's
wrapper — in-process, without a dedicated pod, without GPU resources, without
error isolation. Allowing `ai.*` calls in a Mode 2 inline mapping would
introduce GPU dependencies, model loading latency, and non-determinism into what
is supposed to be a lightweight, synchronous field transformation.

### 1.1 vs 2.0

Semver minor (1.1) signals: new capabilities, backward compatible. A 1.0 script
runs unchanged on a 1.1 engine.

That is achievable for additive features — new stdlib functions, new output
formats, new header options that a 1.0 engine can safely ignore.

TFM support requires:

- **New UDM node semantics** — probabilistic cells are a new node type, not a
  new function. A 1.0 engine does not know what a probabilistic node is and
  cannot safely ignore it.
- **Breaking the purity contract** — the definitional property of the language
- **New execution model** — `mode: component` changes when, where, and how a
  script is allowed to run
- **New header declarations** — `mode:` and `model:` are not 1.0 header fields

A 1.0 engine encountering a 2.0 script with `ai.impute()` must **fail loudly**,
not silently produce wrong results. That requires a major version signal —
`%utlx 2.0` in the script header.

**The verdict: 2.0.**

### Backward compatibility

Every existing UTL-X 1.0 script continues to run unchanged on a 2.0 engine.
The version declaration in the script header is the gate:

```
%utlx 1.0   →  engine applies full 1.0 contract, ai.* is unavailable
%utlx 2.0   →  engine applies 2.0 contract, ai.* available in mode: component
```

A 2.0 script without a `mode:` declaration defaults to `mode: inline` — full
1.0 contract preserved. Existing mappings need no changes.

---

## 3. The UDM in UTL-X 1.0

The **Universal Data Model** is the core abstraction of UTL-X. All inputs —
XML, JSON, CSV, YAML — are parsed into a format-neutral internal tree before
any mapping expression executes. The mapping language operates on UDM nodes,
not on raw bytes. The output UDM is then serialised to the declared output
format.

This is why `$input.Order.Customer.Name` works on both XML and JSON: both have
been lifted into the same UDM representation.

**UDM 1.0 node types:**

```
UDM Node (1.0)
├── Scalar
│   ├── StringValue
│   ├── NumberValue
│   ├── BooleanValue
│   └── NullValue
├── Composite
│   ├── ObjectNode    (named children — JSON object, XML element with children)
│   └── ArrayNode     (indexed children — JSON array, XML repeating elements)
└── Metadata
    ├── AttributeNode  (XML @attribute)
    └── NamespaceNode  (XML namespace declaration)
```

All nodes carry an observed, deterministic value. There is no concept of
uncertainty, probability, or inference provenance. A cell either has a value or
it is null.

---

## 4. UDM 2.0 — extensions for TFM support

UTL-X 2.0 extends the UDM with three new constructs:

### 4.1 ProbabilisticCell

A new scalar node variant that wraps an inferred or classified value with
metadata about how it was produced:

```
ProbabilisticCell {
  value:       <scalar — the inferred value, same type as the column>
  confidence:  <float 0.0–1.0 — model confidence in this value>
  source:      observed | imputed | inferred | classified
  model_ref:   <model registry ref — which model produced this cell>
}
```

In most mapping expressions, a `ProbabilisticCell` behaves as its `.value` —
backward compatible with expressions that don't know about confidence scores.
The metadata is accessible explicitly via dot notation:

```
$row.unitPrice                   // resolves to .value — the imputed number
$row.unitPrice._confidence       // 0.0–1.0
$row.unitPrice._source           // "imputed"
$row.unitPrice._model_ref        // "logistics.models.tabpfn-v2:2.1.0"
```

The `_` prefix on metadata properties is a 2.0 convention — reserved for
platform-injected metadata, never present in observed data.

### 4.2 ColumnAnnotation

A new metadata node attached to array-of-objects (table) UDM nodes, describing
the semantic type of each column — not just the structural type:

```
ColumnAnnotation {
  name:             "unitPrice"
  structural_type:  number
  semantic:         currency_amount       // new in UDM 2.0
  sub_properties: {
    currency:       "EUR"
  }
  nullable:         true
  observed_range:   { min: 0.01, max: 99999.99 }   // learned from data
}
```

Semantic types are drawn from a controlled vocabulary defined in TSCH 2.0
(see section 6). They give TFMs the signal needed to understand what a column
represents, not just its data type.

Accessible in mapping as `$input._schema.columns`:

```
$input._schema.columns
  |> filter(col => col.semantic == "currency_amount")
  |> map(col => col.name)
// → ["unitPrice", "lineTotal", "taxAmount"]
```

### 4.3 TableMeta

A new metadata node attached to the root of a tabular UDM structure:

```
TableMeta {
  row_count:      integer
  column_count:   integer
  schema_ref:     "logistics.schemas.supplier-feed:2.0.0"
  quality_score:  float              // set by ai.score()
  anomaly_count:  integer            // set by ai.anomaly()
  model_versions: [                  // audit trail of models applied
    { model_ref: "...", applied_at: "2026-04-14T09:23:01Z" }
  ]
}
```

Accessible as `$input._meta`:

```
$input._meta.quality_score         // 0.0–1.0
$input._meta.row_count             // 9847
$input._meta.anomaly_count         // 3
```

---

## 5. UTL-X 2.0 — language changes

### 5.1 New header fields

```
%utlx 2.0
input:  rows csv
output: json
mode:   component                  // new — required for ai.* functions
model:  logistics.models.tabpfn-quality-scorer:2.1.0   // new — model ref
---
```

| Header field | 1.0 | 2.0 | Notes |
|---|---|---|---|
| `%utlx` | `1.0` | `1.0` or `2.0` | Version gate |
| `input:` | Required | Required | Unchanged |
| `output` | Required | Required | Unchanged |
| `mode:` | Not present | `inline` / `ref` / `component` | New — defaults to `inline` |
| `model:` | Not present | Model registry ref | New — required when `ai.*` is used |

### 5.2 Execution mode contract

| | UTL-X 1.0 | UTL-X 2.0 mode: inline | UTL-X 2.0 mode: component |
|---|---|---|---|
| Purity required | ✓ | ✓ | ✗ |
| Stateless required | ✓ | ✓ | ✗ |
| Deterministic | ✓ | ✓ | ✗ |
| `ai.*` stdlib | ✗ | ✗ | ✓ |
| ProbabilisticCell nodes | ✗ | passthrough only | ✓ full read/write |
| Runs in receiving wrapper | ✓ | ✓ | ✗ |
| Dedicated pod | ✗ | ✗ | ✓ |
| GPU nodeSelector | ✗ | ✗ | ✓ |
| Error port | ✗ | ✗ | ✓ |

`mode: inline` and `mode: ref` in UTL-X 2.0 preserve the complete 1.0 contract.
All new capabilities are gated behind `mode: component`. The parser rejects
`ai.*` calls in any script that does not declare `mode: component` — this is a
hard parse-time error, not a runtime error.

### 5.3 The `ai.*` standard library namespace

All `ai.*` functions operate on UDM array-of-objects (table) nodes and return
modified versions of the same table with probabilistic cells where values were
inferred or transformed.

```
// Missing value imputation
ai.impute(columns: ["unitPrice", "taxClass"], model: "tabpfn-v2")

// Row classification — adds a new column with classified value
ai.classify(target_column: "recordType", model: "record-classifier-v1")

// Data quality scoring — sets $input._meta.quality_score
ai.score(model: "quality-scorer-v1", threshold: 0.85)

// Anomaly detection — marks anomalous rows, sets $input._meta.anomaly_count
ai.anomaly(model: "anomaly-detector-v1", sensitivity: 0.95)

// Schema inference — populates ColumnAnnotation semantic fields
ai.infer_schema(model: "schema-inferrer-v1")

// Outlier clamping — replaces statistical outliers with imputed values
ai.clamp_outliers(columns: ["unitPrice"], model: "tabpfn-v2", sigma: 3.0)
```

All `ai.*` functions:
- Are **chained via the pipe operator** like all UTL-X stdlib functions
- Return the **same UDM table** with affected cells replaced by `ProbabilisticCell`
  nodes
- Record their `model_ref` and `applied_at` in `$input._meta.model_versions`
- Require `mode: component` — parse-time error otherwise
- Resolve model references via the **Model Registry**

### 5.4 Full UTL-X 2.0 script example

```
%utlx 2.0
input:  rows csv
output: json
mode:   component
model:  logistics.models.tabpfn-supplier-quality:2.1.0
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
  |> ai.anomaly(
       model:       "logistics.models.anomaly-detector-v1:1.0.0",
       sensitivity: 0.95
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

### 5.5 Mixing UTL-X 1.0 and 2.0 in the same pipeline

A pipeline YAML can contain both 1.0 and 2.0 mappings. The version is declared
per script, not per pipeline. Inline Mode 2 mappings remain 1.0. Only Mode 3
mapping components that invoke TFM functions declare 2.0.

```yaml
connections:
  # Mode 2 inline — UTL-X 1.0, pure, stateless
  - id: conn-inbound-to-quality-gate
    transform:
      type: utlx
      mode: inline
      mapping: |
        %utlx 1.0
        input: current json
        output json
        ---
        { rows: $current.payload, source: $current.header.origin }

  # Mode 3 component — UTL-X 2.0, TFM inference
  - id: conn-quality-gate-to-erp
    from: { component: tabular-quality-gate, port: output }
    to:   { component: erp-loader, port: input }
```

---

## 6. TSCH 2.0 — structural + semantic schema

TSCH 1.0 describes the **structure** of tabular data: column names, data types,
separators, quoting, and positional field definitions for fixed-width formats.

TSCH 2.0 adds an optional **semantic layer** — column annotations that describe
what a column means, not just what type it holds. These annotations guide TFM
inference and are surfaced in the UTL-X 2.0 UDM as `ColumnAnnotation` nodes.

```yaml
# TSCH 2.0 — structural + semantic
apiVersion: open-m/v1
kind: Schema
metadata:
  ref:  logistics.schemas.supplier-feed:2.0.0
  type: TSCH
  tsch_version: "2.0"

structure:
  separator:   ","
  quote_char:  "\""
  has_header:  true
  encoding:    UTF-8

columns:
  - name:     productCode
    type:     string
    nullable: false
    semantic: product_identifier       # new in TSCH 2.0
    sub_properties:
      identifier_space: EAN-13

  - name:     unitPrice
    type:     number
    nullable: true                     # TFM will impute missing values
    semantic: currency_amount
    sub_properties:
      currency: EUR
      precision: 2

  - name:     shipCountry
    type:     string
    nullable: true
    semantic: geographic_region
    sub_properties:
      region_standard: ISO-3166-1

  - name:     deliveryDate
    type:     string
    nullable: true
    semantic: date
    sub_properties:
      format: yyyyMMdd

  - name:     taxClass
    type:     string
    nullable: true
    semantic: classification_code
    sub_properties:
      codelist: EU-VAT-CLASS
```

**TSCH 2.0 semantic vocabulary (initial):**

| Semantic | Description |
|---|---|
| `currency_amount` | Numeric monetary value |
| `product_identifier` | Product code, SKU, EAN, GTIN |
| `geographic_region` | Country, region, postal code |
| `date` | Calendar date in declared format |
| `datetime` | Date and time |
| `classification_code` | Controlled vocabulary code |
| `organisation_identifier` | Company, VAT, DUNS, GLN |
| `person_identifier` | Employee ID, customer ID |
| `quantity` | Count or measure with unit |
| `free_text` | Unstructured string — no semantic inference |

TSCH 2.0 is **backward compatible** with TSCH 1.0. The semantic fields are
optional. A UTL-X 1.0 engine reading a TSCH 2.0 schema ignores the semantic
layer. A UTL-X 2.0 engine uses it to guide TFM column-level inference.

---

## 7. The Model Registry

TFM support requires a new registry — the **Model Registry** — analogous in
design to the Schema Registry and Mapping Registry.

### Model ref format

```
{namespace}.models.{name}:{version}

# Examples
logistics.models.tabpfn-v2:2.1.0
logistics.models.quality-scorer-v1:1.0.0
logistics.models.anomaly-detector-v1:1.0.0
open-m.models.tabpfn-prior-labs:2.1.0    # platform-bundled model
```

### Model registry entry

```yaml
apiVersion: open-m/v1
kind: Model
metadata:
  ref:         logistics.models.tabpfn-v2:2.1.0
  name:        tabpfn-v2
  namespace:   logistics
  version:     2.1.0
  description: "Prior Labs TabPFN v2 — zero-shot tabular imputation and classification"
  source:      https://github.com/PriorLabs/TabPFN
  licence:     Apache-2.0

capabilities:
  - impute
  - classify
  - score
  - anomaly

resources:
  preferred:
    memory:          "8Gi"
    nvidia.com/gpu:  "1"
  minimum:
    memory:          "4Gi"
    cpu:             "4"

input_formats:
  - TSCH
  - JSCH     # array-of-objects JSON

compatibility:
  utlx_version: "2.0"
  tsch_version: "2.0"
```

### CLI

```bash
# Register a model
open-m model register \
  --ref     logistics.models.tabpfn-v2:2.1.0 \
  --file    ./models/tabpfn-v2-registry.yaml \
  --weights s3://open-m-models/tabpfn-v2/2.1.0/weights.pt \
  --env     production

# List models
open-m model list --namespace logistics

# Test a model against a sample file
open-m model test \
  --ref    logistics.models.tabpfn-v2:2.1.0 \
  --input  ./test-data/supplier-feed-sample.csv \
  --schema logistics.schemas.supplier-feed:2.0.0
```

---

## 8. Open-M pipeline YAML — Mode 3 TFM component

```yaml
spec:
  components:
    - id:  tabular-quality-gate
      ref:  open-m.components.utlx-mapper:2.0.0
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

## 9. Usability scenarios in Open-M pipelines

### Scenario 1 — Anomaly detection on EDIFACT inbound

```
[EDIFACT AS2 inbound]
      ↓  UTL-X 1.0 inline — TSCH → JSON tabular
[Tabular Quality Gate]     ← UTL-X 2.0 mode: component
      ↓ clean (confidence ≥ 0.90)     ↓ anomalous / low-confidence
[ERP loader]                      [Manual review queue]
```

The quality gate imputes missing values, scores rows, detects anomalies, and
routes clean rows forward. Anomalous rows carry their `_confidence` and
`_source` metadata into the review queue — the reviewer sees exactly which
cells were inferred and with what confidence.

### Scenario 2 — Missing value imputation before ERP load

Inbound supplier CSV consistently has missing `unitPrice`, `taxClass`, and
`shipCountry`. A TFM imputation component fills these before the UTL-X 1.0
mapping runs. The downstream mapper operates on a complete record and does not
need to handle null cases.

### Scenario 3 — Schema inference on unknown partner data

A new trading partner sends a CSV with no schema documentation. A TFM
`ai.infer_schema()` call populates `ColumnAnnotation` semantic fields. The
control plane writes a TSCH 2.0 draft schema to the Schema Registry for human
review. Once approved, the pipeline uses the registered schema for all
subsequent feeds from that partner.

### Scenario 4 — Classification routing on mixed-type flat files

A batch file contains interleaved orders, returns, and credit notes with no
explicit type field. A TFM `ai.classify()` call adds a `recordType` column.
Downstream connections route based on `recordType` value — no brittle regex,
no format-specific parsing rules.

### Scenario 5 — Data quality gate before master data load

Before loading to MDM or a data warehouse, every inbound record is scored by a
TFM quality model. Records below threshold 0.85 are held in a pending queue.
Records above threshold carry their `_meta.quality_score` in the MPPM envelope
payload — visible in the ops dashboard and in downstream audit logs.

---

## 10. Version compatibility matrix

| Component | Version | TFM support | Notes |
|---|---|---|---|
| UTL-X | 1.0 | ✗ | No `ai.*`, no probabilistic nodes |
| UTL-X | 2.0 mode: inline | ✗ | Full 1.0 contract preserved |
| UTL-X | 2.0 mode: component | ✓ | Full TFM support |
| UDM | 1.0 | ✗ | Value nodes only |
| UDM | 2.0 | ✓ | Adds ProbabilisticCell, ColumnAnnotation, TableMeta |
| TSCH | 1.0 | ~ | Structural schema, no semantic layer |
| TSCH | 2.0 | ✓ | Structural + semantic — guides TFM column inference |
| Schema Registry | 1.0 | ~ | Supports TSCH 1.0 |
| Schema Registry | 1.1 | ✓ | Adds TSCH 2.0 support (minor — backward compatible) |
| Mapping Registry | 1.0 | ~ | Stores UTL-X 1.0 scripts |
| Mapping Registry | 1.1 | ✓ | Adds UTL-X 2.0 scripts, tagged by version |
| Model Registry | — | ✓ | New — models, weights, capabilities, resource requirements |
| Open-M wrapper | 1.x | ✗ | Executes UTL-X 1.0 inline only |
| Open-M wrapper | 2.0 | ✓ | Routes UTL-X 2.0 component scripts to dedicated pod |

---

## 11. Design principles preserved

| Principle | How preserved in UTL-X 2.0 |
|---|---|
| Mode 2 purity | `ai.*` is a hard parse error in `mode: inline` and `mode: ref` |
| Mode 2 stateless | No model weights loaded in receiving wrapper — ever |
| Backward compatibility | `%utlx 1.0` scripts run unchanged on 2.0 engine |
| Schema governance | TFM output cells carry `_model_ref` — full provenance in MPPM envelope |
| Pipeline YAML as canonical artefact | Model refs are declared in YAML, versioned, Git-diffable |
| No silent failures | 1.0 engine encountering 2.0 script header fails loudly at parse time |
| Separation of concerns | TFM logic lives in Mode 3 components — topology, mapping, and inference remain distinct |

---

## 12. Open questions

- **Confidence threshold as a pipeline YAML concern or a UTL-X concern?**
  Currently proposed as UTL-X (`filter(row => row.price._confidence >= 0.90)`).
  Could alternatively be a connection-level routing rule in the YAML.

- **Should TSCH 2.0 semantic vocabulary be open or closed?**
  A closed controlled vocabulary is safer for TFM generalisation. An open
  vocabulary is more flexible for domain-specific semantics. Possible answer:
  closed core vocabulary + extensible namespace (`custom.semantics.invoice_ref`).

- **Model Registry governance — who can register a model?**
  Analogous to Schema Registry: namespace-scoped ownership. The logistics team
  owns `logistics.models.*`. Platform-bundled models live in `open-m.models.*`.

- **Determinism for reproducibility — should model version pinning be mandatory?**
  Proposal: yes. A UTL-X 2.0 script that references a model without a version
  (`tabpfn-v2` instead of `tabpfn-v2:2.1.0`) is a validation warning at deploy
  time and a validation error in production environments.

- **GPU resource scheduling — should Open-M own this or delegate to KEDA?**
  Proposal: Open-M declares resource requirements in the component manifest.
  Kubernetes scheduling handles placement. KEDA handles scaling based on queue
  depth of the input topic.
