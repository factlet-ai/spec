# RFC 0005: Software Profile — phase enum on factlets

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-05-08
- **Last updated**: 2026-05-08
- **Discussion**: TBD
- **Target spec version**: v0.2 (as the first concrete profile under the Profiles mechanism — RFC 0002)

## Summary

Define the **Software Engineering Profile** as the first registered profile under the Profiles mechanism (RFC 0002), and within it introduce an optional `phase` field on factlets with the enum values `design`, `implementation`, `testing`, `runtime`. The phase field is **scoped to factbooks declaring `profile: software-engineering`** and is not part of the base spec. Manufacturing, healthcare, and other profiles will define their own lifecycle vocabulary in separate sub-RFCs.

## Motivation

LLM grounding tools observe a recurring pattern in software-engineering use: the same body of project truth is consulted across distinct phases of work, and the same factlets are not equally relevant across phases. Empirical observation across N=6 dogfood sessions and a cross-vendor retrieval audit (April–May 2026):

- A factlet stating "we use Stripe webhooks (not polling) for payment status updates" is highly relevant during **design** discussion of a new payment flow, marginally relevant during **implementation** of that flow, and not relevant during **runtime** debugging of an unrelated incident.
- A factlet stating "tests use the script-style harness, not pytest discovery" is highly relevant during **testing** work, not during **design** or **runtime**.
- A factlet stating "all HTTP handlers run inside the request-context middleware for tracing" is highly relevant during **implementation** and **runtime** debugging, less so during **design**.

Without a phase signal, retrieval injects all relevant factlets uniformly, producing noisy context that includes phase-irrelevant material. With a phase signal, implementations MAY filter to phase-appropriate factlets and reduce token spend on injection while improving relevance density.

The phase enum is also load-bearing for **conformance audit** use cases in software contexts: an audit comparing implemented code to documented design decisions must select factlets tagged with `phase: design` or `phase: implementation`, not `phase: runtime`. Without the enum, the audit either grades every factlet against every code change (high false-positive rate) or relies on free-form `tags` (no shared vocabulary across implementations).

### Why scoped to the Software Profile (not base)

The four phases (`design`/`implementation`/`testing`/`runtime`) are software-lifecycle vocabulary. Other domains have analogous but distinct lifecycles:

- **Manufacturing**: `commissioning` / `operation` / `maintenance` / `decommissioning` (or per-batch `setup` / `exotherm` / `cooldown`).
- **Healthcare**: `intake` / `diagnosis` / `treatment` / `followup`.
- **Legal**: `discovery` / `argument` / `judgment` / `appeal`.

Putting `phase` in the base factlet schema with software vocabulary would enshrine the software lifecycle as universal across all factbooks — exactly the conflation the Profiles mechanism (RFC 0002) was designed to prevent. The Manufacturing Profile (future sub-RFC) will define its own lifecycle field, likely also named `phase` but with manufacturing enum values, scoped under `profile: manufacturing`.

## Detailed design

### 1. Profile registration

This RFC registers the **Software Engineering Profile** under `factlet-ai/spec/profiles/software-engineering/` per the Profiles mechanism in RFC 0002.

Profile identifier: `software-engineering`
Profile version (initial): `0.2` (matches the spec ratification version)
Profile maintainer: factlet-ai/spec working group

### 2. Phase field definition

A factlet **inside a Factbook declaring `profile: software-engineering`** MAY carry a `phase` field. If present, its value MUST be one of:

- `design` — applies during architecture and design-decision phase (technology choices, system structure, API contracts before implementation)
- `implementation` — applies during code-writing phase (conventions, anti-patterns, library usage, naming patterns)
- `testing` — applies during test authoring or test execution (harness conventions, test data, coverage rules)
- `runtime` — applies during production operation (alarm handling, incident response, operational thresholds, vendor quirks observed in production)

The `phase` field MAY be omitted, in which case the factlet is interpreted as **phase-agnostic within the software-engineering profile** (relevant across phases). The default is omission, not a sentinel value.

A factlet WITHOUT a profile declaration (i.e. in a profile-neutral Factbook, or carrying `profile: <other>`), the `phase` field is interpreted as a profile-unknown extension and SHOULD NOT carry software-engineering semantics. Producers MUST NOT use `phase` outside the `software-engineering` profile scope without first defining its semantics under the relevant profile.

### 3. Schema fragment

Added to `schema/profiles/software-engineering/factlet.schema.json` (NOT to base `schema/factlet.schema.json`):

```json
"phase": {
  "type": "string",
  "enum": ["design", "implementation", "testing", "runtime"],
  "description": "The software lifecycle phase to which this factlet applies. If omitted, the factlet is phase-agnostic within the software-engineering profile."
}
```

### 4. Worked example

```yaml
schema_version: v1.0
profile: software-engineering
profile_version: "0.2"
last_updated: 2026-05-08T00:00:00Z
metadata:
  name: payments-service
  scope: project:payments

content:
  - id: f001
    statement: "Refunds older than 90 days require manual ops approval, never auto-processed."
    confidence: 1.0
    sources: ["docs/refund-policy.md:8"]
    phase: design
    tags: [payments, ops, compliance]

  - id: f002
    statement: "Database migrations run via Alembic; never raw SQL ALTER TABLE in application code."
    confidence: 1.0
    sources: ["docs/db-conventions.md:14"]
    phase: implementation
    tags: [database, conventions]

  - id: f003
    statement: "Slow-query metric exported when SQL execution exceeds 500ms; alarm fires at 1s."
    confidence: 1.0
    sources: ["src/db/wrapper.py:184", "alerts.yaml:42"]
    phase: runtime
    tags: [observability, alerts]

  - id: f004
    statement: "Tests use the script-style harness, not pytest discovery."
    confidence: 1.0
    sources: ["docs/testing-conventions.md:5"]
    phase: testing
    tags: [testing, conventions]

  - id: f005
    statement: "Public registry hosts MIT-licensed example factbooks across multiple domains."
    confidence: 1.0
    sources: ["github.com/factlet-ai/registry"]
    # no phase — phase-agnostic factlet within software-engineering profile
```

### 5. Retrieval semantics

Implementations targeting the software-engineering profile MAY filter by phase at retrieval time. The protocol does not mandate a specific filter algorithm; the contract is that an implementation that surfaces phase-filtered results MUST honor the field as defined.

Recommended consumer behavior:
- A consuming agent that knows the current phase (e.g. an IDE plugin during code authoring is in `implementation` phase; an incident-response surface is in `runtime` phase) SHOULD request retrieval scoped to that phase plus phase-agnostic factlets.
- A consuming agent that does not know the phase SHOULD request retrieval across all phases (no filter), which is equivalent to v0.1 behavior.

### 6. Interaction with FactSignal

FactSignal (§6 of v0.1 SPEC) is computed against a retrieved factlet set. If retrieval is phase-filtered, FactSignal is computed against the filtered set. Implementations SHOULD document whether their FactSignal score reflects all-phase or phase-filtered coverage.

### 7. Backward compatibility

Existing v0.1 software-engineering Factbooks contain no `phase` field and no `profile` declaration. They remain valid v0.2 Factbooks under the profile-neutral interpretation defined in RFC 0002. To opt into phase-filtered retrieval, an author adds:

1. `profile: software-engineering` at the Factbook root.
2. `phase: <value>` to factlets where the lifecycle phase is known.

Both additions are optional and can be applied incrementally. No re-authoring is required.

## Alternatives considered

### A. Phase enum in the base spec (universal)

Add `phase` to the base factlet schema with software vocabulary as universal.

**Rejected because:** the four phases are software-lifecycle vocabulary; manufacturing, healthcare, and legal have different lifecycles. Putting software vocabulary in the universal base enshrines one domain's worldview as the spec's worldview. This is the failure mode the Profiles mechanism (RFC 0002) was designed to prevent.

### B. Free-form `tags` instead of an enum

Producers could use tags like `phase:design` without a schema change.

**Rejected because:** the tag namespace is open and unenforceable. Two implementations would diverge on conventions (`phase:design` vs `lifecycle:design` vs `stage:architecture`), and consuming agents could not reliably filter without normalization. Same as Alternative B in RFC 0002.

### C. Multi-phase as a list (`phases: [design, runtime]`)

Allow factlets to span multiple phases with a list rather than scalar.

**Rejected for v0.2** because (a) the empirical pattern is that most factlets are phase-specific or fully phase-agnostic, with the multi-phase case rare; (b) a list complicates retrieval filter semantics; (c) a future profile-version bump can introduce list semantics if the empirical pattern changes. Forward-compatible: a list-valued field in a future version could replace the scalar without breaking v0.2 readers (per v0.1 §9.3 "MUST ignore unknown keys").

### D. Numeric ordinal (1=design, 2=implementation, ...)

Use integers to permit lifecycle-ordering arithmetic.

**Rejected because:** the four phases do not form a strict total order in practice — `runtime` factlets feed back into `design` decisions, and `testing` overlaps `implementation`. A string enum captures the categorical nature without misleading ordering semantics.

## Migration impact

- **For software-engineering Factbook authors**: no required change. Existing v0.1 Factbooks remain valid v0.2 Factbooks (treated as profile-neutral by the Profiles mechanism). Authors who want phase-filtered retrieval declare `profile: software-engineering` at root and add `phase` fields to factlets at their discretion.
- **For implementations**: no required change beyond v0.2 schema validation (per RFC 0002 Profiles mechanism conformance). Implementations that wish to expose phase-filtered retrieval MAY do so when the Factbook declares the software-engineering profile.
- **Breaking change?** No. Additive optional field within a registered profile.

## Open questions

- Should `phase: design` be subdivided in a future profile version (e.g. `design:architecture`, `design:contract`)? Deferred; v0.2 keeps the four-value enum and lets producers use tags for finer subdivision.
- Should this profile add other software-specific fields beyond `phase` in v0.2 (e.g. `language`, `framework`)? Deferred. v0.2 ships the minimal addition needed for phase-filtered retrieval; further fields are added via separate profile-version bumps with their own justification.

## Acceptance criteria

- Two maintainer approvals OR working group consensus
- RFC 0002 (Profiles mechanism) accepted as a prerequisite
- Create `factlet-ai/spec/profiles/software-engineering/` directory containing: `SPEC.md`, `factlet.schema.json`, `examples/payments-service.yaml`
- Add `software-engineering` to `factlet-ai/spec/profiles/REGISTRY.md`
- Reference SDK (factlet-ai/reference-sdk) PR adding (a) software-profile schema validation when `profile: software-engineering` is declared, (b) optional phase-filter parameter on retrieval, (c) test fixtures covering phase-tagged and phase-agnostic factlets within the software profile.
- At least one example factbook in `factlet-ai/registry` updated to declare `profile: software-engineering` and use phase-tagged factlets (additive, not destructive).
