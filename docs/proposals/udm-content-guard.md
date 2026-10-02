# UDM Content Guard — content-inspection / CDR engine capability (post-1.0)

## Status: Design Document

| Field | Value |
|---|---|
| Document | udm-content-guard |
| Status | Exploratory design — not implemented |
| Priority | Medium (high strategic) — depends on 1.1 `validate.*`, BINF, and an assurance/accreditation investment; start low-assurance |
| Created | October 2026 |
| Language version | post-1.0 (uses `%utlx 1.1` `validate.*`) |
| Component | Engine — parsers (hardened profile), serializer (canonical profile), `validate.*` rule engine, UDM; runs as an Open-M `mode: component` pod. **Not** the IDE. |
| Depends on | **1.1 `validate.*`** (pure rules — *same-repo*, see `utlx-1_1-semantic-validation.md`), Message Contract mode (allow-list), BINF (binary inputs), `mode: component` isolation + dead-letter |
| Related | `utlx-1_1-semantic-validation.md`, `utlx-ecosystem-total-fit.md`, `utlx-mil/docs/UDM-content-guard.md` |

> **Repo placement.** This is **post-1.0** work — it builds on `%utlx 1.1` (`validate.*`).
> Per the repo governance, the **`utl-x` repo is kept as 1.0, stable / as-is**, and **all
> 1.1 (and 2.0) work lives separately** (here). So this capability note sits with the
> other post-1.0 language work, **not** in `utl-x`. The **defence / CDS application**
> (NATO ↔ civilian cross-domain guard, releasability rules, restricted-format packs) is
> the **`utlx-mil`** layer and is referenced below; this note tracks the generic,
> dual-use *engine capability*, of which the NATO guard is one headline application.

---

> **One line:** Because every format UTL-X reads normalises into one **UDM**, a single
> set of **pure, deterministic** inspection rules can police many message formats
> (JSON, XML, CSV, and binary via BINF) — a generic, dual-use **content-inspection /
> CDR** engine capability: **allow-list, fail-closed**, canonical re-serialize, not a
> WAF. Its headline application is a **cross-domain guard** between networks of
> different trust (e.g. NATO ↔ civilian) — detailed in `utlx-mil` (see below).

## Why it fits UTL-X

- **UDM normalisation** → one rule set across many formats (N formats → 1 pipeline).
- **`validate.*` (1.1)** → a pure/deterministic rule engine — exactly a guard's rule layer.
- **Message Contract mode** → allow-list schema conformance.
- **Purity contract** → a *security* guarantee, and the clean boundary: **`ai.*` (2.0)
  must never be in the guard path** (non-deterministic = unaccreditable). The 1.0/1.1
  (pure) vs 2.0 (impure) split *is* the guard / non-guard boundary.
- **`parse → UDM → canonical re-serialize`** is structurally **CDR** (normalise →
  inspect → rebuild clean).

## The hard part

- **The parser is attack surface #1** (bombs, XXE, BINF length-overflow, ReDoS,
  parser-differential). Memory-safe runtime bounds it to DoS not RCE; it still needs a
  **hardened, sandboxed, fail-closed guard profile** + heavy fuzzing.
- **Accreditation is the mountain** (Common Criteria / NATO SDIP / national). Start as a
  low-assurance **CDR / policy gateway** on public formats, then climb.

## Design docs (authoritative detail in `utlx-mil`)

- `utlx-mil/docs/UDM-content-guard.md` — CDS/CDR framing, threat model, pipeline,
  non-negotiables, roadmap.
- `utlx-mil/docs/guard-rule-library.md` — the `validate.*` rule catalogue (7 categories:
  structural caps, schema/field allow-list, value, cross-field, releasability, keyword).
- `utlx-mil/docs/guard-parser-profile.md` — hardened read profile (per-format limits) +
  canonical low-fidelity serializer (anti-round-trip, anti-smuggling).

## Scope note

Engine/platform capability, cross-cutting with `utlx-mil`. The **open-core** parts
(hardened parser profile, canonical serializer, `validate.*` guard rules on public
formats) carry no military restrictions; restricted-format guards are a closed-edition /
customer-pack concern (same governance as BINF military packs).

## Status checklist

| Item | Status |
|---|---|
| CDS/CDR framing + threat model | 🟡 designed (`utlx-mil/docs/UDM-content-guard.md`) |
| `validate.*` guard rule library | 🟡 designed (`utlx-mil/docs/guard-rule-library.md`) — pending final 1.1 API |
| Hardened parser profile (per format) | 🟡 designed (`utlx-mil/docs/guard-parser-profile.md`) |
| Canonical low-fidelity serializer | 🟡 designed |
| `validate.*` (1.1) engine | 🔴 proposed (prerequisite — `utlx-1_1-semantic-validation.md`) |
| Sandbox / `mode: component` isolation | 🔴 planned (Open-M) |
| Fuzzing / assurance harness | 🔴 not started |
| Accreditation strategy | 🔴 not started |

---

*Supersedes the draft that briefly lived at `utl-x/docs/features/EF23-…` — moved here
because it is post-1.0 (1.1-dependent) work and the `utl-x` repo is kept at 1.0.*
