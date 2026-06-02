# Software Engineering Profile — Specification

**Status:** Stable (v0.2)
**Identifier:** `software-engineering`
**Profile version:** `0.2`
**Ratifying RFCs:** [RFC 0005](../../rfcs/0005-software-profile-phase-enum.md) (`phase`) · [RFC 0008](../../rfcs/0008-applicability-selector.md) (`applies_to`) · [RFC 0009](../../rfcs/0009-verifiable-assertions.md) (`verify`)
**Maintainer:** factlet-ai/spec working group

## Scope

The Software Engineering Profile applies to Factbooks describing software systems — codebases, APIs, services, libraries. It adds the `phase` field (phase-filtered retrieval: design / implementation / testing / runtime), the `applies_to` field (machine-evaluable applicability selection), and the `verify` field (verifiable oracles) on factlets.

## Activation

A Factbook activates this profile by declaring at root:

```yaml
profile: software-engineering
profile_version: "0.2"
```

A factlet within a different-profile Factbook may opt into this profile via per-factlet `profile: software-engineering` (cross-domain Factbook pattern, see RFC 0002 §1.2).

## Schema extensions

### Factlet — `phase` field

| Field   | Type   | Required | Description |
|---------|--------|----------|-------------|
| `phase` | string | No       | Software lifecycle phase. Enum: `design` / `implementation` / `testing` / `runtime`. Omitted = phase-agnostic. |

JSON Schema in [`factlet.schema.json`](factlet.schema.json).

### Phase semantics

- **`design`** — applies during architecture and design-decision phase (technology choices, system structure, API contracts).
- **`implementation`** — applies during code-writing phase (conventions, anti-patterns, library usage, naming).
- **`testing`** — applies during test authoring or test execution (harness conventions, test data, coverage rules).
- **`runtime`** — applies during production operation (alarm handling, incident response, operational thresholds, vendor quirks observed in production).

### Factlet — `applies_to` field

| Field        | Type   | Required | Description |
|--------------|--------|----------|-------------|
| `applies_to` | object | No       | The technical surface a factlet governs, as a machine-evaluable selector. Optional, AND-combined dimensions: `paths` (gitignore-style globs), `languages` (case-insensitive identifiers), `frameworks` (case-insensitive identifiers). An omitted dimension is unconstrained; an absent `applies_to` applies across the whole Factbook scope. |

Defined in [RFC 0008](../../rfcs/0008-applicability-selector.md). Selection is fail-closed: a consumer that cannot determine an axis treats that dimension as unsatisfied, never as a wildcard.

### Factlet — `verify` field

| Field    | Type   | Required | Description |
|----------|--------|----------|-------------|
| `verify` | object | No       | An oracle that makes the factlet machine-checkable. `kind` (REQUIRED) ∈ `assertion` / `test_ref` / `examples`, with exactly one matching oracle object: `assertion` (`expr` + `vars`), `test_ref` (`runner` + `path` + optional `selector`), or `examples` (a list of `given`/`expect` pairs). Optional `boundary`. |

Defined in [RFC 0009](../../rfcs/0009-verifiable-assertions.md). The `verify` block is a selector-independent oracle (it inherits the factlet's `applies_to`; see RFC 0009 §5 firewall — selection is not scoring).

JSON Schema for all three SE-profile field additions is in [`factlet.schema.json`](factlet.schema.json).

## Retrieval semantics

Implementations MAY filter retrieval by phase. Recommended consumer behavior: when scoping retrieval to a phase, include phase-agnostic factlets (factlets where `phase` is unset) in the result.

## Example

[`examples/payments-service.yaml`](examples/payments-service.yaml).

## Versioning

This profile follows the spec's pre-v1.0 versioning rules (any minor version may break before v1.0). Breaking changes to this profile bump `profile_version`.
