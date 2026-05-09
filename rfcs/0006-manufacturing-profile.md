# RFC 0006: Manufacturing Profile — spatial-scope hierarchy, asset-lifecycle phase, hazard vocabulary

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-05-09
- **Last updated**: 2026-05-09
- **Discussion**: TBD
- **Target spec version**: v0.2 (second registered profile under the Profiles mechanism — RFC 0002, after the Software Engineering profile — RFC 0005)

## Summary

Define the **Manufacturing Profile** as the second registered profile under the Profiles mechanism (RFC 0002). The v0.1 profile bundles three foundational additions:

1. A **spatial scope hierarchy** — `mfg_scope` enum plus conditionally-required identifier fields, modeled on the ISA-95 / B2MML hierarchy widely used in shop-floor architecture. Seven levels: `enterprise / site / area / line / cell / unit / asset`.
2. An **`asset_phase` field** with manufacturing asset-lifecycle vocabulary (`fat / sat / commissioning / operation / maintenance / decommissioning`).
3. A **safety vocabulary**: `severity` enum, `permission_role` enum, `factlet_type` extensions (`hazard`, `alarm-correlation`, `vendor-quirk`), `valid_until` time-bound field, and manufacturing-domain `origination.source_type` extensions.

The Manufacturing Profile is **scoped to factbooks declaring `profile: manufacturing`** and is not part of the base spec. Other extensions surfaced during authoring (per-batch lifecycle, vendor-product field, B2MML/SAP interop appendix, CMMS URI scheme convention) are deferred to follow-up RFCs that need their own design cycle. This RFC delivers what is load-bearing for authoring a realistic regulated-manufacturing factbook in v0.1.

### Naming-collision note (read this first)

Two field names in this profile are **deliberately distinct from same-purpose fields elsewhere** to prevent BI/ETL pipeline collisions and base-spec contradictions:

- `mfg_scope` (not `scope_level`) — the base spec already uses `scope_level` with a different enum (`team / project / feature / personal`). Reusing the field name would create silent semantic conflict for readers that validate against both base + profile.
- `asset_phase` (not `phase`) — the software-engineering profile (RFC 0005) uses `phase`. Reusing the name in manufacturing would force downstream BI tooling to know per-row profile context to disambiguate enum semantics. A column-level filter "show me all factlets where phase=runtime" would silently mix software-runtime and manufacturing-asset-without-`runtime`-value semantics.

The cost is asymmetry between profiles. The benefit is that a Snowflake/BigQuery query against a multi-domain Factbook table can filter on the right field unambiguously.

## Motivation

Manufacturing factbooks describe physical systems — sites, production lines, individual assets — and the operating knowledge that surrounds them. Authoring a 20-factlet strawman against a fictional specialty-chemicals plant (Acme Plant 2 PA, see worked example below) and reviewing it with two senior manufacturing-domain experts (controls engineering + IT/OT integration) surfaced three findings about why the base spec is insufficient on its own for this domain.

### Finding 1 — Without spatial scope, factlets are ambiguous

The factlet *"C-101 oil-mist alarm firing within 90 seconds of NBIRTH publication is 87% predictive of compressor not having completed preheat"* is unintelligible to a downstream consumer (operator, MES integration agent, RCA tool) without knowing **where C-101 is** in the plant's structure. C-101 might be one of several compressors across multiple lines and areas; the factlet's relevance to a query is gated on whether the querying agent is operating in scope of that line.

In software engineering this problem is implicit because the file path supplies the structural context — `src/payments/stripe_client.py` is unambiguous within a repository. Manufacturing has no analogous universal addressing scheme; the community-adopted convention is the **ISA-95 / B2MML hierarchy** (enterprise → site → area → line → cell → unit → asset), which this RFC adopts as the spatial scope vocabulary. The `unit` level (ISA-95 Part 2 equipment-execution context) sits between `cell` and `asset` to support batch-process manufacturing where reactor R-1A is a *unit* (the equipment that executes a recipe phase) and Pump-03 is an *asset* installed within that unit — a distinction that is operationally load-bearing in chemicals and pharma plants. v0.1 includes the seven-level hierarchy.

### Finding 2 — Asset lifecycle drives factlet relevance

The same physical asset has **different operating knowledge across its lifecycle**:

- A factlet about loop-tuning constants for Reactor R-1A is highly relevant during **fat** (vendor-site Factory Acceptance Test) and **sat** (on-site Site Acceptance Test) — but the SAT-validated value is what runs in production, while the FAT-validated value is provisional.
- A factlet about an alarm-correlation pattern for Compressor C-101 is highly relevant during **operation** and **maintenance** (when the alarm fires or maintenance is investigating it), less relevant during **commissioning** (the alarm hasn't yet been observed enough times to correlate).
- A factlet about safe-shutdown SOPs for Reactor R-1A is highly relevant during **decommissioning**, and during emergencies in **operation**.

Without a phase signal, retrieval injects all factlets uniformly across the asset's lifecycle, producing noisy context. With a phase signal, an MES integration agent doing a commissioning checklist for a new pump can request retrieval scoped to `asset_phase: commissioning` plus phase-agnostic factlets and skip the operational-correlation noise.

This mirrors RFC 0005's lifecycle-phase concept for software-engineering, with manufacturing-appropriate enum values. Six values rather than four because FAT (factory acceptance test, at the vendor's site) and SAT (site acceptance test, on-site post-install) are distinct phases — they happen at different physical locations, and the factlets they produce have different evidentiary weight (SAT-validated values typically supersede provisional FAT values).

### Finding 3 — Safety vocabulary cannot be deferred for regulated industries

Without `severity`, `permission_role`, `factlet_type: hazard`, and `valid_until`, the v0.1 profile is **unusable in regulated process manufacturing** (chemicals, pharma, food, semiconductor with cGMP / 21 CFR Part 11 / EU Annex 11 requirements). The worked example uses a specialty-chemicals plant with FDA-IND-validated reactor sequences; deferring this vocabulary would ship a profile that cannot describe its own example domain.

v0.1 includes:
- `severity: low | medium | high | critical` for hazard factlets and operational thresholds.
- `permission_role: operator | maintainer | engineer | supervisor | plant_manager | enterprise` for shop-floor visibility scoping (operators see operating ranges, only QA sees deviation history).
- `factlet_type` extensions: `hazard`, `alarm-correlation`, `vendor-quirk`.
- `valid_until` (ISO 8601 date) for time-bound or vendor-version-bound factlets.

### Why these three together (and not split across RFCs)

The spatial-scope hierarchy is the **load-bearing addition** for manufacturing — every manufacturing factlet has a spatial scope, and downstream consumers cannot reason about a factlet without it. Phase is a parallel useful filter, but is meaningless without spatial scope (you cannot scope `asset_phase: commissioning` retrieval without first knowing which asset is being commissioned). The safety vocabulary makes the profile actually usable in the only example domain anyone will validate against.

Splitting these across three RFCs would ship a v0.1 profile that nobody can use in production and create unnecessary back-and-forth as the deferred-items RFCs land. Pulling the safety vocabulary forward is the minimum scope at which v0.1 is usable for the regulated-manufacturing domains the worked example targets.

The deferred items (per-batch lifecycle, vendor-product field, B2MML/SAP interop appendix, CMMS URI scheme) genuinely need their own design cycles — they raise design questions (e.g. how does batch-phase compose with asset-phase? what URI scheme works across SAP PM / Maximo / eMaint?) that require dedicated discussion, not bundling.

## Detailed design

### 1. Profile registration

This RFC registers the **Manufacturing Profile** under `factlet-ai/spec/profiles/manufacturing/` per the Profiles mechanism in RFC 0002.

- Profile identifier: `manufacturing`
- Profile version (initial): `0.1`
- Profile maintainer: factlet-ai/spec working group

### 2. `mfg_scope` field (required)

A factlet **inside a Factbook declaring `profile: manufacturing`** MUST carry a `mfg_scope` field. Its value MUST be one of:

- `enterprise` — applies across the company (multiple sites)
- `site` — applies plant-wide at one physical location
- `area` — applies to one production area within a site (e.g. `api-synthesis`)
- `line` — applies to one production line within an area (e.g. `line-1`)
- `cell` — applies to one work cell within a line (e.g. `dosing-cell-2`)
- `unit` — applies to one ISA-95-Part-2 equipment unit (e.g. reactor `r-1a`, the equipment that executes a recipe phase)
- `asset` — applies to one specific physical asset installed within a unit, cell, or line (e.g. pump `pump-03`)

The hierarchy is **strictly nested**: `enterprise ⊃ site ⊃ area ⊃ line ⊃ cell ⊃ unit ⊃ asset`. Note that `unit` is the ISA-95 Part 2 equipment-execution context (reactor, vessel, dryer); `asset` is a physical piece of equipment installed within a unit (pump, valve, transmitter). Collapsing `unit` and `asset` into one level breaks batch-process manufacturing where this distinction is operationally load-bearing.

Conformance: a v0.2 reader handling a `profile: manufacturing` Factbook MUST reject a factlet whose `mfg_scope` is not in the enum above. (Per the base spec §15.3 conformance rule, this validation only fires when the reader knows the profile.)

### 3. Identifier fields (conditionally required)

A factlet declares its position in the hierarchy with the identifier corresponding to its `mfg_scope` AND all higher-level (less-specific) identifiers in its lineage:

| `mfg_scope`   | Required identifiers                                                       |
|---------------|----------------------------------------------------------------------------|
| `enterprise`  | `enterprise_id`                                                            |
| `site`        | `enterprise_id` (optional), `site_id`                                      |
| `area`        | `site_id`, `area_id`                                                       |
| `line`        | `site_id`, `area_id`, `line_id`                                            |
| `cell`        | `site_id`, `area_id`, `line_id`, `cell_id`                                 |
| `unit`        | `site_id`, `area_id`, `line_id` (or `cell_id` if cell-scoped), `unit_id`   |
| `asset`       | `site_id`, `area_id`, `asset_id` AND at least one of `{line_id, cell_id, unit_id}` |

`enterprise_id` is required at `enterprise` scope so that two factlets at enterprise scope from different companies are distinguishable in cross-enterprise BI contexts. At `site` scope and below, `enterprise_id` is optional.

The `asset` case allows asset-on-line, asset-in-cell, and asset-in-unit topologies via the `anyOf` rule. Multiple of `{line_id, cell_id, unit_id}` MAY be present (a pump installed in unit R-1A within line-1 within area api-synthesis would carry all three).

**Lineage fields above the declared scope are permitted (informational, not validated).** A factlet at `mfg_scope: site` MAY carry `enterprise_id` (for cross-enterprise BI) without triggering a validation error.

All identifier fields are `string`, lowercase kebab-case, MUST match `^[a-z][a-z0-9_-]{0,62}[a-z0-9]$|^[a-z]$` (must start with a letter, matches the SDK's `_PROFILE_NAME_RE` for cross-system safety).

### 4. `asset_phase` field (optional)

A factlet **inside a Factbook declaring `profile: manufacturing`** MAY carry an `asset_phase` field. If present, its value MUST be one of:

- `fat` — Factory Acceptance Test, at vendor's facility before shipment (loop-tuning provisional values, vendor-confirmed P&IDs, OEM commissioning data)
- `sat` — Site Acceptance Test, on-site post-install (SAT-validated tuning values that supersede FAT, on-site safety verification, plant-specific calibration)
- `commissioning` — Post-SAT bring-up (process recipe validation, operator training runs, MOC pre-startup safety review)
- `operation` — Steady-state production (operating thresholds, alarm responses, SOPs for normal runs, shift-handover conventions, MOC-derived setpoint changes — see "Cross-cutting events" below)
- `maintenance` — Scheduled or reactive maintenance (PM intervals, vendor-quirk workarounds, RCA findings, lockout-tagout procedures)
- `decommissioning` — Removal-from-service (safe-shutdown sequences, hazardous material handling, regulatory disposal documentation)

The `asset_phase` field MAY be omitted, in which case the factlet is interpreted as **phase-agnostic within the manufacturing profile** (relevant across the asset lifecycle). The default is omission, not a sentinel value.

#### Cross-cutting events: HAZOP, MOC, RCA — how to phase

HAZOP studies, Management of Change (MOC), and Root Cause Analyses are **events**, not phases — they cut across the lifecycle. Convention:

- **HAZOP-derived factlets** (hazard identification, mitigation rules) are typically authored during `commissioning` (initial HAZOP) and re-validated during `operation` (post-MOC re-HAZOP). Use the asset_phase in which the HAZOP finding is *currently active* (usually `operation`), not the phase in which it was *first identified*.
- **MOC-derived factlets** (setpoint changes, threshold adjustments, procedure updates after MOC approval) are operational gospel — `asset_phase: operation`. The MOC trail (reference, approval date, change rationale) belongs in `sources` and `origination.source_ref`, NOT in the phase field.
- **RCA-derived factlets** (post-incident learnings) take the phase of the *affected work going forward* — usually `operation` or `maintenance`, not the phase in which the incident occurred.

This convention prevents the trap of putting a vibration-trip-threshold change derived from RCA-2024-08 + MOC-2024-019 under `asset_phase: maintenance` when the changed threshold is what now governs steady-state operation.

### 5. Safety vocabulary

#### 5.1 `severity` (optional)

A factlet **inside a Factbook declaring `profile: manufacturing`** MAY carry a `severity` field. Enum: `low | medium | high | critical`. Use:
- `critical` — life-safety, regulated-environmental release, plant-wide shutdown risk
- `high` — single-line shutdown risk, regulatory finding risk, near-flash / near-miss events
- `medium` — quality deviation, single-asset failure
- `low` — informational thresholds, optimization opportunities

#### 5.2 `permission_role` (optional)

Enum: `operator | maintainer | engineer | supervisor | plant_manager | enterprise`. Indicates the minimum role required to view the factlet in role-aware contexts (regulated environments, contractor segregation). Implementations MAY enforce by hiding factlets above the consumer's role. The protocol does not mandate enforcement; the contract is that the field is preserved on round-trip per RFC 0002.

#### 5.3 `factlet_type` extensions

In addition to the base-spec types (`definition`, `decision`, `convention`, `anti-pattern`, `tenet`, `sop`, `format-rule`, `enum-list`, `constraint`, `state-machine`, `example`), the manufacturing profile registers:

- `hazard` — a safety risk + mitigation rule (HAZOP-derived). MUST carry `severity`. SHOULD carry `permission_role` if scoped to a specific role.
- `alarm-correlation` — an empirical correlation between alarms or between alarm + operational state, useful for RCA and alarm-rationalization.
- `vendor-quirk` — a documented vendor product behavior that deviates from spec, requires a workaround, or constrains design choices.

#### 5.4 `valid_until` (optional, ISO 8601 date)

A factlet MAY carry `valid_until: <ISO 8601 date>` indicating the date after which the factlet's truth is no longer guaranteed without re-validation. Use cases:
- Vendor-quirk factlets that become stale when the vendor fixes the bug (`valid_until: "2026-12-31"` if the next firmware release is committed-fixed).
- FDA / EMA-validated configurations that require periodic re-validation (`valid_until: "2027-06-30"` for a 5-year cGMP re-validation cycle).
- Calibration certificates with an expiry date.

Implementations SHOULD warn (not reject) on retrieval of factlets past their `valid_until`. A future profile-version may add `superseded_by_condition` for non-date-based validity (e.g. "valid until next firmware release"); v0.1 keeps the date-only form for simplicity.

#### 5.5 Manufacturing-domain `origination.source_type` extensions

RFC 0003 §2.1 permits profile-scoped extensions to the base `source_type` enum. The manufacturing profile registers:

- `opc-ua-server` — factlet derived from an OPC UA browse / read of a live PLC / SCADA server.
- `historian-extract` — factlet derived from a historian query (PI System, Aspen IP.21, Ignition Tag History).
- `cmms-export` — factlet derived from CMMS (computerized maintenance management system) export — SAP PM, IBM Maximo, eMaint, etc.
- `plc-tag-config` — factlet derived from a PLC tag configuration export (TIA Portal export, Studio 5000 ACD parse).

These are additive to the base RFC 0003 enum (`manual / llm / import / forward-pass / reverse-pass`); a v0.2 reader unaware of the manufacturing profile will treat them as profile-extension values per RFC 0003 §2.1 (warn + preserve, do not reject).

### 6. Schema fragment

Added to `schema/profiles/manufacturing/factlet.schema.json` (NOT to base `schema/factlet.schema.json`):

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://factlet.ai/schema/profiles/manufacturing/v0.1/factlet.schema.json",
  "title": "Manufacturing Profile — Factlet schema additions",
  "type": "object",
  "properties": {
    "mfg_scope": {
      "type": "string",
      "enum": ["enterprise", "site", "area", "line", "cell", "unit", "asset"]
    },
    "enterprise_id": { "type": "string", "pattern": "^[a-z][a-z0-9_-]{0,62}[a-z0-9]$|^[a-z]$" },
    "site_id":       { "type": "string", "pattern": "^[a-z][a-z0-9_-]{0,62}[a-z0-9]$|^[a-z]$" },
    "area_id":       { "type": "string", "pattern": "^[a-z][a-z0-9_-]{0,62}[a-z0-9]$|^[a-z]$" },
    "line_id":       { "type": "string", "pattern": "^[a-z][a-z0-9_-]{0,62}[a-z0-9]$|^[a-z]$" },
    "cell_id":       { "type": "string", "pattern": "^[a-z][a-z0-9_-]{0,62}[a-z0-9]$|^[a-z]$" },
    "unit_id":       { "type": "string", "pattern": "^[a-z][a-z0-9_-]{0,62}[a-z0-9]$|^[a-z]$" },
    "asset_id":      { "type": "string", "pattern": "^[a-z][a-z0-9_-]{0,62}[a-z0-9]$|^[a-z]$" },
    "asset_phase": {
      "type": "string",
      "enum": ["fat", "sat", "commissioning", "operation", "maintenance", "decommissioning"]
    },
    "severity": { "type": "string", "enum": ["low", "medium", "high", "critical"] },
    "permission_role": { "type": "string", "enum": ["operator", "maintainer", "engineer", "supervisor", "plant_manager", "enterprise"] },
    "valid_until": { "type": "string", "format": "date" }
  },
  "required": ["mfg_scope"],
  "allOf": [
    { "if": { "properties": { "mfg_scope": { "const": "enterprise" } }, "required": ["mfg_scope"] },
      "then": { "required": ["enterprise_id"] } },
    { "if": { "properties": { "mfg_scope": { "const": "site" } }, "required": ["mfg_scope"] },
      "then": { "required": ["site_id"] } },
    { "if": { "properties": { "mfg_scope": { "const": "area" } }, "required": ["mfg_scope"] },
      "then": { "required": ["site_id", "area_id"] } },
    { "if": { "properties": { "mfg_scope": { "const": "line" } }, "required": ["mfg_scope"] },
      "then": { "required": ["site_id", "area_id", "line_id"] } },
    { "if": { "properties": { "mfg_scope": { "const": "cell" } }, "required": ["mfg_scope"] },
      "then": { "required": ["site_id", "area_id", "line_id", "cell_id"] } },
    { "if": { "properties": { "mfg_scope": { "const": "unit" } }, "required": ["mfg_scope"] },
      "then": { "required": ["site_id", "area_id", "line_id", "unit_id"] } },
    { "if": { "properties": { "mfg_scope": { "const": "asset" } }, "required": ["mfg_scope"] },
      "then": { "required": ["site_id", "area_id", "asset_id"],
                "anyOf": [ { "required": ["line_id"] }, { "required": ["cell_id"] }, { "required": ["unit_id"] } ] } }
  ]
}
```

### 7. Worked example

See `profiles/manufacturing/examples/specialty-chemicals-plant.yaml` (sanitized, illustrates all seven scope levels + all six asset_phases + safety vocabulary).

### 8. Retrieval semantics

Implementations targeting the manufacturing profile MAY filter by `asset_phase` and SHOULD support filtering by spatial scope. Recommended consumer behavior:

- An MES or shop-floor agent that knows the current asset/area context (e.g. an alarm-response surface scoped to a specific compressor) SHOULD request retrieval scoped to that asset's hierarchy lineage (`asset_id` + `unit_id`/`line_id` + `area_id` + `site_id`) plus phase-agnostic factlets at those higher scopes.
- An RCA tool examining a maintenance event SHOULD request `asset_phase: maintenance` plus phase-agnostic factlets in scope.
- A role-aware consumer SHOULD filter by `permission_role` against the consumer's role (e.g. an operator UI hides factlets requiring `engineer` or higher).
- A consumer SHOULD warn when retrieving a factlet whose `valid_until` has passed.

### 9. SDK conformance work required

The Manufacturing Profile is the FIRST profile to exercise the cross-profile validator-dispatch path. The reference SDK (factlet-ai/reference-sdk) currently runs `validate_profile_fields` only against `factbook.profile`, skipping per-factlet `profile:` overrides. This RFC requires the SDK to:

1. Iterate all factlets and dispatch per-factlet profile validation according to the per-factlet `profile` (if present) or the factbook-level profile.
2. Surface validation results per factlet with the originating profile attribution.
3. Add a `filter_by_mfg_scope(facts, scope_level, *, scope_id)` helper analogous to `filter_by_phase`.
4. Add a `filter_by_asset_phase(facts, phase)` helper.
5. Ship test fixtures covering all seven `mfg_scope` levels, all six `asset_phase` values, and a cross-domain Factbook mixing software-engineering + manufacturing factlets via per-factlet `profile:` overrides.

### 10. Interaction with FactSignal

FactSignal (§6 of v0.1 SPEC) is computed against a retrieved factlet set. If retrieval is scope-filtered, phase-filtered, or role-filtered, FactSignal is computed against the filtered set. Implementations SHOULD document whether their FactSignal score reflects all-scope or scope-filtered coverage.

### 11. Backward compatibility

The Manufacturing Profile is a new addition; there are no existing v0.1 manufacturing factbooks to preserve. All breaking-change concerns are within future profile-version bumps of `manufacturing`, not against v0.1 base spec.

## Alternatives considered

### A. Reuse base-spec `scope_level` field name with manufacturing enum

**Rejected because:** the base spec already defines `scope_level` with enum `team / project / feature / personal`. Reusing the field name with a different enum creates silent semantic conflict for readers that validate against both base + profile. A reader rejecting `scope_level: team` because it expects manufacturing values, or rejecting `scope_level: site` because it expects base values, both produce confusing error states. Distinct field name (`mfg_scope`) eliminates the contradiction.

### B. Reuse software-engineering `phase` field name with manufacturing enum

**Rejected because:** in BI/ETL contexts, downstream tools do column-level filters (`SELECT * FROM factlets WHERE phase = 'runtime'`), not row-context-aware joins. A multi-domain Factbook table with `phase` shared across software-engineering and manufacturing produces silently-wrong query results. Distinct field name (`asset_phase`) makes the disambiguation explicit at query time. The asymmetry between profiles is the cost; pipeline correctness is the benefit.

### C. Spatial scope in the base spec (universal)

Add `scope_level` (manufacturing version) and the identifier fields to the base factlet schema as universal.

**Rejected because:** the ISA-95 hierarchy is manufacturing-specific. Other domains have entirely different structural addressing. Forcing all domains into ISA-95 vocabulary is exactly the conflation the Profiles mechanism was designed to prevent.

### D. Free-form `tags` for scope (no schema change)

Producers could use tags like `scope:line-1` without a schema change.

**Rejected because:** the tag namespace is open and unenforceable. Two implementations would diverge on conventions, and consuming agents could not reliably filter without normalization. Same rejection as RFC 0005 Alt B.

### E. Single `location` string instead of split identifiers

Use one `location` string formatted as `enterprise/site/area/line/cell/unit/asset` rather than seven separate fields.

**Rejected because:** the seven identifiers are independently useful for filtering. Parsing a slash-delimited string at every retrieval is more brittle than a structured hierarchy.

### F. Drop `unit` to keep the hierarchy at six levels

Match the original RFC draft's six-level hierarchy `enterprise / site / area / line / cell / asset`.

**Rejected because:** batch-process plants (chemicals, pharma) cannot be modeled without distinguishing recipe-execution units (e.g. reactor R-1A) from physical assets installed within them (e.g. Pump-03). Without the `unit` level, retrieval queries like "factlets applicable to all assets within R-1A" cannot be expressed at the schema level.

### G. Defer safety vocabulary (severity, permission_role, hazard, valid_until) to RFC 0007

Match the original RFC draft's deferral plan.

**Rejected because:** without `permission_role` and `valid_until`, the profile is unusable in a 21 CFR Part 11 / EU Annex 11 context. Regulated MES deployments require (a) role-based visibility on operational knowledge (operators see operating ranges, only QA sees deviation history) and (b) explicit validity windows on validated parameters subject to periodic re-validation. The worked example (specialty chemicals + FDA-IND-validated reactor sequences) is the regulated domain that breaks without these fields. Pulling them forward into v0.1 is the minimum scope at which the worked example is internally consistent with the profile.

## Migration impact

- **For new manufacturing Factbook authors**: declare `profile: manufacturing` at root, add `mfg_scope` to every factlet (required), add `asset_phase` and identifier fields per the table in §3, optionally add safety vocabulary fields.
- **For implementations**: implementations that wish to surface manufacturing factbooks MUST validate the schema additions when the Factbook declares `profile: manufacturing`. Implementations unaware of the manufacturing profile remain conformant per RFC 0002 §15.3.4 (treat profile-specific fields as unknown keys, preserve on round-trip, emit a warning).
- **Reference SDK work required**: see §9. Per-factlet profile-override validation, two new filter helpers, test fixtures.
- **Breaking change?** No. Additive new profile.

## Deferred to follow-up RFCs

These additions are valuable but require their own design cycles:

- **RFC 0007 — Per-batch lifecycle** (`batch_phase` enum: `setup / charging / reaction / cooldown / discharge`) for batch-process manufacturing — distinct dimension from asset_phase.
- **RFC 0008 — Vendor-product field** for vendor-quirk factlets (e.g. `vendor: siemens-tia-portal-v18`).
- **RFC 0009 — ISA-95 Part 2 / B2MML / SAP plant master data interop appendix** — non-normative mapping table from `mfg_scope` levels to ISA-95 Part 2 EquipmentHierarchy and SAP S/4HANA plant master data.
- **RFC 0010 — CMMS URI scheme convention** — standardized URI form for cross-system references (e.g. `cmms://maximo/wo/12345`, `urn:cmms:sap-pm:notification:0010234567`) so that source IDs (RCA-2024-08, MOC-2024-019, INC-2024-027) become machine-resolvable.
- **RFC 0011 — `superseded_by_condition`** for non-date-based time-bound factlets ("valid until next firmware release").

## Open questions

- Should `mfg_scope: enterprise` factlets always carry `enterprise_id` (current draft says yes) or be allowed without it on the assumption the Factbook declares the enterprise at root? Current proposal: required, for cross-enterprise BI safety.
- For asset-on-line topology where `line_id` is set, is `unit_id` permitted (informational lineage) or prohibited? Current proposal: permitted (informational).
- Should `permission_role` enforcement be normative or implementation-defined? Current proposal: implementation-defined (the protocol mandates the field is preserved; consumer policy decides what to do with it).

## Acceptance criteria

- Two maintainer approvals OR working group consensus.
- RFC 0002 (Profiles mechanism) accepted as a prerequisite (✅ ratified 2026-05-08).
- Manufacturing-domain-expert review (≥1 reviewer with controls-engineering experience AND ≥1 reviewer with regulated-MES/IT-OT-integration experience) sign-off on the hierarchy choice, phase enum, and safety vocabulary.
- Create `factlet-ai/spec/profiles/manufacturing/` directory containing: `SPEC.md`, `factlet.schema.json`, `examples/specialty-chemicals-plant.yaml`.
- Add `manufacturing` to `factlet-ai/spec/profiles/REGISTRY.md`.
- Reference SDK (factlet-ai/reference-sdk) PR addressing §9 SDK conformance work.
- At least one example factbook in `factlet-ai/registry/examples/manufacturing-plant/` published under the manufacturing profile (worked example from §7).

