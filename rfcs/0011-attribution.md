# RFC 0011: Attribution — producer vs authority

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-06-01
- **Last updated**: 2026-06-01
- **Discussion**: TBD
- **Target spec version**: v0.2
- **Scope**: project:factlet-ai

## §1 Summary

This RFC introduces an optional base-spec `attribution` block on factlets that
splits two relations the spec has so far named only in part: **`produced_by`**,
the agent or process that *generated* the claim, and **`authority`**, the role
or owner *accountable* for the claim being true. The two answer different
questions — *who derived this* versus *who vouches for this* — and they diverge
constantly, most sharply when the producer is an automated agent that can
generate a claim but cannot be accountable for it. RFC 0003 named the producer
(via the `origination` block); no field in the spec names the accountable
authority. RFC 0011 names the authority and pairs it with the producer so the
distinction is legible in one place. The block is domain-general — every domain
has a producer and an accountable authority — so it belongs in the base spec
rather than a profile.

## §2 Motivation

The spec already captures where a factlet's claim can be verified (`sources`,
v0.1) and where the factlet *record* came from in a production pipeline
(`origination`, RFC 0003 — producer type, when produced, by whom). What the
spec does not capture is **who is accountable for the claim being true**.

These are different facts, and the difference is load for any consumer that
must decide whether to trust, gate, or act on a factlet.

### 2.1 The producer is increasingly an automated agent

A common production pattern: a model scans source code, transcripts, or
documents and proposes factlets; a human or a role then reviews and accepts
them. RFC 0003 lets a consumer read *that* a factlet was produced by a model
run rather than typed by a human. It does not let the consumer read *who is
accountable* for the resulting claim.

The distinction matters because an automated agent can **produce** a claim but
cannot **be** its authority. A factlet distilled by a model from a pull request
is produced by that extraction run; the authority for the underlying
convention is the human or role who owns it. Collapsing the two — treating the
agent that wrote the claim as the party that vouches for it — is precisely the
failure a grounding system exists to prevent. A consumer needs both facts, and
needs them separable, to tell an un-vouched candidate apart from a claim a named
owner stands behind.

### 2.2 `origination` names the producer; nothing names the authority

RFC 0003's `origination` block carries `authored_by` (a producer identifier)
and `source_type` (a producer class). Both describe **derivation**: who or what
generated the record. None of the v0.1 or v0.2-draft fields describe
**accountability**:

- `sources` points at where the claim can be verified, not at who owns it.
- `origination.authored_by` is the producer, not the accountable party.
- `review_status` (v0.1 §3.2) records *whether* a factlet was reviewed, not
  *who* is accountable for it once reviewed.

The accountable owner is therefore unnamed in the spec. RFC 0011 names it as
`attribution.authority` and restates the producer alongside it as
`attribution.produced_by`, so the producer/authority pair sits together and the
distinction is explicit at the point of use.

### 2.3 A single combined field cannot express the split

An implementation that records only one "attributed to" value is forced to put
either the producer or the authority in that slot. When the producer is an
automated agent, neither choice is correct: storing the agent loses the
accountable owner, and storing the owner loses the derivation. A consumer
reading a single field cannot recover which of the two it holds. The split into
two named relations is the minimum that lets both facts coexist.

## §3 Specification — the `attribution` block

A factlet **MAY** carry an `attribution` object with two optional sub-objects:
`produced_by` and `authority`. The block as a whole is **OPTIONAL** for
backward compatibility with v0.1 and earlier v0.2-draft factbooks; a reader that
encounters no block **MUST** treat both relations as absent and **MUST**
preserve the absence on round-trip.

### 3.1 `produced_by` — derivation

`produced_by` records the agent or process that generated the claim. It is a
fact about **derivation**, not accountability.

| Sub-field | Type   | Required (within `produced_by`) | Description |
|-----------|--------|---------------------------------|-------------|
| `agent`   | string | Yes                             | Identifier of the producing agent, run, or author (e.g. `extraction:run-1f3a`, `human:alice@example.com`, `session:452`). |
| `kind`    | string (enum) | No                       | Coarse class of producer. Enum: `human`, `agent`, `extraction`, `import`. |

When the `produced_by` sub-object is present, `agent` is **REQUIRED**; a
producer relation with no producer identifier carries no information.

### 3.2 `authority` — accountability

`authority` records the role or owner accountable for the claim being true. It
is a fact about **sign-off and ownership**, not derivation. It **MAY** be
absent: a freshly produced candidate has a producer and no authority yet.

| Sub-field     | Type   | Required (within `authority`) | Description |
|---------------|--------|-------------------------------|-------------|
| `owner`       | string | No                            | Accountable role or named owner (e.g. `role:payments-lead`, `team:payments`). |
| `approved_by` | string | No                            | Identifier of the party that signed off, if the claim has been reviewed. |
| `approved_at` | string (ISO 8601) | No                 | When the sign-off occurred. |
| `review_status` | string | No                          | Recorded **only** when it signals explicit sign-off (e.g. `approved`), accompanying `owner`/`approved_by` — not the transient review-workflow state (see Alternative C). |

All `authority` sub-fields are optional. The absence of `authority` denotes an
un-vouched factlet — produced, not yet owned.

### 3.3 Schema fragment

Added to `schema/factlet.schema.json`:

```json
"attribution": {
  "type": "object",
  "description": "Splits derivation (produced_by) from accountability (authority). produced_by answers 'who generated this claim'; authority answers 'who is accountable for it being true'.",
  "properties": {
    "produced_by": {
      "type": "object",
      "description": "The agent or process that generated this claim. Derivation, not accountability.",
      "properties": {
        "agent": {
          "type": "string",
          "description": "Identifier of the producing agent, run, or author."
        },
        "kind": {
          "type": "string",
          "enum": ["human", "agent", "extraction", "import"],
          "description": "Coarse class of producer."
        }
      },
      "required": ["agent"]
    },
    "authority": {
      "type": "object",
      "description": "The role or owner accountable for this claim being true. Sign-off, not derivation. MAY be absent for unreviewed candidates.",
      "properties": {
        "owner":         { "type": "string" },
        "approved_by":   { "type": "string" },
        "approved_at":   { "type": "string", "format": "date-time" },
        "review_status": { "type": "string", "description": "Review state recorded only when it signals explicit sign-off; distinct from transient review-workflow state (see Alternative C)." }
      }
    }
  },
  "additionalProperties": false
}
```

Both sub-objects are optional at the block level. `produced_by.agent` is
required when the `produced_by` sub-object is present. The block is additive: a
v0.1 reader interprets its absence as no producer and no authority recorded.

## §4 Relationship to RFC 0003 `origination`

RFC 0003 introduced the `origination` block, which records the producer of the
factlet record (`authored_by`, `source_type`). `attribution.produced_by`
expresses the **same producer/derivation concept**, restated inside the
attribution block so that producer and authority sit together and the
distinction between them is visible in one place rather than inferred across two
blocks.

The two are reconcilable, not contradictory:

- `attribution.produced_by.agent` corresponds to `origination.authored_by`.
- `attribution.produced_by.kind` is a **coarser** classification than
  `origination.source_type`. `source_type` distinguishes pipeline contexts
  (`forward-pass`, `reverse-pass`, `llm`, `manual`, `import`); `kind` collapses
  these into four broad producer classes (`human`, `agent`, `extraction`,
  `import`) suited to a reader that only needs to know *whether* a human, an
  automated agent, an extraction run, or an import generated the claim. The two
  vocabularies are deliberately distinct: `kind` is not a renaming of
  `source_type` and the two enums are not required to be in 1:1 correspondence.

An implementation that records `origination` **MAY** project between it and
`attribution.produced_by` (deriving `agent` from `authored_by`, and `kind` from
`source_type` via an implementation-defined mapping). The spec does not require
the projection; it requires only that, if both blocks are present, they not
contradict — `attribution.produced_by.agent` and `origination.authored_by`
**MUST** identify the same producer.

The novel normative content of this RFC is therefore `attribution.authority`:
no prior field names the accountable owner. `attribution.produced_by` exists so
that the producer and the authority can be read as a pair.

## §5 Why base spec, not a profile

The producer/authority distinction is universal across domains, so it belongs in
the base spec rather than a registered profile (RFC 0002):

- Software: an extraction run produces a convention; an engineering role owns it.
- Manufacturing: an operator records a specification; a quality authority owns it.
- Healthcare: a clinician charts an observation; an attending of record owns it.

Every domain has both a party that generated a claim and a party accountable for
it. Because the relation is domain-general, scoping it to a profile would force
each domain to redefine the same two fields. The block is placed in the base
spec for the same reason `origination` (RFC 0003) and the rationale block
(RFC 0010) are base-spec: the concept is not domain-specific.

## §6 Worked examples

```yaml
# Producer + authority: a claim a named role stands behind.
- id: f001
  statement: "Refunds older than 90 days require manual ops approval."
  confidence: 1.0
  sources: ["docs/refund-policy.md:8"]
  attribution:
    produced_by:
      agent: "extraction:run-1f3a"
      kind: extraction
    authority:
      owner: "role:payments-lead"
      approved_by: "human:alice@example.com"
      approved_at: "2026-05-20T16:00:00Z"

# Producer only: an un-vouched candidate. Produced by an agent, no authority yet.
- id: f002
  statement: "Refund webhooks retry up to 5 times with exponential backoff."
  confidence: 0.7
  sources: ["src/payments/webhooks.py:212"]
  attribution:
    produced_by:
      agent: "session:2026-06-01-7f3a2b"
      kind: agent
    # no authority block — produced, not yet owned by an accountable role

# Absent block: a v0.1-shape factlet. Readers treat producer and authority
# as unrecorded.
- id: f003
  statement: "Payment status updates arrive via webhook, not polling."
  confidence: 0.95
  sources: ["docs/payments-arch.md:42"]
  # no attribution block
```

The producer-only case (`f002`) is the one the split exists to express: a claim
generated by an automated agent with no accountable authority. A single combined
field could not represent this state without either fabricating an authority or
discarding the producer.

## §7 PROV-O projection

The two relations project cleanly onto the W3C PROV-O vocabulary:

- `attribution.produced_by` → `prov:wasGeneratedBy` a `prov:Activity` whose
  associated `prov:Agent` carries `prov:hadRole "producer"`.
- `attribution.authority` → `prov:wasAttributedTo` a `prov:Agent` carrying
  `prov:hadRole "authority"`.

Two distinct roles on distinct agents is the standard PROV-O construction for
"generated by X, vouched for by Y." An implementation that emits PROV-O
serializations of factlets **SHOULD** use this mapping so the producer/authority
distinction survives the projection. The mapping is informative, not normative;
implementations that do not emit PROV-O are unaffected.

## §8 Alternatives considered

### A. Scope `attribution` to a profile

Define producer/authority only within profiles that need it (e.g. a regulated
manufacturing or healthcare profile).

**Rejected because:** the relation is universal — every domain has a producer and
an accountable authority. Scoping to a profile would force each domain to
redefine the same two fields, and would leave the base spec unable to express
accountability at all.

### B. A single `attributed_to` field

Record one combined value covering both producer and authority.

**Rejected because:** this is the conflation the RFC exists to name. A single
field forces "the agent that generated the claim" and "the role accountable for
it" into one slot. When the producer is an automated agent, neither value is the
authority, and a consumer reading the field cannot recover which of the two it
holds. The producer-only example in §6 is unrepresentable under a single field.

### C. Record authority only as review state, not on the factlet

Leave accountability to the review/governance workflow and keep it off the
factlet record itself (relying on `review_status` and equivalent workflow state).

**Rejected because:** `review_status` records *whether* a factlet was reviewed,
not *who* is accountable for it. A consumer rendering, gating, or auditing a
factlet needs the accountable owner as a first-class, queryable attribute of the
factlet — not a state that must be reconstructed from a separate workflow record.
Review/governance state remains the *workflow*; `attribution.authority` is the
*settled result* recorded on the factlet. (`authority` MAY itself carry an
optional `review_status` sub-field — see §3.2 — but only as a recorded
explicit-sign-off signal accompanying the accountable `owner`/`approved_by`,
never as a substitute for naming the accountable party. That narrow use is the
*settled result*, not the transient workflow state this alternative would have
relied on.)

### D. Do nothing

Keep the producer named (RFC 0003) and leave the authority unnamed.

**Rejected because:** with the producer increasingly an automated agent, a
consumer can read what generated a claim but not who is accountable for it being
true. That is exactly the case the spec needs to express, and no existing field
expresses it.

This block also distinguishes who *decided* something (the authority for a
decision) from what *generated* the record of that decision (the producer) — two
facts that an unsplit field cannot tell apart.

## §9 Migration impact

- **For factbook authors**: no required change. Existing factbooks remain valid;
  the block is additive and optional. Authors of new factlets **SHOULD** record
  `attribution.produced_by` at minimum, and **SHOULD** record
  `attribution.authority` once a factlet has an accountable owner. Authors **MAY**
  backfill the block on existing factlets; absence is interpreted as "producer
  and authority unrecorded."
- **For implementations**: the block is additive. Implementations that wish to
  render trust badges, gate promotion, or audit accountability **MAY** read the
  block; implementations that ignore it **MUST** still preserve it on round-trip
  (read → write).
- **Breaking change?** No. A v0.1 reader sees the block as absent.

## §10 Open questions

- Is `authority.owner` a free string or a reference into a roles registry? The
  current draft keeps it a free string for v0.2, deferring a registry to a future
  RFC — the same call RFC 0008 makes for framework identifiers.
- Should a factlet whose `produced_by.kind` is `agent` be **REQUIRED** to carry a
  non-empty `authority` before it leaves a candidate/unreviewed state? An
  agent-produced factlet with no accountable authority is exactly the un-vouched
  case. The current draft recommends a **SHOULD** for v0.2 and leaves a possible
  **MUST** to a future RFC once audit experience exists.
- Should the `produced_by.kind` ↔ `origination.source_type` mapping (§4) be
  standardized, or left implementation-defined? The current draft leaves it
  implementation-defined; a future RFC may publish a recommended mapping once
  empirical data exists across implementations.

## Cross-references

This RFC depends on or extends:

- **RFC 0002 — Profiles Mechanism** (base v0.2 spec): provides the base-vs-profile
  model that §5 applies to place `attribution` in the base spec.
- **RFC 0003 — Origination provenance block** (base): names the producer via
  `origination`. §4 reconciles `attribution.produced_by` with `origination` and
  identifies `attribution.authority` as the new normative content.
- **RFC 0008 — Software profile `applies_to`** (SE-profile): §10 mirrors RFC
  0008's deferral of a registry in favor of free-string identifiers for v0.2.
- **RFC 0009 — Software profile verifiable assertions** (SE-profile): consumes
  the producer/authority distinction when deciding what may gate or prove a
  factlet versus what may merely describe it.
- **RFC 0010 — Rationale and alternatives block** (base): the sibling base-spec
  block introduced for the same domain-general reason (§5); both reconcile
  attributes that were previously unnamed or scattered into clean spec blocks.

## Acceptance criteria

- Two maintainer approvals OR working group consensus.
- Update to `SPEC.md` adding the base `attribution` block, documenting the
  producer-vs-authority distinction and its reconciliation with RFC 0003's
  `origination` (per §4).
- Update to `schema/factlet.schema.json` adding the `attribution` object property
  per §3.3.
- Reference SDK (`factlet-ai/reference-sdk`) PR adding:
  - (a) recognition of the `attribution` block on read;
  - (b) preservation on round-trip write;
  - (c) round-trip test fixtures covering the three states in §6 — producer-only
    candidate, producer + authority approved, and absent block;
  - (d) a validation case asserting `produced_by.agent` is required when
    `produced_by` is present;
  - (e) a consistency check asserting `attribution.produced_by.agent` and
    `origination.authored_by` identify the same producer when both blocks are
    present (per §4).
- At least one registry-published example factbook carrying a representative mix:
  a producer-only candidate and a producer + authority factlet.
