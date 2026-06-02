# RFC 0009: Verifiable assertions

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-06-01
- **Last updated**: 2026-06-01
- **Discussion**: TBD
- **Target spec version**: v0.2 (software-engineering profile addition; see RFC 0002 Profiles, RFC 0005 phase enum)
- **Scope**: project:factlet-ai

## §1 Summary

This RFC introduces an optional `verify` block on factlets that records **how to check the factlet's claim** — a machine-checkable predicate, a reference to an executable test, or a set of golden input→output examples. The block separates **what the factlet asserts** (`statement`) and **why it can be trusted** (`sources`, `origination` — see RFC 0003) from **how conformance to it is decided** (`verify`).

A v0.1 factlet carries the claim and its provenance but not an *oracle* — a declared truth condition against which a candidate artifact can be checked. Without an oracle, a verifier can judge conformance only from a reading of the free-text `statement`; a boundary condition such as the difference between `> 90` and `>= 90` is caught only if the reader happens to notice it, never because it was asserted. The `verify` block makes the claim executable: the same block a verifier evaluates to render a verdict is the unit a migration audit compiles into an executable equivalence test.

## §2 Motivation

The v0.1 spec captures `statement` (the claim) and `sources` (where to verify it by hand). RFC 0003 added `origination` (who produced the record). None of these tell a *machine* how to decide whether a given artifact satisfies the factlet. Three consequences follow.

### 2.1 Verification is text-only and uncalibrated

A discriminative verifier asked "does this code satisfy factlet `f002`?" has only the natural-language `statement` to work from. Two failure modes follow: (a) boundary and edge conditions encoded in the statement's prose are silently missed (`> 90` vs `>= 90`), and (b) there is nothing to calibrate against — with no declared truth condition, a confidence score is not anchored to anything checkable.

### 2.2 An audit certificate cannot point at executable evidence

A migration-audit certificate binds, per rule, *legacy evidence ↔ factlet ↔ new-code evidence ↔ verdict*. The verdict is auditor-grade only if the check is reproducible. A prose statement is not reproducible; an assertion, a test reference, or a worked example is.

### 2.3 There is no portable seed for a regression harness

A factbook intended to drive regeneration of behaviorally-equivalent code, and a factbook intended to verify such code, both require an oracle. When a migration flags a boundary case, that flag should be capturable as a *test* that prevents silent regression thereafter. Without a `verify` block in the protocol, such a test lives outside the factbook and is not portable across implementations.

## §3 Block definition

A factlet **inside a Factbook declaring `profile: software-engineering`** **MAY** carry a `verify` object. It is **OPTIONAL**, and **RECOMMENDED** for factlets whose `fact_type` is behavioral or contractual (`rule`, `contract`, `constraint`, `behavioral`, `decision`). Readers that do not understand the block **MUST** preserve it on round-trip (read → write) without applying check semantics.

| Sub-field | Type | Required (within block) | Description |
|---|---|---|---|
| `kind` | string (enum) | Yes | One of `assertion`, `test_ref`, `examples`. Selects the form of the oracle. |
| `assertion` | object | if `kind: assertion` | A side-effect-free boolean expression plus its declared inputs. |
| `test_ref` | object | if `kind: test_ref` | A pointer to an executable test that encodes the check. |
| `examples` | array | if `kind: examples` | Golden input→output pairs the artifact must reproduce. |
| `boundary` | array of strings | No | Human-named boundary or edge conditions the check is known to exercise (e.g. `"age_days==90"`). Advisory; drives surfacing of flagged divergences. |

Exactly one of `assertion` / `test_ref` / `examples` is present, matching `kind`.

The *applicability* of a `verify` block — the set of files, languages, or frameworks where its check is evaluated — is **not** re-declared inside `verify`. It is governed by the factlet's top-level `applies_to` field, defined in RFC 0008 (applicability selector). A `verify` block inherits the scope of the factlet that carries it; it does not carry a nested applicability selector of its own.

### 3.1 `assertion`

```yaml
assertion:
  expr:  "age_days >= 90 implies route == 'MANUAL'"   # restricted grammar, side-effect-free
  vars:                                                # declared inputs + types
    age_days: int
    route:    enum[AUTO, MANUAL]
```

The `expr` grammar is intentionally minimal for v0.2: boolean operators (`and` / `or` / `not` / `implies`), comparisons (`==` `!=` `<` `<=` `>` `>=`), arithmetic (`+`, `-`, `/`, `%`), membership (`in`), and parenthesisation, over the variables declared in `vars`. Multiplication (`*`) and exponentiation (`**`) are **excluded** in v0.2: evaluated against untyped or sequence-typed `vars`, they are unbounded-allocation/compute amplifiers (`[0] * n`, `x ** n`), so a conforming evaluator **MUST** reject them as disallowed operators. They may return in a future profile version once the grammar carries a static type system that proves operands numeric. No function calls, no side effects, no I/O. Richer grammars (quantifiers, temporal operators) are likewise deferred. The expression is the **canonical truth condition**: a verifier evaluates a candidate against it, and any calibration layer is anchored to it.

### 3.2 `test_ref`

```yaml
test_ref:
  runner:   pytest                      # producer-declared test runner
  path:     tests/equivalence/test_refund_age_boundary.py
  selector: "test_age_90_routes_manual" # function / case id
```

`test_ref` points at an executable test — typically generated *into* the migrated codebase as part of an equivalence test suite. It is the strongest oracle: the check is "this test passes against the candidate." An audit certificate cites `runner` + `path` + `selector` as the reproducible evidence.

### 3.3 `examples`

```yaml
examples:
  - given: { age_days: 90,  amount: 10.00 }
    expect: { route: MANUAL }
    note: "boundary — legacy auto-posts at 90; the port must match the agreed behavior"
  - given: { age_days: 89,  amount: 10.00 }
    expect: { route: AUTO }
```

Golden input→output pairs. The oracle is direct equality of produced output versus `expect`. This form is the cheapest to author by hand, and is also the natural shape harvested from a legacy test suite or a bug reproduction.

## §4 How consumers use the block

A consumer of the block selects behavior by `kind` and the consumer's own role:

- A **verifier** selects the oracle by `kind` — evaluates `assertion` symbolically, runs `test_ref`, or checks `examples` — and emits a result (`verdict`, `rejection_code`, `confidence`) bound to the block. A `boundary` entry that the check does not cover routes to a needs-review verdict, not to a silent pass.
- An **auditor** cites the block as the reproducible check; the auditor re-runs `test_ref` or `examples`, or re-evaluates `assertion`, with off-the-shelf tooling.
- A **test generator** compiles each block into an executable test in the target codebase: `assertion` and `examples` become generated tests; `test_ref` already is one.

## §5 Schema fragment

Added to the software-engineering profile schema (`schema/profiles/software-engineering/factlet.schema.json`):

```json
"verify": {
  "type": "object",
  "properties": {
    "kind": { "type": "string", "enum": ["assertion", "test_ref", "examples"] },
    "assertion": {
      "type": "object",
      "properties": {
        "expr": { "type": "string" },
        "vars": { "type": "object", "additionalProperties": { "type": "string" } }
      },
      "required": ["expr"]
    },
    "test_ref": {
      "type": "object",
      "properties": {
        "runner":   { "type": "string" },
        "path":     { "type": "string" },
        "selector": { "type": "string" }
      },
      "required": ["runner", "path"]
    },
    "examples": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "given":  { "type": "object" },
          "expect": { "type": "object" },
          "note":   { "type": "string" }
        },
        "required": ["given", "expect"]
      }
    },
    "boundary": { "type": "array", "items": { "type": "string" } }
  },
  "required": ["kind"],
  "description": "How to CHECK the factlet's claim (oracle). Distinct from `statement` (the claim) and `sources` (where to verify by hand). Applicability is governed by the factlet's top-level `applies_to` (RFC 0008), not re-declared here. SE-profile."
}
```

The schema deliberately defines no `applies_to` property inside `verify`. A check's applicability is the factlet's applicability; the top-level `applies_to` field (RFC 0008) is the single point of declaration. A consumer evaluating the check scopes it by the factlet's `applies_to`, not by a per-block copy.

## §6 Worked example (the day-90 rule)

```yaml
- id: f002
  statement: "Refunds older than 90 days route to manual review."
  fact_type: rule
  phase: implementation            # RFC 0005
  status: verified
  confidence: 1.0
  rationale: "Regulatory edge case; pre-90 auto-posts. Day 90 itself auto-posts in legacy."   # RFC 0010
  sources: ["REFUND-CHK.cbl:11", "docs/refund-policy.md:8"]
  origination: { source_type: reverse-pass, authored_by: "llm:claude-opus-4-8", trust_prior: 0.8 }   # RFC 0003, RFC 0011
  applies_to: { paths: ["src/refund/**"] }   # RFC 0008 — top-level applicability selector
  verify:
    kind: examples
    boundary: ["age_days==90"]
    examples:
      - given: { age_days: 91 }
        expect: { route: MANUAL }
      - given: { age_days: 90 }            # the boundary the legacy suite lacked
        expect: { route: AUTO }
      - given: { age_days: 89 }
        expect: { route: AUTO }
```

The applicability of the check is expressed once, at the factlet's top-level `applies_to`; the `verify` block does not repeat it. A reverse-pass audit of a Java port that wrote `if (ageDays >= 90)` evaluates the `age_days==90` example, gets `MANUAL` where `AUTO` is expected, and emits `rejection_code: BOUNDARY_MISMATCH` routed to sign-off. On accept, the example is emitted as a permanent test.

## §7 Distinction from existing fields

- `statement` — *what* is claimed (natural language). `verify` — *how to check* it (machine).
- `sources` — where to verify by **hand** (file:line of the policy). `verify.test_ref` — where to verify by **machine** (executable test).
- `code_example` (implementation field) — an *illustrative* snippet that consumers render but do not run. `verify` — a *normative* oracle that consumers run. Do not conflate the two.
- `confidence` — the factlet's trust level. `verify` — the condition that, when checked, *earns* a verdict. Any calibration layer anchors confidence to the block.
- `applies_to` (RFC 0008) — *where* the check is evaluated. Declared once at the factlet's top level; `verify` inherits it and does not re-declare it.

## §8 Alternatives considered

### A. Overload `code_example` as the check

**Rejected.** `code_example` is illustrative and non-executable by convention; consumers render it, they do not run it. Overloading the field breaks that contract and provides no place for declared inputs or expected outputs.

### B. Keep the predicate outside the factbook (in a test repository only)

**Rejected.** The oracle would then not be portable, would not be bound to the factlet's provenance and supersession history, and could not be cited by an audit certificate as part of the factbook. The check must travel *with* the fact for the verification and regeneration use cases to hold.

### C. A single rich expression language (e.g. CEL or Rego) only

**Rejected for v0.2.** Forcing every check into one DSL is high-friction: `examples` (harvested from tests and bug reproductions) and `test_ref` (existing CI tests) cover the common cases at lower cost. The tagged union lets producers pick the cheapest faithful form. A future RFC may standardize a richer `assertion` grammar once usage data exists.

## §9 Migration impact

- **Factbook authors:** no required change. v0.1 and early-v0.2 factbooks remain valid. Absence of a `verify` block means "no machine oracle declared," and a verifier falls back to statement-only judgment at correspondingly lower confidence.
- **Implementations:** additive. Producers under the SE profile **SHOULD** emit `verify` for behavioral and contract factlets. Consumers that do not check the block **MUST** preserve it on round-trip.
- **Breaking change?** No.

## §10 Open questions

- Should `verify` be **required** (not merely recommended) for `fact_type: rule` and `fact_type: contract` under the SE profile, enforced by profile validation — with a refuse-to-fabricate parity to `sources`? The argument for is that a behavioral rule with no oracle is unverifiable; the argument against is the authoring burden on existing factbooks.
- Standard `assertion` grammar: adopt an existing expression language (e.g. CEL) versus the minimal grammar above. Deferred pending usage data.
- `examples` provenance: when an example is harvested from a legacy test or a bug reproduction, should it carry a per-example source reference? This would tie each example to the coverage it represents.

## Cross-references

This RFC depends on or extends:

- **RFC 0002 — Profiles Mechanism** (base v0.2 spec): provides the profile-extension model under which the `verify` block is a software-engineering-profile addition.
- **RFC 0003 — Origination provenance block**: provides the `origination` field that records who produced the factlet, distinct from the `verify` block that records how to check it.
- **RFC 0005 — Software profile phase enum**: provides the `phase` value used in the worked example.
- **RFC 0008 — Applicability selector**: provides the top-level `applies_to` field that scopes *where* a `verify` block's check is evaluated. The `verify` block does not re-declare applicability; it inherits the factlet's `applies_to`.
- **RFC 0010 — Rationale and alternatives fields**: provides the `rationale` field shown in the worked example.
- **RFC 0011 — Attribution**: introduces the `attribution` block (`produced_by` / `authority`); its `produced_by.agent` corresponds to the `origination.authored_by` value defined by RFC 0003, pairing record-authorship with accountability. (RFC 0011 does not define `authored_by` itself.)

## Acceptance criteria

- Two maintainer approvals OR working-group consensus.
- `SPEC.md` (SE profile) §schema gains the `verify` block and a worked example; the interaction with `phase` is noted.
- `schema/profiles/software-engineering/factlet.schema.json` gains the `verify` object property.
- Reference SDK (`factlet-ai/reference-sdk`): (a) recognise and validate `verify` on read; (b) preserve it on round-trip; (c) fixtures for all three `kind` values; (d) a helper that selects and evaluates the oracle given a candidate; (e) a compiler stub that turns `assertion` and `examples` into an executable test.
- At least one registry example factbook annotated with a representative `verify` block per `kind`.
