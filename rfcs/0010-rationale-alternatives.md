# RFC 0010: Rationale + Alternatives

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-06-01
- **Last updated**: 2026-06-01
- **Discussion**: TBD
- **Target spec version**: v0.2
- **Scope**: project:factlet-ai

## §1 Summary

This RFC adds two OPTIONAL fields to the **base** factlet schema: `rationale`, a free-text string recording *why a claim holds or why a choice was made*, and `alternatives`, an array of structured objects recording *options that were considered and rejected at decision time*. Both fields are additive: a v0.1 reader treats them as absent, and a v0.2 reader that encounters a factlet without them behaves exactly as before. The fields belong to the base schema rather than to a profile because intent — the reason for a choice and the road not taken — is domain-general: a torque specification, a refund policy, a clinical-pathway selection, and a code convention each have a rationale and a rejected alternative. Unlike profile-scoped fields whose value spaces are domain-specific (the `phase` enum of RFC 0005, the path-glob selectors of RFC 0008), neither `rationale` nor `alternatives` has a constrained value space, so nothing couples them to a particular profile.

## §2 Motivation

A factbook records claims about a system. A claim states *what* is true — a threshold, a policy, a convention, an interface contract. It does not, by itself, record *why that value and not another*. The moment a decision is implemented, the rejected option and the reason for the choice are no longer present in the artifact that encodes the decision: source code, a configuration file, or a policy document records the chosen value, but not the design space it was selected from. That context survives, if at all, only in a commit message, a chat thread, or a person's memory — none of which is part of the factbook and none of which is durable.

`rationale` and `alternatives` are where the factbook holds that context. Recording the chosen value alongside the reason for it, and alongside the options that were rejected, makes a factlet legible to a later reader — human or automated — who must understand or reconstruct the decision without access to the original discussion.

Without these fields in the open protocol, a conforming implementation, or an auditor's general-purpose tooling, cannot depend on intent being present in a portable form. Any claim that a factbook preserves the reasoning behind a decision — and therefore supports faithful reconstruction of behavior from the factbook alone — would rest on implementation-private storage rather than on the spec. Promoting these fields to the base schema converts implementation-private intent into a portable guarantee that any conforming reader can rely on.

## §3 Specification — base schema fields

A conforming implementation **MAY** persist and **MUST** round-trip the following two OPTIONAL fields on a factlet. Both are absent-by-default; a factlet that omits them is valid.

### 3.1 Schema fragment

The fragment below extends the base `schema/factlet.schema.json` factlet-object definition.

```json
"rationale": {
  "type": "string",
  "description": "Why the claim holds or why this choice was made. The intent that source artifacts structurally discard."
},
"alternatives": {
  "type": "array",
  "description": "Options considered and rejected at decision time. Distinct from supersession (base-spec `supersedes`): a rejected alternative was never adopted and has no factlet of its own.",
  "items": {
    "type": "object",
    "properties": {
      "summary":          { "type": "string", "description": "The rejected option, stated plainly." },
      "rejected_because": { "type": "string", "description": "Why it was not chosen." },
      "ref":              { "type": "string", "description": "Optional factlet id, if the rejected alternative was itself once a recorded factlet (links to the base-spec `supersedes` relation)." }
    },
    "required": ["summary"],
    "additionalProperties": false
  }
}
```

### 3.2 Field semantics

| Field                          | Type            | Required | Description |
|--------------------------------|-----------------|----------|-------------|
| `rationale`                    | string          | OPTIONAL | Why the claim holds or why the choice was made. Free text. No constrained value space. |
| `alternatives`                 | array of object | OPTIONAL | Options considered and rejected at decision time. Empty array and absent field are equivalent. |
| `alternatives[].summary`       | string          | REQUIRED (within an alternative) | The rejected option, stated plainly. The minimal useful record of a rejected alternative. |
| `alternatives[].rejected_because` | string       | OPTIONAL | Why the option was not chosen. SHOULD be populated when known; this is where reconstruction value concentrates. |
| `alternatives[].ref`           | string          | OPTIONAL | A factlet id, present only when the rejected alternative was itself once a recorded factlet. Bridges to the base-spec `supersedes` relation (see §4). |

`summary` is the only required sub-field of an `alternatives` entry: the minimal useful record is "this option was considered." `rejected_because` carries the reasoning a later reader needs and **SHOULD** be populated when the reason is known. `ref` is populated only in the specific case where the rejected alternative was previously a factlet in its own right — see §4 for why this is the only point of contact between `alternatives` and the supersession relation.

### 3.3 Versioning

Both fields are additive and OPTIONAL. A v0.1 reader **MUST** ignore unknown fields and therefore reads a factlet carrying `rationale` or `alternatives` exactly as it reads the same factlet without them. A v0.2 reader **MUST NOT** require either field to be present. No factlet is invalid for omitting them.

## §4 Two distinct WHYs

A factbook expresses two different kinds of "why," and they must not be merged. This section is normative on the distinction because the two are easy to conflate in review.

| | What it records | Where it lives |
|---|---|---|
| **Causal why** | which factlet this one *replaced* — lineage between recorded facts, ordered in time | base-spec `supersedes` (write-time) · RFC 0007 `superseded_by` (read-time) |
| **Intent why** | *why this option was chosen*, and the alternative that was *rejected* | this RFC (`rationale`, `alternatives`) |

These are different relations:

- A **superseded factlet** (base-spec `supersedes`, surfaced at read time via RFC 0007 `superseded_by`) was *once recorded as true* and later replaced. It has its own factlet id and its own lineage. The supersession relation is between two recorded facts, ordered in time.
- A **rejected alternative** (`alternatives`) was *never adopted*. It has no factlet of its own. It exists only as the contrast that makes the chosen claim legible.

Both can be described as "the road not taken," but at different layers. Supersession is lineage between recorded facts over time; `alternatives` is the decision context captured at the moment of choice. Folding `alternatives` into supersession would discard every rejected option that never became a factlet — which is the majority of them.

The single point of contact is `alternatives[].ref`. When — and only when — a rejected alternative *was itself* a prior factlet that the current factlet superseded, an implementation **MAY** set `ref` to that prior factlet's id. In that case the rejected alternative is both an intent-why record (it was a considered-and-rejected option) and a causal-why record (it was a superseded factlet); `ref` links the two without conflating the relations. When the rejected alternative was never a factlet, `ref` is absent.

## §5 Provenance projection (informative)

This section is informative. It describes how the two fields project into a provenance model and does not impose a normative requirement.

PROV-O models derivation and influence through relations such as `wasDerivedFrom` and `wasInformedBy`. These express *causal* why — which input a fact came from — and align with the supersession relation described in §4 (and, at the factbook level, RFC 0004 `dependencies`). PROV-O has no native vocabulary for *intent* why: there is no standard relation that expresses "this option was chosen over that one because of this reason."

A natural projection that stays within PROV-O's extension model: represent each rejected alternative as a `prov:Entity` that was `prov:wasInvalidatedBy` the decision Activity that generated the factlet, and carry the `rationale` as a domain-extension annotation on that Activity. This keeps the factlet's provenance core auditable with general-purpose PROV tooling, while the intent layer rides as a typed extension — the intended use of PROV-O specialization. The base schema defines the *fields*; a provenance-serialization profile would define the *projection*. The projection is named here only so it need not be re-derived; its normative specification is deferred to a future provenance-serialization RFC.

## §6 Worked example

The example uses a refund-routing domain. It is illustrative and protocol-neutral.

```yaml
content:
  - id: f014
    statement: "Refund routing uses a 90-day age cutoff for manual review."
    confidence: 1.0
    sources: ["docs/refund-policy.md:8"]
    rationale: >
      90 days aligns the manual-review window with the card-network chargeback
      window, so operations never auto-approves a refund that could still be disputed.
    alternatives:
      - summary: "Flat dollar threshold (e.g. all refunds over $500 to manual)."
        rejected_because: >
          Decouples review from chargeback risk; high-value, low-age refunds are
          low-risk and would clog the manual-review queue.
      - summary: "60-day cutoff."
        rejected_because: >
          Leaves a 30-day gap in which a still-disputable refund is auto-approved.
        ref: f009          # f009 was the superseded 60-day factlet (supersession lineage)
```

A reader reconstructing this routing rule from the factbook alone has the cutoff (`statement`), the reason it is 90 days and not some other value (`rationale`), and the two designs it must not reintroduce (`alternatives`). None of these survive in the routing code itself. The second alternative also carries `ref: f009`, marking the case where the rejected 60-day option was previously a recorded factlet that this factlet superseded (§4).

## §7 Design alternatives considered

This section records the design choices behind this RFC itself.

- **Scope the fields to the software-engineering profile instead of the base schema.** Rejected: every domain has intent. Scoping these fields to software would force every other profile to redefine the same two fields and would wrongly signal that rationale is a software-specific concern. The profile mechanism (RFC 0002) exists to isolate domain-specific value spaces — the `phase` enum of RFC 0005, the path-glob selectors of RFC 0008 — not domain-general relations that have no constrained value space.
- **Keep `alternatives` as a flat array of strings.** Rejected: a bare list of strings cannot carry `rejected_because`, which is where the reconstruction and audit value lives. The structured object is only marginally heavier and round-trips losslessly in YAML.
- **Fold `alternatives` into supersession** (base-spec `supersedes`). Rejected: see §4. Most rejected options never become factlets, so the supersession relation cannot hold them.
- **Leave intent as implementation-private storage (do nothing).** Rejected: that is the status quo this RFC addresses. With no field in the open protocol, intent is not portable across conforming implementations, and any reconstruction guarantee that depends on it cannot be relied upon by a conforming reader.

## §8 Open questions

### 8.1 Should `rationale` be required above an authority threshold?

A future RFC may consider requiring `rationale` for factlets above a confidence or authority threshold — for example, requiring that high-authority compliance factlets carry an explicit reason. The current draft recommends **SHOULD**, not **MUST**, for v0.2, and defers a hard requirement pending evidence that audit demand warrants it.

### 8.2 Distinguishing who *decided* from who *recorded*

The fields in this RFC record *why* a decision was made but not *who* made it, which is distinct from who recorded the resulting factlet (RFC 0003 origination). A `decided_at` / `decided_by` pair — the decision authority, separate from the recording actor — is adjacent to attribution and is deferred to RFC 0011 (attribution), not specified here.

## Cross-references

This RFC depends on or relates to:

- **RFC 0002 — Profiles Mechanism**: provides the profile-extension model. §1 and §7 cite the profile mechanism's purpose — isolating domain-specific value spaces — as the reason `rationale` and `alternatives` belong to the base schema rather than to a profile.
- **RFC 0003 — Origination provenance block**: records who recorded a factlet and where it came from. §8.2 distinguishes the recording actor (RFC 0003) from the decision authority.
- **Base spec (`supersedes`) + RFC 0007 (`superseded_by`)**: define the supersession relation — the causal-why lineage between recorded facts. §4 distinguishes intent-why (`alternatives`) from this supersession lineage; `alternatives[].ref` (§3.2) is the single bridge to it. (RFC 0004 — Composable factbooks via dependencies — governs factbook-level composition through `dependencies.factbooks`, a different relation; there is no factlet-level `depends_on` field in the spec.)
- **RFC 0005 — Software-profile `phase` enum** and **RFC 0008 — Applicability selector (`applies_to`)**: profile-scoped fields whose value spaces are software-specific. §1 and §7 cite them as the contrast that justifies keeping `rationale` and `alternatives` in the base schema.
- **RFC 0011 — Attribution**: §8.2 defers the decision-authority question (who made the call, distinct from who recorded the factlet) to this sibling RFC.

## Acceptance criteria

- RFC 0010 opened against `factlet-ai/spec`, adding `rationale` and `alternatives` to the **base** schema, with the §4 "two distinct WHYs" distinction documented in `SPEC.md`.
- Addition of the `rationale` and `alternatives` field definitions to `schema/factlet.schema.json` (or equivalent base JSON-schema fragment) capturing the shapes from §3.1.
- Reference SDK (`factlet-ai/reference-sdk`): schema validation for both fields, plus YAML round-trip fixtures covering (a) `rationale` only, (b) `alternatives` with a `ref`, and (c) both fields absent.
- At least one example factbook demonstrating a non-trivial `alternatives` set, including one entry that carries a `ref` to a superseded factlet (the §4 bridge case) and one that does not.
