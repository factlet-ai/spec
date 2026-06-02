# RFC 0008: Applicability selector (the `applies_to` block)

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-06-01
- **Last updated**: 2026-06-01
- **Discussion**: TBD
- **Target spec version**: v0.2
- **Scope**: project:factlet-ai

## §1 Summary

This RFC introduces an optional `applies_to` block on factlets that records **the technical surface a factlet governs** — path globs, source languages, and frameworks — as a machine-evaluable selector. It separates **where a claim was observed** (provenance, RFC 0003) from **where a claim applies** (`applies_to` — a forward-looking predicate over candidate artifacts).

In the v0.1 spec, applicability is implicit. A factlet's governing surface is inferred from free-text `tags`, from the evidence files it cites, or from the whole-factbook `scope`. None of these is a predicate a machine can evaluate against an arbitrary artifact to answer "does this factlet apply here?". The `applies_to` block makes that question decidable. The field is **scoped to factbooks declaring `profile: software-engineering`** (RFC 0005) and is not part of the base spec; other profiles define their own applicability vocabulary (manufacturing already does — see §2.4 and RFC 0006).

## §2 Motivation

The v0.1 spec scopes a whole factbook with `metadata.scope` (e.g. `scope: project:payments`) and records per-factlet evidence in its provenance block (RFC 0003). RFC 0005 adds `phase` (a *when-in-the-lifecycle* axis). None of these answers *which artifacts a factlet governs*. Three consequences are observable in cross-tool retrieval and conformance settings.

### 2.1 Injection-time retrieval over-injects across surfaces

A factlet stating "all HTTP handlers run inside the request-context middleware for tracing" governs a Python web tier. With no applicability predicate, retrieval surfaces it while an agent edits a TypeScript front-end file or a Terraform module, because relevance is computed from text similarity alone. The `phase` axis (RFC 0005) does not disambiguate here — the factlet is `implementation`-phase both in the surface it governs and in the surface it does not. The effect is that phase-agnostic and cross-tier factlets dominate the injected set, lowering relevance density and raising token spend on irrelevant context.

### 2.2 Conformance verification grades factlets against the wrong surface

A verification pass that asks "does this change satisfy factlet f002?" must first decide *whether f002 is even in scope for the change*. Without an applicability predicate, an implementation either (a) grades every factlet against every changed file — producing false scope-mismatch rejections when a Python convention is checked against a `.tsx` file — or (b) relies on free-text `tags` heuristics that diverge across implementations. Scope is asserted but not machine-selectable, so a verifier cannot cleanly answer "is this factlet's `verify` block applicable to this artifact?" before evaluating it.

### 2.3 Provenance is overloaded as both evidence and selector

Provenance fields (RFC 0003) list the files associated with a factlet. In practice they are read two ways at once: as **provenance** (the files from which the claim was distilled) and as an **ad-hoc selector** (the files the claim is assumed to govern). These are different relations. A convention learned from one exemplar file (`src/db/wrapper.py`) governs an entire directory (`src/db/**`); a decision recorded against a design doc governs code that does not yet exist. Conflating the two means widening provenance to express governance corrupts the evidence record, and keeping provenance precise leaves governance unexpressed. `applies_to` gives governance its own field and leaves provenance (RFC 0003) untouched.

### 2.4 Why scoped to the Software profile (not base)

Path globs, source languages, and frameworks are software-domain vocabulary. Other domains have analogous but structurally different applicability surfaces:

- **Manufacturing** (RFC 0006) scopes applicability spatially over the ISA-95 / B2MML hierarchy — `enterprise ⊃ site ⊃ area ⊃ line ⊃ cell ⊃ unit ⊃ asset`. A path glob is meaningless there; an asset identifier is meaningless in software.
- **Healthcare** would scope by care setting / specialty / patient cohort.
- **Legal** would scope by jurisdiction / matter-type / court.

Putting `paths`/`languages`/`frameworks` on the base factlet schema would enshrine the software applicability surface as universal — the conflation the Profiles mechanism (RFC 0002) is designed to prevent, and the same reasoning that scoped `phase` to the Software profile in RFC 0005. The base spec keeps only `layer_type` (the organizational layer, per RFC 0007); each profile defines its own *technical* applicability surface.

## §3 Block definition

A factlet **inside a Factbook declaring `profile: software-engineering`** **MAY** carry an `applies_to` object. It is **OPTIONAL**. A reader that does not understand it **MUST** preserve it on round-trip (read → write) without applying selection semantics, per v0.1 §9.3.

`applies_to` is a **conjunction of optional dimensions**. Each dimension constrains one axis; an omitted dimension is unconstrained on that axis. The three dimensions for v0.2:

| Dimension | Type | Match semantics |
|---|---|---|
| `paths` | array of glob strings | gitignore-style globs over repo-relative POSIX paths. `!`-prefixed patterns are negations (exclusions). An artifact path matches iff it matches at least one non-negated pattern and no negated pattern. |
| `languages` | array of strings | Case-insensitive identifier equality against the artifact's language (linguist-style identifiers, e.g. `python`, `typescript`, `go`). |
| `frameworks` | array of strings | Case-insensitive identifier equality against frameworks the consumer attributes to the artifact (e.g. `django`, `react`, `stripe-sdk`). |

## §4 Selection semantics (normative)

Let *A* be a candidate artifact (a file, or a change touching a file) and *F* a factlet carrying `applies_to`. **F applies to A** iff, for every dimension *present* in `applies_to`, *A* satisfies that dimension:

- `paths` present: *A*'s repo-relative path **MUST** match the glob set per the §3 table (at least one positive match, no negated match).
- `languages` present: the language the consumer attributes to *A* **MUST** be a case-insensitive member of the list.
- `frameworks` present: at least one framework the consumer attributes to *A* **MUST** be a case-insensitive member of the list.

The dimensions combine with **AND**; the list inside a dimension combines with **OR** (with `paths` negation as exclusion). A factlet **without** `applies_to` applies across the entire factbook `scope` — identical to v0.1 behavior. An `applies_to` present but empty (`{}`) is equivalent to absence and **SHOULD** be normalized to absence by producers.

**Path matching** **MUST** follow gitignore glob semantics (`*`, `**`, `?`, character classes, leading `!` negation, trailing `/` directory-only), which are well specified and portable. **Language and framework attribution is consumer-side and implementation-defined**: the spec does not mandate how a consumer determines an artifact's language or frameworks, only that *if* the consumer asserts them, matching is case-insensitive identifier equality. A consumer that cannot determine an axis treats the corresponding dimension as **unsatisfied** (the factlet does not apply on the strength of that dimension), never as a wildcard match — fail-closed, so an undetermined language never silently widens applicability.

## §5 Applicability is a selector, not a verdict (firewall)

`applies_to` decides *whether a factlet is in scope* for a retrieval or a verification pass. It **MUST NOT** be used as evidence of conformance. "F applies to A" never implies "A satisfies F." A verification implementation evaluates the `verify` block (RFC 0009) — or, absent one, a model judgment — *only over the set of factlets that apply*; applicability gates which checks run, it does not contribute to any check's outcome. The selection signal and the grading signal stay disjoint: applicability is a precondition for grading, never an input to it.

## §6 Schema fragment

Added to the software-engineering profile factlet schema (`schema/profiles/software-engineering/factlet.schema.json`), **NOT** to the base factlet schema:

```json
"applies_to": {
  "type": "object",
  "properties": {
    "paths": {
      "type": "array",
      "items": { "type": "string" },
      "description": "gitignore-style globs over repo-relative POSIX paths; '!'-prefixed entries are exclusions."
    },
    "languages": {
      "type": "array",
      "items": { "type": "string" },
      "description": "linguist-style language identifiers; case-insensitive match against the artifact's language."
    },
    "frameworks": {
      "type": "array",
      "items": { "type": "string" },
      "description": "framework identifiers; case-insensitive match against frameworks attributed to the artifact."
    }
  },
  "additionalProperties": false
}
```

All three properties are optional; an object with none present is equivalent to omission (see §4).

## §7 Worked example

```yaml
schema_version: v1.0
profile: software-engineering
profile_version: "0.2"
last_updated: 2026-06-01T00:00:00Z
metadata:
  name: payments-service
  scope: project:payments

content:
  - id: f001
    statement: "Refunds older than 90 days require manual ops approval, never auto-processed."
    confidence: 1.0
    sources: ["docs/refund-policy.md:8"]
    phase: design                       # RFC 0005 — orthogonal axis
    applies_to:
      paths: ["src/payments/**", "!src/payments/**/*_test.py"]
      languages: [python]
      frameworks: [stripe-sdk]
    tags: [payments, ops, compliance]

  - id: f002
    statement: "All HTTP handlers run inside the request-context middleware for tracing."
    confidence: 1.0
    sources: ["src/web/middleware.py:31"]
    phase: implementation
    applies_to:
      paths: ["src/web/**"]
      languages: [python]
    tags: [observability, web]

  - id: f003
    statement: "Money is represented as integer minor units, never floats, across the codebase."
    confidence: 1.0
    sources: ["docs/money.md:3"]
    # no applies_to — governs the whole factbook scope (project:payments)
    tags: [money, conventions]
```

Selection outcomes for a change touching `src/payments/refund.py` (Python, stripe-sdk):

- **f001 applies** — path matches `src/payments/**` and is not a `*_test.py`; language is `python`; framework `stripe-sdk` is attributed. It is in scope for retrieval, and its `verify` block (if present, per RFC 0009) is in scope for the verification pass.
- **f002 does not apply** — path does not match `src/web/**`.
- **f003 applies** — no `applies_to`, so it governs the whole `project:payments` scope.

The same change touching `src/web/api/routes.py` flips f001 (out) and f002 (in); f003 still applies. A change touching `web/ui/Cart.tsx` (TypeScript) leaves only f003 in scope — f001/f002 are Python-only.

## §8 Retrieval and verification semantics

- **Injection-time retrieval** (grounding before generation) **MAY** pre-filter candidate factlets to those whose `applies_to` matches the artifact(s) the agent is working on, then rank the survivors by existing relevance signals. This raises relevance density and lowers injected-token spend without changing ranking. A consumer that does not know the working surface requests retrieval without the filter — equivalent to v0.1 behavior.
- **Verification** (conformance after generation) **MUST** compute the applies-to set before grading and **MUST** restrict grading to factlets that apply to the artifact under review (§5). A factlet that does not apply is neither PASS nor FAIL for that artifact; it is *out of scope* and absent from the verdict.
- A coverage signal (e.g. the FactSignal of v0.1 §6), when computed against an applies-to-filtered set, reflects coverage of the in-scope factlets only. Implementations **SHOULD** document whether a reported coverage signal is scope-filtered or whole-factbook.

## §9 Relationship to `layer_type`, provenance, and `phase`

These are four distinct relations and this RFC keeps them distinct:

- **`layer_type`** (base spec, RFC 0007 layers): the *organizational* owner of the factlet — `Individual` / `Project` / `Team` / … . Answers "whose truth is this." A `Team`-layer factlet may carry a narrow `applies_to`; the two axes are independent.
- **provenance** (RFC 0003): where the claim was *observed / evidenced*. Point-in-time, backward-looking. `applies_to` is forward-looking governance. A factlet may cite one evidence file and govern a whole directory.
- **`phase`** (RFC 0005): *when in the lifecycle* the factlet is relevant. Orthogonal to *which surface* it governs. A factlet can be `implementation`-phase and path-scoped to `src/web/**` simultaneously, as in f002 above.

## §10 Alternatives considered

### A. Overload provenance as the selector

Widen the existing provenance file list to express the governing surface.

**Rejected because:** provenance is the evidence a claim was distilled from; governance is a different relation (§2.3). Overloading one field for both corrupts the evidence record the moment a claim governs more than its exemplar, and it cannot express language/framework axes or path negation at all.

### B. Free-form `tags` (`lang:python`, `path:src/payments`)

Producers could encode applicability in the open `tags` namespace without a schema change.

**Rejected because:** the tag namespace is unenforceable and implementations diverge on conventions (`lang:python` vs `language:py` vs `stack:python`), so consumers cannot filter reliably without normalization. This is the same rejection recorded for the `phase` enum (RFC 0005, Alternative B) and the Profiles mechanism (RFC 0002, Alternative B).

### C. Applicability fields in the base spec (universal)

Put `paths`/`languages`/`frameworks` on the base factlet schema.

**Rejected because:** these are software-domain surfaces. Manufacturing (RFC 0006) scopes applicability over the ISA-95 spatial hierarchy; healthcare over care setting; legal over jurisdiction. Enshrining software vocabulary in the universal base is the conflation the Profiles mechanism prevents — identical reasoning to RFC 0005's base-vs-profile decision.

### D. A full boolean expression language

Allow arbitrary nested AND/OR/NOT over predicates.

**Rejected for v0.2** because the observed cases are covered by a conjunction-of-dimensions with OR-lists and path negation; an arbitrary expression language is heavier to author, harder to validate, and harder to evaluate consistently across implementations. Forward-compatible: a future profile-version bump can introduce a richer grammar without breaking v0.2 readers (unknown keys are ignored per v0.1 §9.3).

### E. Regular expressions instead of globs for `paths`

Use regex for path matching.

**Rejected because:** globs are the lingua franca of path selection (gitignore, CODEOWNERS, tool config) and are what authors already know; regex is more error-prone, less portable across engines, and easy to write in a way that over- or under-matches paths silently. Glob semantics are bounded and well specified.

### F. A single `match` map keyed by axis name

Express dimensions as `match: { path: …, language: … }` rather than top-level keys.

**Rejected because:** the flat-key form (`paths`, `languages`, `frameworks`) reads identically to the surrounding factlet fields, validates more simply, and matches the precedent set by RFC 0005 (`phase` as a top-level field). The nesting buys no expressiveness.

## §11 Migration impact

- **For software-engineering Factbook authors**: no required change. Existing v0.1/v0.2 Factbooks contain no `applies_to`; their factlets govern the whole factbook `scope`, exactly as today. Authors opt into surface-scoped retrieval and verification by adding `applies_to` to factlets incrementally. No re-authoring required.
- **For the base spec**: none. This is a profile-scoped addition under `software-engineering`; the base schema is untouched.
- **For implementations**: no required change beyond software-profile schema validation (per RFC 0002 conformance). Implementations that expose surface-filtered retrieval or scope-restricted verification **MAY** consume the field when the Factbook declares the software-engineering profile. A verification implementation that consumes `applies_to` **MUST** honor §5 (selection, not verdict).
- **Breaking change?** No. Additive optional object within a registered profile.

## §12 Open questions

- Should `applies_to` support a `profiles`/`stacks` shorthand (a named bundle of path+language+framework constraints reused across factlets)? Deferred — v0.2 ships the explicit form; a named-bundle indirection can be added in a profile-version bump if duplication proves common.
- Should language/framework attribution be standardized (a registry of identifiers) rather than left implementation-defined? Deferred — v0.2 recommends linguist-style language identifiers and leaves frameworks open; a registry identifier list (`factlet-ai/registry`) can follow if divergence is observed.
- Should `paths` matching pin to a declared repo root when a factbook composes across repositories (RFC 0004 dependencies)? Deferred — for v0.2, paths are interpreted relative to the consuming repository's root; cross-repo applicability is an open item tracked with the composition story.

## §13 Cross-references

This RFC depends on or extends:

- **RFC 0002 — Profiles Mechanism**: provides the profile-extension model that scopes `applies_to` to the software-engineering profile rather than the base spec.
- **RFC 0007 — Factbook Layers v2 (DRAFT)**: provides the `layer_type` organizational axis (§9) that `applies_to` is independent of.
- **RFC 0003 — Origination provenance block**: provides the provenance relation (§2.3, §9) that `applies_to` is distinguished from.
- **RFC 0005 — Software profile + phase enum**: the host profile for `applies_to`, and the source of the orthogonal `phase` axis (§9).
- **RFC 0006 — Manufacturing profile (DRAFT)**: the contrasting profile whose spatial applicability surface (§2.4) motivates keeping applicability out of the base spec.
- **RFC 0009 — Verifiable assertions (the `verify` block)**: the verification block whose grading is gated by — but kept disjoint from — `applies_to` selection (§5, §8). The forward-reference inside the RFC 0009 `verify` block (`applies_to → RFC 0008`) resolves here.
- **RFC 0010 — Rationale + alternatives (BASE)** and **RFC 0011 — Attribution (BASE)**: adjacent factlet metadata blocks; orthogonal to `applies_to` and not gated by it.

## §14 Acceptance criteria

- Two maintainer approvals OR working group consensus.
- RFC 0002 (Profiles mechanism) accepted as a prerequisite; RFC 0005 (Software profile) is the host profile.
- Add `applies_to` to `schema/profiles/software-engineering/factlet.schema.json` and document selection semantics (§4, §5) in `profiles/software-engineering/SPEC.md`.
- Reference SDK (`factlet-ai/reference-sdk`) PR adding: (a) software-profile schema validation for `applies_to`; (b) a portable gitignore-glob path matcher; (c) an `applies_to(artifact)` predicate helper with the fail-closed undetermined-axis behavior (§4); (d) an optional applies-to filter parameter on retrieval; (e) test fixtures covering path-only, language-only, framework-only, combined-conjunction, negated-path, and no-`applies_to` factlets.
- At least one example factbook in `factlet-ai/registry` updated to declare `profile: software-engineering` and carry `applies_to` on at least one factlet (additive, not destructive).
- Conformance test asserting the firewall (§5): an `applies_to` match never alters a `verify`-block verdict.
