# RFC 0002: Profiles mechanism

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-05-08
- **Last updated**: 2026-05-08
- **Discussion**: TBD (link to GitHub Discussion thread when opened)
- **Target spec version**: v0.2

## Summary

Introduce a **Profiles mechanism** as the canonical way to extend the Factlet Protocol for specific domains (software engineering, manufacturing, healthcare, legal, others) without bloating the base spec with domain-specific vocabulary. A Factbook MAY declare a `profile` at the file level (and individual factlets MAY override at record level). Each registered profile maintains its own schema extensions in `factlet-ai/spec/profiles/<name>/`. v0.2 ratifies the **mechanism only**; concrete profiles ship as separate sub-RFCs (RFC 0005 Software Profile, future RFC 0006 Manufacturing Profile, etc.).

## Motivation

The Factlet Protocol v0.1 defines a domain-neutral base: a factlet is an atomic, source-cited truth, full stop. Empirical use across two domains (software engineering and manufacturing) has surfaced that some factlet attributes are intrinsically domain-specific:

- A **software** factlet meaningfully carries a lifecycle phase: design / implementation / testing / runtime.
- A **manufacturing** factlet meaningfully carries an ISA-95-aligned scope_level (`enterprise` / `site` / `area` / `work-cell` / `equipment`), a role-based permission_role (`operator` / `maintainer` / `engineer` / `supervisor` / `plant-manager`), and a hazard severity (per IEC 61882).
- A **healthcare** factlet would meaningfully carry HL7-FHIR resource references, a regulatory body (FDA / EMA / NICE), and PHI handling annotations.
- A **legal** factlet would meaningfully carry jurisdiction (state/federal/EU) and statute citations distinct from the existing free-form `sources` field.

Folding all of these into the base factlet schema produces three failure modes: (1) the base spec becomes unwieldy with many optional fields most consumers ignore; (2) cross-domain readers must understand vocabulary they will never use; (3) domain-specific producers cannot extend the schema without forking the spec.

Two prior approaches have been observed and rejected (see Alternatives, §A and §B): the v0.1 `extension` field (vendor-scoped, not domain-scoped) and free-form `tags` (no enforceable convention).

The Profiles mechanism preserves a small, domain-neutral base while enabling domain-scoped vocabulary that the protocol itself ratifies.

### Concrete observed need (2026-05)

A manufacturing pilot factbook (`docs/MIRA-FACTBOOK-STRAWMAN-MAY-2026.md`) authored against v0.1 added several manufacturing-specific factlet types (`hazard`, `alarm-correlation`, `vendor-quirk`) and additional fields (`scope_level`, `permission_role`, `related_factlets`). These do not belong in the base spec — they are not relevant to a Python project's factbook — but they are needed for any conforming manufacturing implementation to interoperate.

In parallel, a software engineering factbook needs a lifecycle `phase` field (RFC 0005). That field is meaningless in the manufacturing context, where the analogous lifecycle is `commissioning` / `operation` / `maintenance` / `decommissioning`.

Without a Profiles mechanism, both extensions would either land in the base spec (bloating it) or live as informal conventions (preventing interoperability across implementations).

## Detailed design

### 1. Field definition

#### 1.1 At Factbook file level

A Factbook MAY declare a `profile` field at the top level:

```yaml
schema_version: v1.0
profile: software-engineering
profile_version: "0.2"
content:
  - id: f001
    statement: "..."
```

The `profile` value is a string identifier from the registered-profile namespace (§3 below). The optional `profile_version` records which version of the profile's schema the Factbook is authored against.

If `profile` is omitted, the Factbook is interpreted as **profile-neutral**: only base-spec fields apply, and any profile-specific field is treated as an unknown key (preserved on round-trip per v0.1 §9.3).

#### 1.2 At individual factlet level

A factlet MAY override the file-level profile:

```yaml
profile: software-engineering
content:
  - id: f001
    statement: "Database queries longer than 500ms emit a slow-query metric."
    # inherits profile: software-engineering
  - id: f042
    profile: manufacturing
    statement: "Pump-03's vibration threshold is 8.2 mm/s."
    scope_level: equipment   # manufacturing-profile field
```

This supports cross-domain Factbooks (e.g. an industrial-software project that mixes software-engineering and manufacturing factlets in one repository).

### 2. Schema fragment

Added to `schema/factbook.schema.json` as optional top-level properties:

```json
"profile": {
  "type": "string",
  "description": "Identifier of the registered profile this Factbook is authored against. See factlet-ai/spec/profiles/ for registered profiles."
},
"profile_version": {
  "type": "string",
  "description": "Version of the profile schema this Factbook is authored against."
}
```

Added to `schema/factlet.schema.json`:

```json
"profile": {
  "type": "string",
  "description": "Override the file-level profile for this individual factlet. Used for cross-domain Factbooks."
}
```

### 3. Registered-profile namespace

Profiles are registered under `factlet-ai/spec/profiles/<name>/` in the spec repository. Each profile directory contains:

- `SPEC.md` — profile-specific normative spec (defines the additional fields, factlet types, and any constraints beyond the base)
- `factlet.schema.json` — JSON Schema for the profile's factlet extensions
- `factbook.schema.json` (optional) — JSON Schema for any Factbook-root extensions
- `examples/` — at least one worked example Factbook

The registered-profile list is maintained at `factlet-ai/spec/profiles/REGISTRY.md`. Each entry is added via a sub-RFC. Registered profiles as of v0.2 ship target:

- `software-engineering` — RFC 0005 (this v0.2 batch)
- `manufacturing` — future sub-RFC (timing aligned with manufacturing pilot)
- `healthcare`, `legal`, others — open for community proposal

### 4. Conformance

A v0.2-conforming implementation MUST:

1. Parse the optional `profile` and `profile_version` fields at Factbook root.
2. Parse the optional `profile` field on individual factlets.
3. If the implementation knows the named profile (its schema is included or available), it SHOULD apply profile-specific schema validation in addition to base validation.
4. If the implementation does not know the named profile, it MUST treat profile-specific fields as unknown keys (preserve on round-trip per v0.1 §9.3 conformance), and SHOULD log a warning identifying the unknown profile so the operator can decide whether to install profile support.
5. A Factbook with a `profile` field MUST NOT be rejected by a v0.2 reader on the basis of profile-specific fields the reader does not recognize.

This is forward-compatible: v0.1 readers ignore the `profile` field entirely (per existing v0.1 §9.3); v0.2 readers honor it; profile-aware v0.2 readers apply richer validation.

### 5. Worked example

A software-profile Factbook with an embedded manufacturing factlet:

```yaml
schema_version: v1.0
profile: software-engineering
profile_version: "0.2"
last_updated: 2026-05-08T00:00:00Z
metadata:
  name: my-iot-platform
  scope: project:my-iot-platform
content:
  - id: f001
    statement: "All HTTP handlers run inside the request-context middleware for tracing."
    confidence: 1.0
    sources: ["src/middleware.py:34"]
    phase: implementation     # software-profile field (RFC 0005)
    tags: [tracing, conventions]

  - id: f002
    profile: manufacturing
    statement: "Pump-03's vibration threshold is 8.2 mm/s."
    confidence: 1.0
    sources: ["RCA-2024-08", "PLC-export:Pump03-AlarmConfig.csv"]
    scope_level: equipment    # manufacturing-profile field (future sub-RFC)
    permission_role: maintainer
    factlet_type: alarm-correlation
```

Both factlets coexist under one Factbook. A reader knowing only the software profile validates `f001` against software-profile schema and treats `f002` as base-spec-with-unknown-keys. A reader knowing both profiles validates each appropriately.

### 6. Profile registration process

A new profile is added via a sub-RFC against this spec repo. The RFC MUST:

1. Justify why the domain warrants a separate profile (vs. extending an existing one or staying in base).
2. Define the additional fields and factlet types in normative form.
3. Provide JSON Schema fragments.
4. Provide at least one worked example Factbook.
5. Demonstrate that base-spec fields are preserved with their original semantics.

Profile RFCs are reviewed against the same maintainer-approval criteria as any other spec RFC.

### 7. Profile evolution

A registered profile may version independently of the base spec via its `profile_version` field. Breaking changes within a profile follow the same pre-v1.0 rules as the base spec (any minor version may break before v1.0). A profile MUST document its own versioning policy in its `SPEC.md`.

## Alternatives considered

### A. Use the existing v0.1 `extension` field

The v0.1 spec (§3.2) already defines an `extension` object for vendor-specific fields. Domain-specific fields could go there.

**Rejected because:** `extension` is documented as "Vendor-specific fields. Implementations MUST ignore unknown keys." Two problems: (1) domain extensions are meant to be **shared across vendors** of the same domain (every manufacturing implementation needs the same `scope_level` semantics), so vendor-scoping is the wrong axis; (2) "MUST ignore unknown keys" is the opposite of the desired behavior — profile-aware readers MUST validate, not ignore.

### B. Use free-form `tags` for domain markers

Producers could tag factlets with `tags: [profile:manufacturing, scope:equipment]` etc.

**Rejected because:** tags are open-namespace and unenforceable. Two implementations diverge on conventions (`profile:manufacturing` vs `domain:mfg` vs `category:industrial`), and consuming readers cannot reliably filter or validate. Same failure mode as the rejected alternative for RFC 0005's phase enum.

### C. Domain forks of the spec

Each domain could fork `factlet-ai/spec` into `factlet-ai/spec-manufacturing`, `factlet-ai/spec-software`, etc.

**Rejected because:** loses the small-shared-base benefit. A team running a healthcare service that includes a software platform would need two separate Factbook formats. Cross-domain Factbooks (§5 example) become impossible. Conformance testing fragments. The whole protocol becomes "many small unrelated specs" rather than "one base + composable profiles."

### D. Inline domain extensions in the base spec

Add `phase`, `scope_level`, `permission_role`, etc. directly to the base factlet schema and let producers populate what's relevant.

**Rejected because:** (1) the base spec balloons with fields most consumers don't use; (2) cross-domain readers must understand vocabulary they will never use; (3) adding a new domain becomes a base-spec change requiring full RFC + breaking-version cadence rather than a sub-RFC under an established mechanism; (4) the spec loses its "small enough for one read" property documented as a v0.1 design goal.

## Migration impact

- **For Factbook authors**: no required change. Existing v0.1 Factbooks remain valid v0.2 Factbooks (treated as profile-neutral). Authors who want profile-specific validation MAY add a `profile` declaration.
- **For implementations**: must parse `profile` and `profile_version` fields without error per §4. Honoring profile-specific schema validation is recommended but not required for base v0.2 conformance.
- **For domain spec authors**: a new profile is added via sub-RFC under `factlet-ai/spec/profiles/<name>/` rather than as a base-spec change.
- **Breaking change?** No. Additive optional fields. v0.1 readers ignore the new fields per existing v0.1 §9.3 conformance.

## Open questions

- Should a Factbook be allowed to declare **multiple** profiles at the file level (`profiles: [software-engineering, manufacturing]`)? Deferred. Initial design uses a single file-level profile with per-factlet overrides for the cross-domain case (§1.2). If empirical use shows multi-profile Factbooks are common, a future RFC can introduce a list-valued field; the current scalar form is forward-compatible.
- How are profile-version compatibility ranges expressed when a Factbook depends on another (RFC 0004)? Initial recommendation: dependency declarations (RFC 0004) MAY specify `profile` and `profile_version` constraints; if they don't, the consumer applies defaults from the dependency Factbook itself. A future RFC may formalize semver-style constraint syntax.
- Does the profile mechanism interact with the FactSignal callback (v0.1 §7)? Initial recommendation: FactSignal is computed against the retrieved factlet set regardless of profile; profile-specific scoring weights are an implementation choice. A future RFC may standardize a profile-aware scoring extension.

## Acceptance criteria

- Two maintainer approvals OR working group consensus
- Update to `SPEC.md`: new §15 "Profiles" defining the mechanism; update §13 (open questions) to mark "Cross-Factbook references" as resolved by RFC 0004 and add Profiles to the resolved list.
- Create `factlet-ai/spec/profiles/REGISTRY.md` (initially empty / template).
- Create `factlet-ai/spec/profiles/.template/` directory with a starter `SPEC.md`, `factlet.schema.json`, `factbook.schema.json`, `examples/` for new profile authors.
- Update `schema/factbook.schema.json` and `schema/factlet.schema.json` (add the `profile` properties).
- Reference SDK (factlet-ai/reference-sdk) PR adding (a) parsing of `profile` and `profile_version` fields, (b) a profile-registry resolver that loads profile schemas if installed, (c) graceful warning for unknown profiles, (d) test fixtures covering profile-neutral, profile-aware, and cross-profile Factbooks.
- RFC 0005 (Software Profile — phase enum) accepted concurrently as the first registered profile, demonstrating the mechanism end-to-end.
