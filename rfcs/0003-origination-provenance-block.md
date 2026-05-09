# RFC 0003: Origination provenance block

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-05-08
- **Last updated**: 2026-05-08
- **Discussion**: TBD
- **Target spec version**: v0.2

## Summary

Introduce an optional `origination` block on factlets that records who or what produced the factlet, when, from what source material, and with what prior trust level. The block separates **what the factlet says** (existing `statement` + `sources`) from **where the assertion came from in the production pipeline** (`origination`). This separation is required so that consuming agents can render trust badges, gate auto-promotion on producer type, and audit the LLM-vs-human authorship ratio in a Factbook over time.

## Motivation

The v0.1 spec captures `sources` as provenance pointers to the underlying material a factlet describes (file:line, commit hash, URL, incident ID). It does not capture the provenance of the **factlet record itself** — who wrote the YAML entry, when, by what process, and with what confidence the producer warrants relative to others.

This distinction matters in three observed scenarios:

### 1. Mixed-author Factbooks (LLM + human)

A common pattern in production tooling: nightly LLM passes scan source code or session transcripts and propose factlets; humans review and accept/reject them. A Factbook in this regime contains a mix of:

- Human-authored entries (founder typed them in)
- LLM-proposed-and-human-approved entries (forward-pass writer with reviewer signoff)
- LLM-proposed-pending entries (in a review queue)

A consuming agent rendering a factlet to a user needs to display these differently. A user reviewing factlet f042 should know whether a human wrote it or whether an LLM proposed it after extracting from session-transcripts. Without origination metadata, the consumer cannot make this distinction.

### 2. Auto-promotion gates

A reviewer-driven workflow may apply different acceptance gates depending on producer:

- Human-authored entries: accept on submission.
- LLM-proposed entries: gate on a confidence threshold + an eval-pass before promotion.
- Cross-Factbook-imported entries (per RFC 0004): gate on the source factbook's trust prior.

Without an explicit producer type and trust-prior field, the workflow has no canonical place to read this signal — implementations resort to ad-hoc tag conventions (`tags: [llm-proposed]`) which are not enforceable by the spec.

### 3. Audit and FactSignal calibration

Empirical question that arose during a 2026-05 retrieval audit: "what fraction of factlets the LLM cited were originally LLM-proposed vs human-authored?" If the answer is near 100% LLM-proposed, the LLM is essentially citing its own past extractions — a closed feedback loop that may amplify extraction errors. The audit could not be answered without an `origination.source_type` field.

## Detailed design

### 1. Block definition

A factlet **MAY** carry an `origination` object with the following sub-fields:

| Sub-field      | Type            | Required (within block) | Description |
|----------------|-----------------|-------------------------|-------------|
| `source_type`  | string (enum)   | Yes                     | Producer type. Enum: `manual`, `llm`, `import`, `forward-pass`, `reverse-pass`. |
| `source_ref`   | string          | No                      | Free-form pointer to the production event (commit hash of the proposing PR, session ID, importer name+version). |
| `authored_at`  | string (ISO 8601) | No                    | When the factlet record was created. Distinct from `freshness.extracted_at` if present (which refers to when the underlying claim was last validated). |
| `authored_by`  | string          | No                      | Identifier for the producer. For `source_type: manual`, conventionally `human:<email>`. For `source_type: llm`, conventionally `llm:<model-id>`. For `source_type: import`, conventionally `<importer-id>:<version>`. |
| `trust_prior`  | float           | No                      | Producer's prior trust level in `[0.0, 1.0]`. Distinct from per-factlet `confidence`. Used for cross-factbook composition (RFC 0004) and auto-promotion gating. Default if omitted: 1.0 for `manual`, implementation-defined for other types. |

The `origination` block as a whole is **OPTIONAL** for backward compatibility with v0.1 Factbooks. If present, `source_type` is the only required sub-field; all others are optional but recommended.

### 2. Enum values for `source_type`

- `manual` — a human authored the factlet directly (typed it into a Factbook file).
- `llm` — a language model proposed the factlet content, regardless of whether a human reviewed it. (Reviewer status is captured by `review_status` from v0.1 §3.2, not by `source_type`.)
- `import` — the factlet was imported from another Factbook or external source via an explicit import operation.
- `forward-pass` — the factlet was emitted by a generation-time pipeline (e.g. a writer that fires while an LLM is producing code).
- `reverse-pass` — the factlet was emitted by an audit-time pipeline (e.g. a writer that fires while an audit is comparing code to existing factlets).

The forward-pass / reverse-pass distinction is intentionally separate from generic `llm` because the production context differs: forward-pass writers operate during user-facing generation; reverse-pass writers operate during background audits. Consuming agents may want to weight or filter them differently.

#### 2.1 Profile-specific source_type extensions

The base enum is the **minimum required vocabulary** for v0.2 conformance. A registered Profile (per RFC 0002) MAY extend the enum with profile-specific values to capture domain-relevant production sources. Examples:

- A future Manufacturing Profile may add `opc-ua-server`, `historian-extract`, `plc-config-export`.
- A future Healthcare Profile may add `hl7-message`, `ehr-extract`.

A profile-extended source_type MUST be documented in that profile's `SPEC.md` under §schema. Readers that do not know the profile MUST treat the unknown source_type as if `origination` were absent (i.e. preserve the field on round-trip without applying source-type-specific logic). This preserves base-spec conformance while enabling profile-aware producers to record richer provenance.

### 3. Schema fragment

Added to `schema/factlet.schema.json`:

```json
"origination": {
  "type": "object",
  "properties": {
    "source_type": {
      "type": "string",
      "enum": ["manual", "llm", "import", "forward-pass", "reverse-pass"]
    },
    "source_ref":  { "type": "string" },
    "authored_at": { "type": "string", "format": "date-time" },
    "authored_by": { "type": "string" },
    "trust_prior": { "type": "number", "minimum": 0.0, "maximum": 1.0 }
  },
  "required": ["source_type"],
  "description": "Provenance of the factlet RECORD itself (distinct from the `sources` field, which provides provenance for the underlying claim)."
}
```

### 4. Worked example

```yaml
- id: f001
  statement: "We use Stripe webhooks (not polling) for payment status updates."
  confidence: 0.95
  sources: ["docs/payments-arch.md:42", "commit:abc123"]
  origination:
    source_type: manual
    authored_at: "2026-04-15T10:30:00Z"
    authored_by: "human:alice@example.com"
    trust_prior: 1.0

- id: f002
  statement: "Database queries longer than 500ms emit a slow-query metric to Prometheus."
  confidence: 0.85
  sources: ["src/db/wrapper.py:184", "metrics.py:62"]
  origination:
    source_type: forward-pass
    source_ref: "session:2026-05-08-claude-code-7f3a2b"
    authored_at: "2026-05-08T14:22:00Z"
    authored_by: "llm:claude-opus-4-7"
    trust_prior: 0.7

- id: f003
  statement: "Refunds older than 90 days require manual ops approval."
  confidence: 1.0
  sources: ["docs/refund-policy.md:8"]
  # no origination block — interpreted as v0.1-shape factlet, treated by v0.2 readers as origination.source_type=unknown
```

### 5. Distinction from existing `sources` field

The two fields answer different questions:

- `sources`: "Where can I verify the factlet's claim?" → file:line of the policy doc, commit that introduced the behavior, the incident this came from.
- `origination`: "Where did this YAML record come from in our pipeline?" → producer type, when produced, by whom.

Both can be present and unrelated. A factlet about Stripe webhook usage (`sources: ["docs/payments-arch.md:42"]`) might have been authored by a human (`origination.source_type: manual`) or proposed by a forward-pass writer (`origination.source_type: forward-pass`). The two carry no contradiction.

### 6. Distinction from existing `freshness` and `review_status` fields

Implementations that adopted the optional `freshness: {extracted_at, method}` and `review_status` patterns from v0.1 §3.2 retain those fields. `freshness.extracted_at` records when the underlying claim was last verified; `origination.authored_at` records when the factlet record was created. They may differ if a factlet record is updated (re-verified) without re-authoring.

`review_status` records the reviewer-acceptance state (e.g. `unverified`, `verified`). It is orthogonal to producer type — an LLM-proposed factlet can carry `review_status: verified` after human review, with `origination.source_type: llm`.

## Alternatives considered

### A. Conflate origination with `sources`

Reuse the `sources` list with conventional prefixes (e.g. `sources: ["llm:claude-opus-4-7", "docs/payments-arch.md:42"]`).

**Rejected because:** the two pointer types are different in nature (verification source vs production-pipeline source) and conflating them defeats the point of separable rendering. A factbook viewer rendering "click sources to verify" should not include LLM session IDs in that list.

### B. Origination as flat factlet fields (no nested block)

Add `source_type`, `authored_at`, etc. as top-level optional fields on the factlet rather than nested under `origination`.

**Rejected because:** (a) the field names would be ambiguous against existing `sources`; (b) the nested block makes it clear these five fields move together as a unit; (c) future origination sub-fields (e.g. `pipeline_version`, `producer_eval_score`) can be added under `origination` without polluting the top-level factlet namespace.

### C. Origination only for LLM-produced factlets (skip `manual`)

Some consuming use cases only care to flag LLM-produced entries; origination could be required-when-LLM-only.

**Rejected because:** the audit use case ("what fraction is LLM-vs-human") needs explicit `manual` markers, not implicit absence-of-origination. An entry without origination cannot be reliably distinguished from "human-authored, omitted block" vs "LLM-authored, lazy producer."

## Migration impact

- **For Factbook authors**: no required change. Existing v0.1 Factbooks remain valid v0.2 Factbooks. Authors of new factlets **SHOULD** include the `origination` block with at minimum `source_type: manual`. Authors of existing factlets **MAY** backfill the block; absence is interpreted as "unknown source" by v0.2 consumers.
- **For implementations**: the block is additive. Implementations that wish to render trust badges or gate on producer type **MAY** read the block; implementations that ignore it **MUST** still preserve it on round-trip (read → write).
- **Breaking change?** No.

## Open questions

- Should the spec recommend a default `trust_prior` table (e.g. `manual=1.0, llm=0.7, forward-pass=0.7, reverse-pass=0.8, import=0.5`)? Currently left implementation-defined to avoid premature standardization. A future RFC may propose defaults once empirical calibration data exists across multiple implementations.
- Is `pipeline_version` (the version of the producing pipeline) worth a sub-field? Deferred to a possible future RFC; for now folded into `source_ref` as free-form text.
- Should `origination.authored_by` carry a structured identifier (e.g. an OIDC claim) instead of free-form string? Deferred — v0.2 keeps it free-form to accommodate the broad range of producers (human emails, LLM model IDs, importer names).

## Acceptance criteria

- Two maintainer approvals OR working group consensus
- Update to SPEC.md §3.2 (add `origination` row to optional fields table) and §3.3 (add a worked example with the block)
- Update to `schema/factlet.schema.json` (add the object property)
- Reference SDK (factlet-ai/reference-sdk) PR adding (a) recognition of the block on read, (b) preservation on round-trip write, (c) test fixtures covering all five `source_type` enum values, (d) a query helper that filters factlets by `origination.source_type`.
- At least one example factbook in the registry annotated with a representative mix of `source_type` values.
