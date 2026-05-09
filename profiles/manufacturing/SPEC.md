# Manufacturing Profile — Specification

**Status:** Draft (v0.1)
**Identifier:** `manufacturing`
**Profile version:** `0.1`
**Ratifying RFC:** [RFC 0006](../../rfcs/0006-manufacturing-profile.md)
**Maintainer:** factlet-ai/spec working group

## Scope

The Manufacturing Profile applies to Factbooks describing physical production systems — sites, areas, production lines, work cells, equipment units, and individual assets. It bundles three foundational schema additions:

1. A **spatial scope hierarchy** (`mfg_scope`) modeled on ISA-95 / B2MML — seven levels: enterprise / site / area / line / cell / unit / asset.
2. An **asset-lifecycle phase** (`asset_phase`) — six values: fat / sat / commissioning / operation / maintenance / decommissioning.
3. A **safety + governance vocabulary** — `severity`, `permission_role`, `valid_until`, factlet_type extensions (`hazard`, `alarm-correlation`, `vendor-quirk`), and manufacturing-domain `origination.source_type` extensions.

The profile is appropriate for chemicals, pharma, food, automotive, semiconductor, discrete manufacturing, and process manufacturing factbooks — including regulated environments (cGMP, 21 CFR Part 11, EU Annex 11).

## Naming-collision avoidance (read first)

The profile uses `mfg_scope` (not `scope_level`, which is already defined by the base spec with different enum values) and `asset_phase` (not `phase`, which is used by the software-engineering profile with different enum values). These distinct names prevent BI/ETL pipeline collisions and base-spec contradictions. See [RFC 0006 §"Naming-collision note"](../../rfcs/0006-manufacturing-profile.md) for the rationale.

## Activation

A Factbook activates this profile by declaring at root:

```yaml
profile: manufacturing
profile_version: "0.1"
```

A factlet within a different-profile Factbook may opt into this profile via per-factlet `profile: manufacturing` (cross-domain Factbook pattern, see RFC 0002 §1.2). This is useful for industrial-software Factbooks that mix `software-engineering` factlets (about the MES / SCADA codebase) and `manufacturing` factlets (about the physical plant).

## Schema extensions

### Factlet — `mfg_scope` (required)

| Field       | Type   | Required | Description |
|-------------|--------|----------|-------------|
| `mfg_scope` | string | **Yes**  | One of: `enterprise` / `site` / `area` / `line` / `cell` / `unit` / `asset`. Hierarchy is strictly nested. |

`unit` (ISA-95 Part 2 equipment-execution context, e.g. reactor R-1A) sits between `cell` and `asset` (physical asset installed within a unit, e.g. Pump-03).

### Factlet — identifier fields (conditionally required)

| Field           | Type   | Required when                                            | Pattern |
|-----------------|--------|----------------------------------------------------------|---------|
| `enterprise_id` | string | `mfg_scope == enterprise`                                | `^[a-z][a-z0-9_-]{0,62}[a-z0-9]$\|^[a-z]$` (slug, not legal name; for long enterprise names tokenize and abbreviate) |
| `site_id`       | string | `mfg_scope` ∈ `{site, area, line, cell, unit, asset}`    | same |
| `area_id`       | string | `mfg_scope` ∈ `{area, line, cell, unit, asset}`          | same |
| `line_id`       | string | `mfg_scope` ∈ `{line, cell, unit}`; OR if asset-on-line  | same |
| `cell_id`       | string | `mfg_scope == cell`; OR if asset-in-cell or unit-in-cell | same |
| `unit_id`       | string | `mfg_scope == unit`; OR if asset-in-unit                 | same |
| `asset_id`      | string | `mfg_scope == asset`                                     | same |

**Asset case:** an `asset` factlet requires `site_id` + `area_id` + `asset_id` AND **at least one** of `{line_id, cell_id, unit_id}` to disambiguate the physical containment. Multiple of those three MAY be present (a pump installed in unit R-1A within line-1 within area api-synthesis carries all three).

**Lineage fields above the declared scope are permitted (informational).** A factlet at `mfg_scope: site` MAY carry `enterprise_id` (for cross-enterprise BI) without triggering a validation error.

### Factlet — `asset_phase` (optional)

| Field         | Type   | Required | Description |
|---------------|--------|----------|-------------|
| `asset_phase` | string | No       | Asset lifecycle phase. Enum: `fat` / `sat` / `commissioning` / `operation` / `maintenance` / `decommissioning`. Omitted = phase-agnostic. |

JSON Schema in [`factlet.schema.json`](factlet.schema.json).

### Phase semantics

- **`fat`** — Factory Acceptance Test, at vendor's facility before shipment (loop-tuning provisional values, vendor-confirmed P&IDs, OEM commissioning data).
- **`sat`** — Site Acceptance Test, on-site post-install (SAT-validated tuning values that supersede FAT, on-site safety verification, plant-specific calibration).
- **`commissioning`** — Post-SAT bring-up (process recipe validation, operator training runs, MOC pre-startup safety review).
- **`operation`** — Steady-state production (operating thresholds, alarm responses, SOPs for normal runs, shift-handover conventions, MOC-derived setpoint changes).
- **`maintenance`** — Scheduled or reactive maintenance (PM intervals, vendor-quirk workarounds, RCA findings, lockout-tagout procedures).
- **`decommissioning`** — Removal-from-service (safe-shutdown sequences, hazardous material handling, regulatory disposal documentation).

### Cross-cutting events: HAZOP, MOC, RCA — convention

- **HAZOP-derived factlets** use the `asset_phase` in which the finding is *currently active* (usually `operation`), not the phase in which it was first identified.
- **MOC-derived factlets** use `asset_phase: operation`. The MOC trail belongs in `sources` and `origination.source_ref`, NOT in the phase field.
- **RCA-derived factlets** use the phase of the *affected work going forward* — usually `operation` or `maintenance`.

### Factlet — safety vocabulary (optional)

| Field            | Type   | Required | Description |
|------------------|--------|----------|-------------|
| `severity`       | string | No       | `low` / `medium` / `high` / `critical`. |
| `permission_role`| string | No       | Minimum role: `operator` / `maintainer` / `engineer` / `supervisor` / `plant_manager` / `enterprise`. `enterprise` indicates corporate-tier visibility (auditors, compliance) and is parallel to — not a strict superset of — `plant_manager`; some shop-floor SOPs are `plant_manager`-visible but NOT `enterprise`-visible. |
| `valid_until`    | string | No       | ISO 8601 date after which factlet truth requires re-validation. |

### Factlet — `factlet_type` profile extensions

In addition to base-spec types, this profile registers:
- `hazard` — safety risk + mitigation rule. MUST carry `severity`. SHOULD carry `permission_role` — defaulting to `operator` for shop-floor-visible hazards, since hiding active hazards from the person at the console is itself a safety risk.
- `alarm-correlation` — empirical correlation between alarms or between alarm + operational state.
- `vendor-quirk` — documented vendor product behavior deviating from spec.

### Origination — `source_type` profile extensions

Per RFC 0003 §2.1, this profile registers:
- `opc-ua-server` — derived from OPC UA browse / read of a live PLC / SCADA server.
- `historian-extract` — derived from a historian query (PI System, Aspen IP.21, Ignition Tag History).
- `cmms-export` — derived from CMMS export (SAP PM, IBM Maximo, eMaint).
- `plc-tag-config` — derived from a PLC tag configuration export (TIA Portal, Studio 5000 ACD).

## Retrieval semantics

Implementations MAY filter retrieval by `asset_phase` and SHOULD support filtering by spatial scope. Recommended consumer behavior: when scoping retrieval to a spatial level (e.g. `area: api-synthesis`), include factlets at narrower scopes (line / cell / unit / asset) within that area, and include phase-agnostic factlets in any phase-scoped query. Role-aware consumers SHOULD filter by `permission_role` against the consumer's role. Consumers SHOULD warn when retrieving a factlet whose `valid_until` has passed.

## Example

[`examples/specialty-chemicals-plant.yaml`](examples/specialty-chemicals-plant.yaml).

## Versioning

This profile follows the spec's pre-v1.0 versioning rules (any minor version may break before v1.0). Breaking changes to this profile bump `profile_version`. Planned additions for follow-up RFCs documented in [RFC 0006 §"Deferred to follow-up RFCs"](../../rfcs/0006-manufacturing-profile.md).
