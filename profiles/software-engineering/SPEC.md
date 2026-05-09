# Software Engineering Profile — Specification

**Status:** Stable (v0.2)
**Identifier:** `software-engineering`
**Profile version:** `0.2`
**Ratifying RFC:** [RFC 0005](../../rfcs/0005-software-profile-phase-enum.md)
**Maintainer:** factlet-ai/spec working group

## Scope

The Software Engineering Profile applies to Factbooks describing software systems — codebases, APIs, services, libraries. It adds the `phase` field on factlets to enable phase-filtered retrieval (design / implementation / testing / runtime).

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

## Retrieval semantics

Implementations MAY filter retrieval by phase. Recommended consumer behavior: when scoping retrieval to a phase, include phase-agnostic factlets (factlets where `phase` is unset) in the result.

## Example

[`examples/payments-service.yaml`](examples/payments-service.yaml).

## Versioning

This profile follows the spec's pre-v1.0 versioning rules (any minor version may break before v1.0). Breaking changes to this profile bump `profile_version`.
