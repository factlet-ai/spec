# RFC 0013: Schema composition and validation levels

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-06-02
- **Last updated**: 2026-06-02
- **Discussion**: TBD
- **Target spec version**: v0.2
- **Scope**: project:factlet-ai

## Summary

This RFC specifies how a base factlet schema and a profile factlet schema combine for validation, and resolves the open-vs-closed-world question this raises by distinguishing **three validation categories** for the keys of a factlet:

1. **Document level** (the factlet record as a whole) — **open-world**: an unknown top-level key is preserved, never rejected.
2. **Closed owned-objects** (a nested object the spec or a profile fully defines and authors `additionalProperties: false`, e.g. `attribution`, a `verify` oracle object) — **closed-world**: an unrecognized key inside it is an error.
3. **Open-by-design objects** (a nested object whose schema deliberately leaves `additionalProperties` open so it can carry added keys — `extension` for vendor keys, `origination` for profile-added provenance keys) — **open-world**: unknown keys are preserved, not rejected.

It makes two non-breaking base-schema changes. **(1)** The base schema sets top-level `additionalProperties: true` (the profile schemas already are `true`/unset). The current base top-level `additionalProperties: false` contradicts the spec's preserve-and-do-not-reject requirements (SPEC.md §15.3, RFC 0002 §4) and rejects profile fields (`phase`, `applies_to`, `verify`) those clauses require a reader to preserve. **(2)** The base `origination.source_type` is relaxed from an enforced `enum` to `type: string` with the baseline values recorded as *recommended*, because RFC 0003 makes `source_type` profile-extensible and its own description says readers SHOULD warn on (not reject) an unknown value — yet the literal `enum` rejects them. No nested object's `additionalProperties` changes; both changes only widen acceptance, so they are non-breaking.

## Motivation

The base factlet schema (`schema/factlet.schema.json`) declares top-level `additionalProperties: false`. Three published rules require the opposite at the document level:

- **SPEC.md §9.3** (Conformance) — a reader MUST ignore unknown keys in the `extension` object without error.
- **SPEC.md §15.3** (Profiles conformance, clause 4) — a reader that does not know a Factbook's declared profile MUST treat profile-specific fields as unknown keys and preserve them on round-trip; it MUST NOT reject the Factbook on the basis of fields it does not recognize. **RFC 0002 §4** (Conformance) states the same rule for the Profiles mechanism.
- **RFC 0005 / RFC 0008 / RFC 0009** — `phase`, `applies_to`, and `verify` live in the software-engineering profile, not the base. Under a strict top-level base, a factlet carrying any of them fails base validation.

The contradiction is observable. A factlet that declares `profile: software-engineering` and carries `phase` is rejected when validated against the base schema in isolation, even though SPEC.md §15.3 says it must be preserved. A manufacturing factlet carrying a profile field hits the same wall. A future-version field does too. The base schema, applied closed-world at the top level, breaks the forward-compatibility and profile-extensibility the rest of the spec mandates.

The fix is not to weaken validation everywhere. Strictness is still wanted **inside** objects the spec fully owns and authors closed: a `verify.assertion` block with a misspelled `expr` key, or an `attribution.produced_by` with a typo'd `agent`, is a real authoring error and should fail loud. But other nested objects are open **by design** — `extension` (vendor keys a reader ignores) and `origination` (which RFC 0003 leaves open so a profile may add provenance keys). The resolution is to name three categories rather than force one `additionalProperties` flag onto every level.

## Detailed design

### Three categories

A factlet is a mapping. Each key belongs to one of three categories, and validation differs per category:

| Category | Where | Rule |
|---|---|---|
| **Document level** | top-level factlet keys | **open-world** — preserve unknown, never reject |
| **Closed owned-object** | a nested object whose schema declares `additionalProperties: false` | **closed-world** — unknown key is an error |
| **Open-by-design object** | a nested object whose schema omits/sets `additionalProperties: true` to allow extension | **open-world** — preserve unknown, never reject |

The category of a nested object is read **from its own schema**, not assumed. An object is closed iff its schema declares `additionalProperties: false`; otherwise it is open.

### Document level — open-world (normative)

The base schema and every profile schema **MUST** set top-level `additionalProperties: true` (or leave it unset, which JSON Schema treats as `true`). A validator **MUST NOT** reject a factlet because it carries a top-level key the validator does not recognize; it **MUST** preserve that key on round-trip, and **SHOULD** warn if the key belongs to a declared-but-uninstalled profile (per SPEC.md §15.3 / RFC 0002 §4). This generalizes the existing preserve rules — `extension` keys (§9.3) and profile fields (§15.3) — to any unknown top-level key, which is what forward-compatibility across spec versions requires.

### Closed owned-objects — closed-world (normative)

A nested object whose schema declares `additionalProperties: false` is **closed**: an unrecognized key inside it **MUST** fail validation. This is where the spec's fail-loud guarantee lives. The closed owned-objects defined as of v0.2 are:

- **Base spec:** `attribution`, `attribution.produced_by`, `attribution.authority`, and each `alternatives[]` item (RFC 0010, RFC 0011).
- **Software-engineering profile:** `applies_to`, `verify`, `verify.assertion`, `verify.test_ref`, and each `verify.examples[]` item (RFC 0008, RFC 0009).

A profile **MUST NOT** loosen a base closed owned-object to open, nor redefine its existing keys; it may only add new closed owned-objects of its own.

### Open-by-design objects — open-world (normative)

A nested object whose schema omits `additionalProperties` (or sets it `true`) is **open by design** and **MUST NOT** be tightened to closed by this or any profile, because its openness is load-bearing:

- **`extension`** (§3) — the sanctioned home for vendor-specific keys a reader **ignores**. (`extension` is distinct from profile fields, which a profile-aware reader **validates**; RFC 0002 §4 rejected reusing `extension` for profile vocabulary for exactly this reason.)
- **`origination`** (RFC 0003) — open for unknown **keys**: a profile or vendor MAY add domain-specific provenance keys (e.g. a manufacturing `plc_tag_id`), which a reader preserves rather than rejects, so `origination` is not closed.

  **Second axis — `source_type` enum values (resolved here).** `additionalProperties` governs unknown *keys*, not the *values* of the known `source_type` key, so the open-world change above does nothing for them. The base `source_type` is today a fixed `enum` of five values, yet RFC 0003 and the field's own description ("readers SHOULD warn on unknown values, not reject") make it profile-extensible — so the literal `enum` rejects values the prose promises to preserve. **This RFC relaxes the base `source_type` from an enforced `enum` to `type: string`, recording the five values (`manual`, `llm`, `import`, `forward-pass`, `reverse-pass`) as *recommended* in the field description.** Any string then validates; a reader **SHOULD** warn on a value outside the recommended set, not reject it. A profile documents its own `source_type` values in its SPEC.md. This aligns the schema with RFC 0003's stated extensibility.

A reader **SHOULD** warn on an unrecognized key inside an open-by-design object, and **MUST** preserve it on round-trip.

### Profile composition — the effective schema

For a Factbook declaring `profile: <name>`, the **effective** factlet schema a profile-aware validator applies composes base and profile at two levels:

```
Top level:    effective.properties = base.properties ∪ profiles/<name>.properties
              effective.additionalProperties = true        (document level — open-world)

Owned object: when a profile adds a KEY to a base open-by-design object (e.g. a
              provenance key on origination), the effective object admits it; a base
              CLOSED owned-object is not extended, only added-alongside by new
              profile-owned closed objects. (A profile-defined source_type value
              validates because the base source_type carries no enforced enum — see
              §Open-by-design, second axis — not via additionalProperties.)
```

A profile-aware validator validates each known field — base or profile — against its defined shape (including the closed `additionalProperties: false` inside each closed owned-object), admits profile-added keys to open-by-design objects, and preserves any remaining unknown top-level key. A profile-unaware validator validates the base fields it knows, treats profile fields as unknown (preserve, warn), and never rejects on them.

**Implementer note (composition mechanics).** Compose by **property-level union**, not by `allOf: [base, profile]`. A JSON Schema `allOf` whose branches carry `additionalProperties: false` rejects each branch's *sibling* properties, because `additionalProperties` in one subschema does not see the properties declared in a sibling subschema — so `allOf`-composing a closed base with a profile silently rejects the profile's fields. Union the `properties` maps and apply a single document-level `additionalProperties: true`; keep `additionalProperties: false` only inside each individual closed owned-object (where the SE profile already uses an internal `allOf` correctly, scoped within one object).

### Normative schema change

- `schema/factlet.schema.json`: **(1)** top-level `additionalProperties` changes `false` → `true`; **(2)** `origination.source_type` drops its enforced `enum`, keeping `type: string` with the five baseline values (`manual`, `llm`, `import`, `forward-pass`, `reverse-pass`) listed as recommended in the field description. No nested object's `additionalProperties` changes.
- `profiles/<name>/factlet.schema.json`: top-level `additionalProperties` is `true` (or unset) — already the case for the software-engineering and manufacturing profiles; this RFC makes it normative.
- Every closed owned-object listed above retains `additionalProperties: false`; every open-by-design object (`extension`, `origination`) retains its open posture.

## Backward compatibility

Both changes are **non-breaking** — each only widens the accepted set. Relaxing the base top-level `additionalProperties` `false`→`true` keeps every previously-valid factlet valid and now accepts factlets the old schema wrongly rejected (those carrying profile or forward-compatible fields). Relaxing `origination.source_type` from `enum` to `type: string` keeps all five baseline values valid and now accepts profile-defined values the strict enum wrongly rejected, matching RFC 0003. **No nested object's `additionalProperties` changes** — no `origination` or `extension` tightening — and no previously-caught authoring error inside a closed owned-object is newly admitted.

## Alternatives considered

### A. Keep the base closed-world at the top level and define a union rule

A reader applies `additionalProperties: false` over the union of base and active-profile properties. **Rejected:** this still rejects fields from a profile the reader does not know, contradicting SPEC.md §15.3 / RFC 0002 §4, and it cannot preserve a forward-compatible field. It also requires a "union" step whose only purpose is to undo the base's own top-level strictness — strictness the spec does not want at the document level.

### B. Make the entire schema open-world (nested objects too)

Set `additionalProperties: true` everywhere, including closed owned-objects. **Rejected:** this discards the fail-loud guarantee inside owned objects. A `verify.assertion` with a misspelled key, or an `attribution` with a typo'd sub-field, would validate silently, defeating the machine-checkability and accountability those objects exist to provide.

### C. Add profile field names to the base schema

List `phase`, `applies_to`, `verify` directly on the base. **Rejected:** this re-pollutes the domain-neutral base with software-profile vocabulary — the conflation the Profiles mechanism (RFC 0002) exists to prevent — and does not generalize (every future profile would have to amend the base).

## Acceptance criteria

- `schema/factlet.schema.json` top-level `additionalProperties` is `true`.
- `schema/factlet.schema.json` `origination.source_type` has no enforced `enum`; it is `type: string` with the five baseline values documented as recommended.
- The closed owned-objects enumerated above (base: `attribution`, `attribution.produced_by`, `attribution.authority`, `alternatives[]`; SE profile: `applies_to`, `verify`, `verify.assertion`, `verify.test_ref`, `verify.examples[]`) each retain `additionalProperties: false`.
- The open-by-design objects (`extension`, `origination`) retain their open posture; neither is tightened.
- `profiles/software-engineering/factlet.schema.json` **and** `profiles/manufacturing/factlet.schema.json` top-level `additionalProperties` is `true` (or unset); the rule is profile-general, not SE-specific.
- A factlet declaring `profile: software-engineering` and carrying `phase`/`applies_to`/`verify`, and a factlet declaring `profile: manufacturing` and carrying both an unknown `origination` key (e.g. `origination.plc_tag_id`) and a profile-defined `source_type` (e.g. `opc-ua-server`), all validate against the base schema and are preserved unchanged on round-trip by a profile-unaware reader.
- A factlet with a misspelled key inside a closed owned-object (e.g. `verify.assertion`) fails validation.
- `SPEC.md` documents the three categories and the effective-schema composition rule (a new subsection under §15).

## Cross-references

- **SPEC.md §9.3** — ignore unknown keys in the `extension` object (the open-by-design basis for `extension`).
- **SPEC.md §15.3 (Profiles conformance, clause 4) / RFC 0002 §4 (Conformance)** — preserve unknown profile fields and do not reject; the document-level open-world basis and the base-vs-profile separation this RFC's composition rule formalizes.
- **RFC 0003 — Origination** — `origination` is open by design (a profile may add provenance keys); the object this RFC must not close. This RFC relaxes its `source_type` from an enforced `enum` to advisory, aligning the schema with RFC 0003's stated per-profile extensibility (§Normative schema change).
- **RFC 0005 / RFC 0008 / RFC 0009** — `phase` / `applies_to` / `verify`: the profile fields a strict top-level base wrongly rejected.
- **RFC 0010 / RFC 0011** — `alternatives[]` / `attribution`: closed owned-objects that retain `additionalProperties: false`.

> **Errata notes.**
> 1. Several published documents (SPEC.md §15.3, RFC 0002 §4, RFC 0003) cite "§9.3" as the round-trip-preserve-unknown rule, but published §9.3 is scoped to the `extension` object. The general preserve-unknown-profile-fields requirement is SPEC.md §15.3 clause 4. This RFC cites the correct sections; the upstream `§9.3` cross-references are a separate documentation-errata item.
> 2. RFC 0003's prose table lists `source_type` as an `enum` of five values, but its §2.1 and the field's own description make it profile-extensible ("readers SHOULD warn on unknown values, not reject"). This RFC resolves the resulting schema-vs-prose mismatch by relaxing the base `source_type` to `type: string` with the five values recommended (§Normative schema change). RFC 0003's table SHOULD be read as the recommended baseline, and MAY be annotated "(recommended; extensible per profile)" as a minor documentation follow-up.
