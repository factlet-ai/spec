# RFC 0004: Composable factbooks via `dependencies`

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-05-08
- **Last updated**: 2026-05-08
- **Discussion**: TBD
- **Target spec version**: v0.2

## Summary

Introduce an optional top-level `dependencies` block on Factbooks, with a `factbooks` sub-list, declaring other Factbooks the host Factbook composes against. A consumer evaluating retrieval against the host Factbook **MAY** include factlets from declared dependencies, subject to namespace and trust-prior rules. This addresses the v0.1 §13 open question on cross-Factbook references and enables a usage pattern observed across multiple implementations: project-specific Factbooks declaring composition against universal best-practice Factbooks (e.g. `security:owasp-top-10-2026`, `testing:pytest-conventions-2026`).

## Motivation

The v0.1 spec treats each Factbook as a standalone artifact. In practice, two patterns recur and are unaddressed:

### 1. Universal best-practice composition

A team's project Factbook captures truths specific to that project ("we use Stripe webhooks"; "Pump-03's threshold is 8.2 mm/s"). It does not — and should not — restate everything the team has learned about general best practices that apply across projects.

LLMs trained on the public internet are systematically wrong (or out-of-date) about:
- Language/framework best practices (Python 3.12+ patterns, React Hooks rules-of-hooks, etc.)
- Security guidance (OWASP top-10, secret-handling)
- Operational discipline (12-factor app, observability conventions)
- Test discipline (testing pyramid, common pytest patterns)

The remediation pattern that has emerged: a project Factbook declares it depends on a community-maintained Factbook (`best-practices:python-3.12-2026`, `security:owasp-top-10-2026`). Retrieval combines factlets from both. The composed answer is grounded in both project-specific truth and current community best practice.

### 2. Vendor-published fact distribution

Vendors publish authoritative Factbooks for their own products: a hypothetical `stripe-payments-factbook` documents canonical Stripe API behavior; `aws-s3-factbook` documents S3 quirks. Teams building on these services compose their project Factbook with the vendor's, getting up-to-date vendor-canonical truth without duplicating it locally.

### Why a top-level field is needed

The v0.1 spec offers no canonical mechanism to declare such relationships. Implementations have resorted to:
- Filename conventions (e.g. importing `*-factbook.yaml` in adjacent directories) — not portable across implementations.
- Concatenation at build time (gluing multiple Factbooks into one before publishing) — loses provenance and update independence.
- Per-factlet `extension` fields pointing to other Factbooks — works for one-off cross-references, but does not declare a Factbook-level dependency.

A top-level declaration solves all three: it is portable (every conforming implementation parses it), it preserves separable update cadences (each Factbook ships independently), and it is unambiguous about Factbook-to-Factbook composition.

## Detailed design

### 1. Field definition

A Factbook **MAY** carry a top-level `dependencies` object with a `factbooks` sub-list:

```yaml
dependencies:
  factbooks:
    - id: "best-practices:python-3.12-2026"
      version: "v1.2"
      source: "github.com/factlet-ai/registry/best-practices-python-3.12-2026"
      trust_prior: 0.9
      retrieval_mode: "merged"
    - id: "security:owasp-top-10-2026"
      version: "v1.0"
      source: "github.com/factlet-ai/registry/owasp-top-10-2026"
      trust_prior: 1.0
      retrieval_mode: "merged"
```

Each entry in `factbooks` has the following sub-fields:

| Sub-field         | Type    | Required | Description |
|-------------------|---------|----------|-------------|
| `id`              | string  | Yes      | Scoped identifier of the dependency. Conventionally `<scope>:<factbook-id>` per RFC 0001. |
| `version`         | string  | No       | Specific version of the dependency. If omitted, the consumer **SHOULD** resolve to the latest version available at retrieval time. |
| `source`          | string  | No       | URL or path to fetch the dependency. If omitted, the consumer relies on a registry resolver. |
| `trust_prior`     | float   | No       | Override trust prior for factlets pulled from this dependency. If omitted, the consumer uses the dependency's own factlet-level `origination.trust_prior` (RFC 0003) or implementation-defined default. |
| `retrieval_mode`  | string  | No       | One of `merged`, `fallback`, `disabled`. Default `merged`. See §3 below. |

### 2. Schema fragment

Added to `schema/factbook.schema.json` as an optional top-level property:

```json
"dependencies": {
  "type": "object",
  "properties": {
    "factbooks": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id":             { "type": "string" },
          "version":        { "type": "string" },
          "source":         { "type": "string" },
          "trust_prior":    { "type": "number", "minimum": 0.0, "maximum": 1.0 },
          "retrieval_mode": { "type": "string", "enum": ["merged", "fallback", "disabled"] }
        },
        "required": ["id"]
      }
    }
  }
}
```

### 3. Retrieval semantics

The `retrieval_mode` value controls how dependency factlets are combined with host factlets at retrieval time:

- **`merged`** (default): retrieval considers the union of host factlets and dependency factlets; ranking is implementation-defined but **MUST** preserve the source-Factbook attribution on each returned factlet (so the consumer can render "from project-x" vs "from best-practices:python").
- **`fallback`**: dependency factlets are considered only when host retrieval returns no results above an implementation-defined relevance threshold. Useful for "use community best practices only when project has no opinion."
- **`disabled`**: the dependency is recorded for documentation/tooling purposes but excluded from retrieval. Useful for capturing intent without yet adopting.

Implementations **MUST** report which Factbook each retrieved factlet came from (e.g. via the existing `id` field on the factlet, scoped per RFC 0001) so the consumer can render attribution and apply per-source policy.

### 4. Conflict resolution

When host and dependency Factbooks contain factlets that disagree:

- A conflict is detected when two factlets with overlapping retrieval relevance carry contradictory `statement` content.
- The protocol does **NOT** mandate a resolution algorithm. Implementations **MAY** apply trust_prior-weighted ranking, host-wins-on-tie, or surface both with conflict metadata.
- Implementations **SHOULD** document their conflict-resolution policy.

A future RFC may standardize a conflict-resolution interface if practice converges.

### 5. Cycle detection

Dependency declarations form a directed graph. Implementations **MUST** detect and report cycles (`A` depends on `B` depends on `A`); behavior on cycle detection is implementation-defined (raise an error; resolve via topological pruning; warn and continue).

### 6. Worked example

A project Factbook for a Python payments service:

```yaml
schema_version: v1.0
last_updated: 2026-05-08T00:00:00Z
metadata:
  name: payments-service
  owner: payments-team
  scope: project:payments
dependencies:
  factbooks:
    - id: "best-practices:python-3.12-2026"
      version: "v1.2"
      source: "github.com/factlet-ai/registry/best-practices-python-3.12-2026"
      retrieval_mode: "merged"
    - id: "security:owasp-top-10-2026"
      version: "v1.0"
      source: "github.com/factlet-ai/registry/owasp-top-10-2026"
      retrieval_mode: "merged"
    - id: "vendor:stripe-2026"
      source: "github.com/factlet-ai/registry/stripe-2026"
      retrieval_mode: "fallback"
content:
  - id: f001
    statement: "Refunds older than 90 days require manual ops approval."
    confidence: 1.0
    sources: ["docs/refund-policy.md:8"]
    phase: design
    tags: [payments, ops, compliance]
```

A retrieval query "how should we authenticate Stripe webhook callbacks?" against this Factbook may surface:

- `best-practices:python-3.12-2026:f014` ("verify HMAC signatures using `hmac.compare_digest`, not `==`")
- `security:owasp-top-10-2026:f003` ("never trust webhook payload claims; always verify signature first")
- `vendor:stripe-2026:f011` ("Stripe webhook signature is in the `Stripe-Signature` header; use `stripe.Webhook.construct_event` for verification") — pulled in `fallback` mode if the host has no relevant factlet.

The consumer renders each factlet with its source-Factbook scoping (per RFC 0001) so the user can see which source each piece of guidance came from.

### 7. Registry resolution

When a Factbook declares a dependency by `id` without a `source` field, the consumer needs a way to fetch the dependency. The protocol does not mandate a specific registry implementation. Common approaches:

- A well-known public registry (e.g. `factlet-ai/registry` repository).
- A team-internal registry server (URL configured per consumer).
- Filesystem co-location (the dependency Factbook is in a sibling directory).

Implementations **SHOULD** document which resolvers they support. The reference SDK (factlet-ai/reference-sdk) ships with a default resolver that fetches from `factlet-ai/registry`.

## Alternatives considered

### A. Inline (concatenate) dependencies at publish time

Tooling could glue multiple Factbooks into one before publishing.

**Rejected because:** loses provenance (consumer cannot tell which factlet came from which source); loses independent update cadence (host must be republished when any dependency updates); loses trust-prior separation (every factlet gets host's prior).

### B. Per-factlet cross-references only

Allow individual factlets to reference other Factbooks via `cross_ref` fields without a top-level dependency declaration.

**Rejected because:** the Factbook-level dependency is a different relationship — "this Factbook composes against these others" — not "this factlet refers to that factlet." Per-factlet cross-references are a separate concern (worth a future RFC if needed) and do not subsume the composition use case.

### C. Standardize a specific registry

Mandate `factlet-ai/registry` as the canonical resolver.

**Rejected because:** the protocol commitment is interoperability, not centralization. A team may run a private registry for internal Factbooks. The protocol only specifies the dependency declaration format; resolver discovery is left to implementations.

### D. Make `dependencies` a list of strings (id only)

Simpler shape: `dependencies: ["best-practices:python-3.12-2026", "security:owasp-top-10-2026"]`.

**Rejected because:** `version`, `source`, `trust_prior`, and `retrieval_mode` are useful per-dependency parameters that a string list cannot express. The object form is harder to author but more expressive; the empirical pattern from existing implementations is that all four fields are needed in practice.

## Migration impact

- **For Factbook authors**: no required change. Existing v0.1 Factbooks remain valid v0.2 Factbooks. Authors **MAY** add a `dependencies` block when composition is wanted.
- **For implementations**: parsing the block is required for v0.2 conformance. Honoring the `merged` / `fallback` / `disabled` retrieval modes is recommended; an implementation that does not yet support composition **MUST** at minimum surface a clear error to the consumer when a Factbook declares dependencies it does not resolve.
- **For registry maintainers**: factlet-ai/registry adds a resolution endpoint or convention so scoped IDs can be looked up without callers needing per-dependency `source` URLs.
- **Breaking change?** No. Additive optional field on the Factbook root.

## Open questions

- Should the spec mandate a maximum dependency-graph depth (e.g. transitively no more than 5 levels deep) to bound retrieval cost? Deferred — empirical pattern suggests 1-2 levels is typical, deeper graphs are rare.
- Should there be a standard `pinned` mode to mean "always use exactly this version, even if newer is available"? `version: "v1.2"` already provides this when the resolver respects it; an explicit `pin: true` flag may be redundant. Deferred for empirical observation.
- How does composition interact with the Low-FactSignal warning (v0.1 §7)? Open question: should FactSignal be computed against host-only factlets, the merged set, or both with separate scores reported? Initial recommendation: implementations **SHOULD** compute against the merged set (so dependency coverage counts as coverage); a consumer that wants host-only signal can request it explicitly.
- Conflict resolution algorithm — see §4. Deferred to a future RFC if practice diverges across implementations.

## Acceptance criteria

- Two maintainer approvals OR working group consensus
- Update to SPEC.md §5 (add a §5.4 covering `dependencies`) and §13 (remove "Cross-Factbook references" from open questions or mark resolved)
- Update to `schema/factbook.schema.json` (add the object property)
- Reference SDK (factlet-ai/reference-sdk) PR adding (a) parsing of the block, (b) a default registry resolver fetching from factlet-ai/registry, (c) `merged` / `fallback` / `disabled` retrieval modes, (d) cycle detection.
- factlet-ai/registry adds at least one community Factbook (e.g. a starter `best-practices:python-3.12-2026` with ~25 factlets) demonstrating the dependency target.
- factlet-ai/registry adds at least one example project Factbook demonstrating the dependency consumer.
