# Factlet Protocol — v0.1 Specification

**Status**: Draft
**Version**: v0.1
**Last updated**: 2026-05-04
**Editor**: Mihir Choudhary <mihir@kernora.ai>
**License**: MIT

This document specifies the Factlet Protocol v0.1, an open, vendor-neutral specification for grounding large language models in private, team-owned truth, with a runtime signal for when that grounding runs out.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 1. Motivation

Large language models trained on the public internet confidently answer questions about private things — your codebase, your team's decisions, your customer's data — and get them wrong. Each frontier vendor (Anthropic Memory, OpenAI Memory, Google Project Astra) ships its own per-vendor private-knowledge layer. Teams who use multiple tools today get fragmented across incompatible memory stores.

The Factlet Protocol defines five primitives that any tool can implement, so private context becomes portable across models and vendors, and so consumers get a runtime signal when the model is about to answer in a zone with no relevant grounding.

## 2. Terminology

The protocol introduces five named primitives. They cascade logically; each has exactly one job.

- **Factlet**: one atomic, source-cited truth about private information. The unit of cite, retrieve, supersede, and export.
- **FactMap**: a structured collection of factlets covering one body of work (a codebase, a product, a customer base). Implementations choose a structure that fits their domain.
- **Factbook**: a packaged FactMap, versioned and portable across implementations. The artifact a team owns.
- **FactSignal**: an integer score (0–5) indicating how strong the factlet coverage is for a given query, using the mobile-signal-bars metaphor.
- **Low-FactSignal warning**: a runtime callback that fires when a model is about to answer in a zone where FactSignal is below a threshold.

## 3. Factlet record format

A factlet **MUST** be representable as a YAML or JSON object with the following fields.

### 3.1 Required fields

| Field        | Type    | Description |
|--------------|---------|-------------|
| `id`         | string  | Stable identifier, unique within a Factbook. Conventionally `f###` (e.g. `f001`). MUST be immutable once published. **In v0.2 (per [RFC-001](rfcs/0001-scoped-fact-ids.md))**, external references MUST use the scoped form `<scope>:<id>` (e.g. `factlet-ai:f001`); bare IDs remain valid INSIDE the host Factbook because the file-level `scope:` field provides the prefix. |
| `statement`  | string  | The factlet text. SHOULD be a single declarative sentence. |
| `confidence` | float   | Producer's confidence the statement is true, in `[0.0, 1.0]`. |
| `sources`    | list[string] | One or more provenance pointers (file:line, commit hash, URL, document path, incident ID). MUST be non-empty. |

### 3.2 Optional fields

| Field             | Type           | Description |
|-------------------|----------------|-------------|
| `tags`            | list[string]   | Free-form tags for filtering and grouping. |
| `scope_level`     | string         | One of `team`, `project`, `feature`, `personal`, indicating the breadth of applicability. |
| `supersedes`      | list[string]   | IDs of older factlets this one replaces. The superseded factlets MUST be marked `archived: true`. |
| `merged_into`     | string         | If this factlet was merged into another, the target ID. |
| `archived`        | boolean        | If true, the factlet is no longer authoritative. Implementations MUST NOT include archived factlets in retrieval results unless explicitly requested. |
| `archived_reason` | string         | Free-form reason for archiving. |
| `retired_at`      | string (date)  | ISO 8601 timestamp when the factlet was archived. |
| `extension`       | object         | Vendor-specific fields. Implementations MUST ignore unknown keys. |
| `origination`     | object         | **v0.2 ([RFC 0003](rfcs/0003-origination-provenance-block.md))** — provenance of the factlet RECORD itself (distinct from `sources`, which provides provenance for the underlying CLAIM). Sub-fields: `source_type` (string; recommended values `manual` / `llm` / `import` / `forward-pass` / `reverse-pass`; profile-extensible — readers SHOULD warn on unknown values, not reject, per RFC 0013), `source_ref` (free-form string, max 256 chars), `authored_at` (ISO 8601), `authored_by` (free-form string, max 256 chars), `trust_prior` (float in `[0.0, 1.0]`). |
| `profile`         | string         | **v0.2 ([RFC 0002](rfcs/0002-profiles-mechanism.md))** — per-factlet profile override of the file-level profile. Used in cross-domain Factbooks. See §15. |
| `rationale`       | string         | **v0.2 ([RFC 0010](rfcs/0010-rationale-alternatives.md))** — why the claim holds or why this choice was made (the intent the source artifacts structurally discard). |
| `alternatives`    | list[object]   | **v0.2 ([RFC 0010](rfcs/0010-rationale-alternatives.md))** — options considered and rejected at decision time. Each entry: `summary` (REQUIRED), `rejected_because`, optional `ref` to a superseded factlet. Distinct from the supersession lineage (§15.5). |
| `attribution`     | object         | **v0.2 ([RFC 0011](rfcs/0011-attribution.md))** — splits `produced_by` (`agent` REQUIRED, `kind`) — the agent that generated the claim — from `authority` (`owner`, `approved_by`, `approved_at`, `review_status`) — the role accountable for it being true. |

### 3.3 Example

```yaml
- id: f002
  statement: "Refunds older than 90 days require manual ops approval, never auto-processed."
  confidence: 1.0
  sources:
    - "docs/refund-policy.md:8"
    - "incident:2026-01-15"
  tags: [payments, ops, compliance]
  scope_level: team
```

## 4. FactMap

A FactMap is a structured spatial organization of factlets. Implementations **MAY** organize a FactMap by directory, module, domain, customer, or any other axis appropriate to the body of work.

The contract is: given a query, the FactMap **MUST** be able to return:
1. A subset of factlets considered relevant to the query.
2. A structural region descriptor identifying where in the FactMap those factlets live.

The retrieval algorithm is implementation-defined. Embedding similarity, BM25, graph traversal, LLM-based scoring, or any combination is permitted.

## 5. Factbook container

A Factbook is a packaged Factbook **MUST** be a YAML or JSON document with the following structure.

### 5.1 Container fields

| Field            | Type             | Required | Description |
|------------------|------------------|----------|-------------|
| `schema_version`  | string            | Yes      | The Factbook protocol version. For this spec: `"v1.0"`. |
| `last_updated`    | string (ISO 8601) | No       | When the Factbook was last modified. |
| `metadata`        | object            | No       | Free-form Factbook-level metadata (name, owner, description). |
| `content`         | list[factlet]     | Yes      | The factlet records, conforming to §3. |
| `profile`         | string            | No       | **v0.2 ([RFC 0002](rfcs/0002-profiles-mechanism.md))** — registered profile this Factbook is authored against (e.g. `software-engineering`). See §15. |
| `profile_version` | string            | No       | **v0.2** — version of the profile schema this Factbook is authored against. |
| `dependencies`    | object            | No       | **v0.2 ([RFC 0004](rfcs/0004-composable-factbooks-via-dependencies.md))** — composable-factbooks declaration. See §5.4. |

### 5.2 Versioning

Factbooks **SHOULD** be versioned in git or a comparable source control system, so that the team owns the artifact and can audit history. Implementations **MUST NOT** require a centralized server for Factbook storage.

### 5.3 Example (minimal)

```yaml
schema_version: v1.0
last_updated: 2026-05-04T00:00:00Z
metadata:
  name: payments-service
  owner: payments-team
content:
  - id: f001
    statement: "We use Stripe webhooks (not polling) for payment status updates."
    confidence: 0.95
    sources: ["docs/payments-arch.md:42", "commit:abc123"]
    tags: [payments, architecture]
```

### 5.4 Composable factbooks (v0.2 — RFC 0004)

A Factbook MAY declare composition against other Factbooks via a top-level `dependencies.factbooks` list. Each entry has:

| Field            | Type   | Required | Description |
|------------------|--------|----------|-------------|
| `id`             | string | Yes      | Scoped identifier of the dependency (e.g. `best-practices:python-3.12-2026`). |
| `version`        | string | No       | Specific version. If absent, consumer SHOULD resolve to latest. |
| `source`         | string | No       | URL or path to fetch the dependency. If absent, consumer relies on a registry resolver. |
| `trust_prior`    | float  | No       | Override trust prior in `[0.0, 1.0]` for factlets pulled from this dependency. |
| `retrieval_mode` | string | No       | `merged` (default) / `fallback` / `disabled`. |

**Auto-fetch:** v0.2 readers SHOULD NOT auto-fetch dependencies from arbitrary URLs without explicit consumer consent (SSRF / supply-chain risk). Recommended pattern: a registered resolver the consumer configures explicitly. The reference SDK ships with no auto-fetch; consumers supply a resolver callback.

**Cycle detection:** consumers MUST detect transitive cycles in the dependency graph and error / warn rather than recurse unbounded.

## 6. FactSignal

FactSignal is a function with the signature:

```
factsignal(query: string, factmap: FactMap) -> int
```

The return value **MUST** be an integer in `[0, 5]`, with the following semantics:

| Bars | Meaning |
|------|---------|
| 5    | Dense coverage — multiple directly relevant factlets exist. |
| 4    | Good — at least one strong direct factlet plus adjacent context. |
| 3    | Mixed — partial coverage; some aspects covered, others not. |
| 2    | Sparse — only one tangentially relevant factlet. |
| 1    | Thin — only loosely related factlets. |
| 0    | Dead zone — no relevant factlets. |

The scoring algorithm is implementation-defined. Embedding-based scoring, keyword overlap, graph distance, or LLM-based scoring are all permitted. The contract is the bars output, not the algorithm.

Implementations **SHOULD** document their scoring methodology so consumers can calibrate expectations.

## 7. Low-FactSignal warning

The protocol mandates a runtime callback that fires when a model is about to answer in a zone with FactSignal below a configurable threshold.

### 7.1 Callback signature

```
on_low_factsignal(query: string, score: int, retrieved: list[factlet], threshold: int) -> void
```

### 7.2 Required behavior

- Implementations **MUST** invoke the callback before the model's answer is exposed to the user.
- The default threshold **SHOULD** be 2.
- The threshold **MUST** be configurable by the consuming agent.
- The callback receives the query, the score, the (possibly empty) retrieved factlet set, and the threshold value.

### 7.3 Policy is the agent's

The protocol mandates the signal; the policy is the consuming agent's. The callback **MAY** be implemented as any of:

- A user-facing banner ("Nora has no facts about X").
- A model escalation (route to a different model).
- A refusal to answer.
- A logging/telemetry sink.
- A combination of the above.

## 8. Vendor rendering

Factlets are stored canonically in YAML/JSON. They are rendered into model-vendor-specific formats at injection time.

| Vendor   | Recommended rendering |
|----------|------------------------|
| Anthropic Claude | Structured XML inside `system` block |
| OpenAI GPT       | Markdown bulleted list |
| Google Gemini    | `systemInstruction` field |
| Local / open-source | Plain text or whatever the model best ingests |

The rendering layer is a separate concern from the canonical store. Implementations **MUST NOT** require canonical storage in any vendor-specific format.

## 9. Conformance

A conforming implementation:

1. **MUST** parse Factbooks of the format defined in §5.
2. **MUST** preserve all required factlet fields (§3.1) on retrieval.
3. **MUST** ignore unknown keys in the `extension` object (§3.2) without error.
4. **MUST** return integer FactSignal scores in `[0, 5]` (§6).
5. **MUST** invoke the low-FactSignal callback before answer exposure when the score is below the configured threshold (§7).
6. **SHOULD** document its retrieval algorithm and FactSignal scoring methodology.
7. **SHOULD** version Factbooks in git or comparable source control.

A non-conforming implementation **MUST NOT** describe itself as "Factlet Protocol compatible."

## 10. Versioning policy

Pre-v1.0, breaking changes **MAY** occur in any minor version (e.g. v0.1 → v0.2). After v1.0, breaking changes **MUST** require a major version bump.

The current version is **v0.1**. v0.2 is targeted within 90 days of v0.1 publication, incorporating community feedback via the [RFC process](rfcs/).

## 11. Reference implementation

The canonical reference implementations are at [`factlet-ai/reference-sdk`](https://github.com/factlet-ai/reference-sdk) (Python + TypeScript). All conforming implementations **SHOULD** validate against the reference test fixtures.

## 12. Examples

See the [`examples/`](examples/) directory for complete Factbooks across multiple domains.

## 13. Open questions for v0.2

These were explicitly unresolved in v0.1; status updated as v0.2 RFCs land:

- ~~Cross-Factbook references (when one team's factlet should reference another team's).~~ **Resolved by [RFC 0004](rfcs/0004-composable-factbooks-via-dependencies.md).**
- ~~Domain-specific factlet vocabulary (software vs manufacturing vs healthcare).~~ **Resolved by [RFC 0002](rfcs/0002-profiles-mechanism.md) — Profiles mechanism.** First registered profile: software-engineering, see [RFC 0005](rfcs/0005-software-profile-phase-enum.md).
- ~~Provenance of the factlet record itself (vs the underlying claim).~~ **Resolved by [RFC 0003](rfcs/0003-origination-provenance-block.md) — Origination block.**
- Factlet revocation semantics (vs supersession). Open.
- Multi-tenancy model for shared Factbooks. Open.
- Streaming retrieval API for very large Factbooks. Open.
- Standard FactSignal scoring algorithm (currently implementation-defined; may benchmark and recommend). Open.

To propose an RFC, see [CONTRIBUTING.md](CONTRIBUTING.md) and the [RFC template](rfcs/0000-template.md).

## 14. Acknowledgments

This protocol borrows the open-spec / reference-implementation pattern from the Model Context Protocol (MCP). Naming conventions for `factlet` and `FactSignal` were locked after a vocabulary review against existing terms in the AI grounding literature.

## 15. Profiles (v0.2)

The Profiles mechanism (ratified by [RFC 0002](rfcs/0002-profiles-mechanism.md)) is the canonical way to extend the protocol with domain-specific vocabulary without bloating the base spec. A Factbook MAY declare a `profile` at the file level (and individual factlets MAY override at record level). Each registered profile maintains its own schema extensions in `profiles/<name>/`.

### 15.1 File-level fields

A Factbook MAY include:

| Field             | Type   | Description |
|-------------------|--------|-------------|
| `profile`         | string | Identifier of the registered profile this Factbook is authored against (e.g. `software-engineering`). |
| `profile_version` | string | Version of the profile schema this Factbook is authored against. |

If `profile` is omitted, the Factbook is interpreted as **profile-neutral** — only base-spec fields apply.

### 15.2 Per-factlet override

A factlet MAY include `profile: <name>` to override the file-level profile. Used for cross-domain Factbooks (e.g. an industrial-software project mixing `software-engineering` + `manufacturing` factlets).

### 15.3 Conformance

A v0.2 reader:
1. MUST parse the `profile` and `profile_version` fields without error.
2. MUST parse the per-factlet `profile` override.
3. SHOULD apply profile-specific schema validation when it knows the named profile (its schema is installed or available).
4. MUST treat profile-specific fields as unknown keys when it does not know the named profile (preserve on round-trip per v0.1 §9.3) and SHOULD emit a warning identifying the unknown profile.
5. MUST NOT reject a Factbook on the basis of profile-specific fields the reader does not recognize.

### 15.4 Registry

Registered profiles are listed in [`profiles/REGISTRY.md`](profiles/REGISTRY.md). Each profile lives at `profiles/<name>/` with a `SPEC.md`, `factlet.schema.json`, optional `factbook.schema.json`, and `examples/`.

Adding a profile requires a sub-RFC; see [`profiles/.template/`](profiles/.template/) for the starter template.

### 15.5 Other v0.2 base-spec extensions

In addition to Profiles, v0.2 ratifies these universal (base-spec) extensions:
- **`origination`** block on factlets (RFC 0003) — provenance of the YAML record itself, distinct from `sources` (provenance of the claim). See §3.2.
- **`dependencies.factbooks`** at Factbook root (RFC 0004) — composition against other Factbooks. See §5.4.
- **`rationale`** + **`alternatives`** on factlets ([RFC 0010](rfcs/0010-rationale-alternatives.md)) — the intent-why (why a choice was made; options considered and rejected), distinct from the supersession lineage.
- **`attribution`** block on factlets ([RFC 0011](rfcs/0011-attribution.md)) — splits `produced_by` (the agent that generated the claim) from `authority` (the role accountable for it).

The organizational-layer model — `layer_type`, `precedence_rank`, supersession-at-read-time — is defined in [RFC 0007](rfcs/0007-factbook-layers.md). Software-profile field additions `applies_to` ([RFC 0008](rfcs/0008-applicability-selector.md)) and `verify` ([RFC 0009](rfcs/0009-verifiable-assertions.md)) live in the software-engineering profile, not the base spec. How a base schema and a profile schema combine for validation is specified in §15.6 ([RFC 0013](rfcs/0013-schema-composition-validation-levels.md)).

### 15.6 Schema composition and validation levels (RFC 0013)

A factlet's keys fall into three validation categories ([RFC 0013](rfcs/0013-schema-composition-validation-levels.md)):

1. **Document level** (top-level keys) — **open-world**. The base and profile schemas set top-level `additionalProperties: true`. A validator MUST NOT reject a factlet for an unrecognized top-level key; it MUST preserve it on round-trip and SHOULD warn if the key belongs to a declared-but-uninstalled profile (§15.3, RFC 0002 §4).
2. **Closed owned-objects** — **closed-world**. A nested object whose schema declares `additionalProperties: false` (e.g. `attribution`, `verify` and their sub-objects, `alternatives[]` items) rejects unrecognized keys; this is the spec's fail-loud guarantee.
3. **Open-by-design objects** — **open-world**. `extension` (vendor keys a reader ignores) and `origination` (a profile may add provenance keys) stay open; unknown keys are preserved, not rejected.

For a Factbook declaring `profile: <name>`, the **effective** factlet schema is the property-level union of base and profile properties with a document-level `additionalProperties: true`; closed owned-objects keep `additionalProperties: false`. Compose by property-union, **not** `allOf: [base, profile]` (an `allOf` with `additionalProperties: false` rejects sibling-branch fields). The base `origination.source_type` carries no enforced `enum` — the five baseline values are recommended; profiles may add values and readers SHOULD warn on (not reject) an unknown value.
