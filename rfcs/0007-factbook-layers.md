# RFC 0007: Factbook layers — 10-layer enum, precedence rules, supersession-at-read-time

- **Status**: Draft
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-05-20
- **Last updated**: 2026-05-20
- **Discussion**: TBD (link to GitHub Discussion thread when opened)
- **Target spec version**: v0.2
- **Extends**: RFC 0002 (Profiles mechanism)
- **Scope**: project:factlet-ai

## Summary

This RFC extends RFC 0002 with a layered organizational model for Factbooks. A conforming Factbook MUST classify each knowledge pack (kp) with a `layer_type` drawn from a fixed 10-value enum (`Individual`, `Codebase`, `Project`, `Program`, `Product`, `Service`, `Team`, `Division`, `Company`, `Domain-Pack`); the parent_kp_id graph MUST satisfy a parent-layer compatibility matrix (§3); conflicting factlets are resolved by `precedence_rank` (§4); and retrieval MUST filter `WHERE superseded_by IS NULL` by default (§5). RFC 0002 specified the Profiles mechanism along the *domain* axis (software, manufacturing, healthcare); this RFC specifies a parallel mechanism along the *organizational* axis. The two are orthogonal and compose.

## Motivation

RFC 0002 introduced Profiles as the canonical extension axis for domain vocabulary. A Profile classifies *what kind of knowledge* a factlet encodes (software-engineering, manufacturing, legal). Empirical use across multi-Factbook deployments has surfaced a second, orthogonal classification need: *whose knowledge it is, and at what organizational scope*.

A flat Factbook design — every factlet a peer of every other — has three observed failure modes when applied to organizations larger than one person:

1. **No way to express that a Codebase factlet inherits from its Project, which inherits from its Product, which inherits from its Company.** Without inheritance, the same factlet ("we redact PII before logging") must be duplicated into every Codebase Factbook. Duplication is the root cause of contradiction at retrieval time.

2. **No way to express override authority.** When a Codebase factlet ("this service logs raw payloads for debug builds") conflicts with a Company factlet ("PII MUST be redacted before logging"), retrieval has no signal that one is authoritative and the other is local. A naive merge returns both with equal weight; a regulator audit cannot distinguish policy from local practice.

3. **No way to express that some factlets travel with the user (preferences, working style) while others travel with the employer (compliance, policy).** A user moving between employers loses their Individual factlets if those are stored in Company-rooted trees; an employer loses Company factlets if those are stored in Individual-rooted trees. The portability boundary needs to be representable.

The Profiles mechanism (RFC 0002) does not address any of these — it operates within a single Factbook, not across an organizational graph. This RFC introduces the layer-type axis to address them.

### Concrete observed need

Across observed Factbook deployments (N organizations sampled 2026-Q2, ranging from one-person teams to multi-division companies), the same three roots recur:

- A **per-user** root holding portable preferences, working style, terminal habits, communication conventions — the factlets that should follow the user across employers.
- A **per-employer** root holding organizational policy, compliance posture, internal vocabulary — the factlets that belong to the company and stay with the company.
- A **per-vertical** root holding domain knowledge (manufacturing standards, healthcare regulation citations) — the factlets that are neither user-specific nor employer-specific, but apply to anyone operating in that vertical.

These three are not nested; they are orthogonal. A factlet authored at the Codebase layer might inherit from a Company root (policy), be conditioned by a Domain-Pack root (vertical regulation), and be filtered against the operator's Individual root (preferences). Without all three roots represented as first-class, the model collapses into a single hierarchy that cannot express the observed relationships.

The 10-layer enum (§3) decomposes the observed organizational topology into the smallest set of nameable layers that still admits a parent-layer compatibility matrix without ambiguity. Layers below 10 (e.g. omitting `Program` or `Service`) cause observed organizations to be unrepresentable without overloading a single layer with two meanings. Layers above 10 add nameable concepts whose distinctions are not load-bearing in retrieval (Division-of-a-Division collapses to "Division with parent_kp_id pointing at another Division").

## Detailed design

### §3. Specification — Parent-layer compatibility matrix

A conforming Factbook MUST classify each kp with a `layer_type` field whose value is one of:

`Individual` · `Codebase` · `Project` · `Program` · `Product` · `Service` · `Team` · `Division` · `Company` · `Domain-Pack`

The `parent_kp_id` of a kp MUST point at a kp whose `layer_type` is in the allowed-parents set for the child's layer. The full matrix:

| Child layer  | Allowed parents                                              |
|--------------|--------------------------------------------------------------|
| Individual   | (none — root)                                                |
| Codebase     | Project · Team                                               |
| Project      | Program · Product · Service · Division · Company             |
| Program      | Division · Company · Product                                 |
| Product      | Division · Company                                           |
| Service      | Division · Company · Product · Program                       |
| Team         | Division · Company · Product · Service                       |
| Division     | Company                                                      |
| Company      | (none — root)                                                |
| Domain-Pack  | (none — root)                                                |

Three layers have no parent and are roots: `Individual`, `Company`, `Domain-Pack`. All other layers MUST declare a `parent_kp_id` pointing at a kp of an allowed parent type. A conforming implementation MUST reject any `INSERT` or `UPDATE` that violates the matrix.

#### 3.1. Cycle prohibition

A conforming implementation MUST reject any operation that would introduce a cycle in the `parent_kp_id` graph. The graph rooted at any kp via `parent_kp_id` traversal MUST be a directed acyclic graph (in fact, a tree: each non-root kp has exactly one parent).

A reference implementation may enforce cycle prohibition via either (a) a recursive CTE check at the write path that creates/updates parent links, or (b) an offline integrity check run on schema migration. Method (a) is RECOMMENDED because it catches violations at write time; method (b) is allowed for backfill scenarios where pre-existing data may need a transitional period to be brought into compliance.

#### 3.2. Single-parent constraint

A kp MUST have at most one `parent_kp_id`. Multi-parent layering (a Codebase that belongs to two Projects simultaneously) is out of scope for this RFC. Cross-cutting classification SHOULD use the `domain_tags` field (§6.2) rather than multi-parent links.

#### 3.3. Layer-type immutability

Once a kp is created with a given `layer_type`, the `layer_type` value SHOULD NOT change. Changing the layer type of an existing kp may invalidate the parent-layer compatibility of its children and may invalidate precedence-rank assumptions cached by consumers. If a layer-type change is operationally required, the conforming implementation MUST re-validate the entire subtree rooted at the changed kp against the matrix.

### §4. Specification — Precedence ranks

When two factlets at different layers conflict on the same `conflict_group_id`, resolution follows the **ancestry chain** defined by the §3 parent matrix — not a global rank comparison. A factlet is overridden only by a conflicting factlet whose layer is in its `can_be_overridden_by` set: the **transitive ancestor layers** of its own layer per §3. When neither of two conflicting layers is an ancestor of the other, they are **lateral** — both factlets are retained and reported with `lateral_conflict=true` (§4.1), and the caller resolves. The canonical assignment:

| Layer        | precedence_rank | can_be_overridden_by (transitive ancestors per §3)                |
|--------------|-----------------|-------------------------------------------------------------------|
| Company      | 1               | (none — root authority)                                           |
| Division     | 2               | Company                                                           |
| Product      | 3               | Company · Division                                                |
| Program      | 4               | Company · Division · Product                                      |
| Service      | 5               | Company · Division · Product · Program                            |
| Project      | 6               | Company · Division · Product · Program · Service                  |
| Team         | 7               | Company · Division · Product · Program · Service                  |
| Codebase     | 8               | Company · Division · Product · Program · Service · Project · Team  |
| Individual   | 9               | (none — sovereign)                                                |
| Domain-Pack  | (N/A)           | (orthogonal — see §6)                                             |

`precedence_rank` is a total order **consistent with** ancestry: every ancestor layer has a strictly lower rank (lower rank = closer to an authority root). It is used as the §5.4 supersession ceiling and the §4.1 tie signal — **not** as a standalone "lowest rank always wins" override key. (Each row's `can_be_overridden_by` is exactly that layer's transitive ancestor set under the §3 matrix; `Project` and `Team` are §3 peers — neither is an ancestor of the other — so they share the same ancestor set and conflict between them is always lateral, never a silent override.) `Individual` has `precedence_rank=9` and an empty `can_be_overridden_by` list because the Individual root is the user's portable layer and is not overridable by any employer layer; in retrieval terms, an Individual factlet that conflicts with a Company factlet is reported as `lateral_conflict=true` with the Individual factlet retained, not silently shadowed.

#### 4.1. Lateral conflicts

When two factlets sharing a `conflict_group_id` are at layers where **neither is the other's transitive ancestor** per §3 — *lateral peers* — the retrieval API MUST return both, MUST set `lateral_conflict=true` on each, and the caller MUST resolve. Among the employer layers the **only** lateral pair is `Project` (rank 6) and `Team` (rank 7): per §3 neither is in the other's allowed-parents closure, so neither overrides the other. (Two factlets at the *same* layer are the common intra-layer case; they resolve by §5.4 supersession, not as a lateral conflict.) A conforming implementation MUST NOT auto-resolve a lateral conflict by recency, by author, by numeric `precedence_rank` comparison, or by any other implicit heuristic; the resolution is a caller responsibility.

The `Project`/`Team` boundary is the most common lateral site in practice, because matrixed organizations routinely place a Codebase under both a standing Team and a cross-team Project. A conforming UI SHOULD surface lateral conflicts for human arbitration rather than fail open.

#### 4.2. Enterprise non-overridability

Factlets carrying `compliance_tier='non_overridable'` MUST be returned by retrieval regardless of relevance ranking, and MUST NOT be marked as superseded by any factlet at a lower precedence rank (i.e. a higher numeric rank). A Codebase factlet (rank 8) MUST NOT supersede a Company factlet (rank 1) carrying `compliance_tier='non_overridable'`, even if the conflict_group_id matches.

The `compliance_tier` enum values defined by this RFC are:

- `non_overridable` — MUST be returned; MUST NOT be superseded by lower-authority layers.
- `standard` — standard supersession rules apply per §4. This is the default tier when `compliance_tier` is omitted.
- `advisory` — informational only; MAY be filtered out of retrieval when stricter tiers exist in the same `conflict_group_id`.

Implementations MAY add additional tiers in their reference SDK; the three above are the conformance baseline.

#### 4.3. Why integer ranks (not enum-keyed table)

The choice of INTEGER for `precedence_rank` (rather than a separate enum-keyed precedence table) is a tradeoff. INTEGER simplifies SQL comparison (`ORDER BY precedence_rank ASC`), enables index range scans, and keeps the canonical assignment in the spec rather than in a runtime table that can drift. An enum-keyed lookup table would be more extensible (a new layer could be added without renumbering), but extensibility is the explicit non-goal: the 10-layer enum is fixed by this RFC and any future layer addition requires a sub-RFC under this mechanism. Drift between implementations is therefore the larger risk, and INTEGER is the lower-drift choice.

A conforming implementation MUST NOT redefine the precedence ranks. The integer values above are normative.

### §5. Specification — Supersession-at-read-time

A factlet MAY carry a `superseded_by` field whose value is the ID of the factlet that replaces it. A factlet with `superseded_by IS NULL` is **canonical**; a factlet with a non-null `superseded_by` is **historical**.

#### 5.1. Default retrieval filter

A conforming retrieval API MUST filter `WHERE superseded_by IS NULL` by default. Default behavior returns only canonical factlets. Historical factlets are stored but not surfaced by default queries.

#### 5.2. Opt-in to history

A retrieval caller MAY pass `include_superseded=true` (parameter name normative) to opt into full history. When that flag is set, the API MUST return both canonical and historical factlets, and SHOULD annotate each historical factlet with a reference to its `superseded_by` successor for downstream audit.

#### 5.3. Why read-time (not write-time)

The supersession-at-read-time invariant is a deliberate separation of two concerns: (a) the write-time act of marking a factlet historical (one author asserting that another factlet replaces it) and (b) the read-time decision to surface only the canonical view. Read-time filtering keeps historical factlets queryable for audit and reconciliation without polluting default retrieval.

A naive write-time alternative — physically deleting superseded factlets on write — would lose the audit trail and prevent reconciliation when supersessions are themselves contested. The read-time approach is observed to be the cheaper invariant to maintain: writes are simple (set `superseded_by`), reads are simple (`WHERE superseded_by IS NULL`), and audit is always possible.

#### 5.4. Supersession authority

A factlet MAY only be superseded by a factlet at the **same layer** (intra-layer revision, see below) or at an **ancestor layer** in its `can_be_overridden_by` set (§4). Because `precedence_rank` is a topological order of the §3 ancestor DAG, every permitted superseding layer necessarily has `precedence_rank ≤ R`; the converse does **not** hold — a lower-rank **lateral peer** (e.g. `Project` rank 6 vs `Team` rank 7, where neither is the other's ancestor) MUST NOT supersede across the lateral boundary. A Codebase factlet (rank 8) MUST NOT carry `superseded_by` pointing at a kp whose layer_type is `Individual` (rank 9), because Individual is a sovereign root and is not an ancestor of any employer layer. Cross-layer supersessions that violate ancestry direction MUST be rejected by the conforming write API.

A factlet MAY be superseded by another factlet at the same `precedence_rank` (e.g. one Project factlet supersedes another Project factlet on the same `conflict_group_id`). This is the common case of intra-layer revision.

#### 5.5. Supersession is not deletion

Setting `superseded_by` does NOT delete the historical factlet. Historical factlets remain in storage with their original `sources`, `confidence`, and `created_at`. They simply cease to appear in default retrieval. A conforming implementation MUST NOT prune historical factlets without an explicit retention-policy operation outside the scope of normal retrieval.

### §6. Specification — Three roots: Individual + Company + Domain-Pack

A conforming Factbook tree has exactly three valid root types — kp records with no `parent_kp_id`:

- **Individual** — per-user, portable across employers. Carries preferences, working style, communication conventions. Travels with the user.
- **Company** — per-employer, employment-tied. Carries organizational policy, compliance posture, internal vocabulary. Stays with the employer when an individual leaves.
- **Domain-Pack** — per-vertical, registry-published. Carries domain knowledge (manufacturing standards, healthcare regulation citations, legal jurisdictional rules). Orthogonal to both Individual and Company trees.

Any other layer MUST have a parent per §3. A kp with `layer_type` in `{Project, Program, Product, Service, Team, Division, Codebase}` and `parent_kp_id IS NULL` is non-conforming.

#### 6.1. Domain-Pack orthogonality

A Domain-Pack is NOT a child of Company. It is a root in its own right. The Domain-Pack tree is orthogonal to the Company tree: a single Codebase may compose (via RFC 0004 `dependencies`) against both a Company-rooted ancestry and a Domain-Pack-rooted ancestry.

Cross-cutting classification — "this Codebase operates in the manufacturing domain" — SHOULD NOT be expressed via `parent_kp_id` (which would force the Codebase into the Domain-Pack tree and break its Company ancestry). It SHOULD instead be expressed via the `domain_tags` field defined in §6.2.

#### 6.2. The `domain_tags` field

A kp MAY carry a `domain_tags` field, a JSON array of Domain-Pack identifiers. This declares cross-cutting domain classification without altering the parent_kp_id graph. Example:

```yaml
kp_id: kp-codebase-payment-service
layer_type: Codebase
parent_kp_id: kp-project-checkout
domain_tags:
  - domain-pack:pci-dss-4.0
  - domain-pack:payments-glossary-2026
```

Retrieval against this Codebase MAY include factlets from the named Domain-Packs, subject to the composition rules in RFC 0004 (dependencies) and the precedence rules in §4.

#### 6.3. Mixed-root composition

A retrieval operation MAY span multiple roots. A query rooted at a Codebase kp resolves:

1. The Codebase kp's ancestry chain (via parent_kp_id) up to its Company root.
2. The Codebase kp's `domain_tags` Domain-Pack roots.
3. The current operator's Individual root (if known to the implementation).

Factlets from all three are merged. Conflicts are resolved by §4 precedence (with Domain-Pack orthogonality preserved — Domain-Pack factlets do not enter the rank comparison; they are merged as advisory unless their own `compliance_tier` says otherwise).

### §7. Migration path from RFC-0002

RFC-0002 specifies the Profiles mechanism but does not require Factbooks to be layered. Pre-RFC-0007 Factbooks are flat — a single `content:` list with no kp grouping. RFC-0007 does not break those Factbooks; it adds an optional organizational structure for implementations that need it.

#### 7.1. Flat-to-layered transition

A v0.1 or pre-RFC-0007 v0.2 Factbook MAY be migrated to a layered Factbook by:

1. Creating a single kp record with `layer_type` chosen by the author (typically `Codebase` for a software project Factbook, `Company` for a top-level policy Factbook, `Individual` for a personal Factbook).
2. Assigning all existing factlets to that kp via a `kp_id` field.
3. Optionally introducing parent kps (e.g. a `Project` kp for a multi-Codebase team) and re-pointing the Codebase kp's `parent_kp_id` accordingly.

A conforming implementation MUST treat a Factbook without `kp_id` on its factlets as a degenerate single-kp Factbook (one implicit kp containing all factlets). The implicit-kp's `layer_type` MAY be declared at the file root as `default_layer_type`; if absent, the implementation SHOULD treat it as `Codebase` for software Profiles and as `Domain-Pack` for vertical Profiles, and SHOULD log a warning identifying the assumed default.

#### 7.2. Tool-call interface migration

Existing retrieval tool calls that accept a single Factbook path SHOULD be extended to accept an optional `kp_id` parameter scoping retrieval to a subtree. Calls without `kp_id` retrieve across the full Factbook (the v0.1 behavior). This is forward-compatible: pre-RFC-0007 callers see no change; layer-aware callers can scope.

#### 7.3. Schema fragments

Added to `schema/factbook.schema.json`:

```json
"default_layer_type": {
  "type": "string",
  "enum": ["Individual", "Codebase", "Project", "Program", "Product",
           "Service", "Team", "Division", "Company", "Domain-Pack"],
  "description": "Layer type for the implicit kp containing all factlets in a flat Factbook. Used only when factlets lack an explicit kp_id."
},
"kps": {
  "type": "array",
  "description": "Knowledge-pack records grouping factlets into a layered organizational graph.",
  "items": { "$ref": "#/$defs/kp" }
}
```

Added to `schema/kp.schema.json` (new schema introduced by this RFC):

```json
{
  "$defs": {
    "kp": {
      "type": "object",
      "required": ["kp_id", "layer_type"],
      "properties": {
        "kp_id":         { "type": "string" },
        "layer_type":    {
          "type": "string",
          "enum": ["Individual", "Codebase", "Project", "Program", "Product",
                   "Service", "Team", "Division", "Company", "Domain-Pack"]
        },
        "parent_kp_id":  { "type": ["string", "null"] },
        "precedence_rank": { "type": "integer", "minimum": 1, "maximum": 9 },
        "compliance_tier": {
          "type": "string",
          "enum": ["non_overridable", "standard", "advisory"],
          "default": "standard"
        },
        "domain_tags":   {
          "type": "array",
          "items": { "type": "string" }
        },
        "display_name":  { "type": "string" }
      }
    }
  }
}
```

Added to `schema/factlet.schema.json`:

```json
"kp_id": {
  "type": "string",
  "description": "Knowledge-pack containing this factlet. Required for layered Factbooks; optional for flat Factbooks (defaults to the implicit single kp)."
},
"conflict_group_id": {
  "type": "string",
  "description": "Identifier shared by factlets that may conflict. Conflict resolution uses precedence_rank of the containing kp."
},
"superseded_by": {
  "type": ["string", "null"],
  "description": "If non-null, the ID of the factlet that supersedes this one. Default retrieval filters WHERE superseded_by IS NULL."
}
```

#### 7.4. Display-name customization

A kp MAY carry a `display_name` field that overrides the canonical `layer_type` for UI rendering. Canonical `layer_type` MUST remain in storage and MUST be the value returned by any tool-call interface (e.g. retrieval RPC); `display_name` is presentation-layer only.

Use cases observed: a company that uses the term "Squad" instead of "Team" sets `display_name: "Squad"` on its Team-layer kps. The retrieval API still returns `layer_type: "Team"` so downstream consumers can apply uniform precedence rules; the UI renders "Squad" for human readability.

A conforming implementation MUST NOT use `display_name` for precedence resolution, conflict matching, or any other semantic operation. It is opaque to the protocol.

### §8. Open questions

- **Should `precedence_rank` be declared at the kp level (one per kp) or at the layer-type level (one per layer)?** Initial design uses layer-level (the §4 canonical assignment is by layer_type). A future RFC may allow per-kp override for organizations where a specific subsidiary needs to override its parent (e.g. a regulated-Division Company subsidiary whose Division precedence is higher than its parent Company). Per-kp override would require a precedence-cycle check analogous to §3.1.

- **Should `Domain-Pack` factlets participate in precedence at all, or remain purely advisory?** Initial design treats Domain-Pack as orthogonal (no rank). An alternative: Domain-Pack factlets carry their own `precedence_rank` interpreted as a separate dimension, allowing a regulatory Domain-Pack to be non-overridable. The `compliance_tier` mechanism (§4.2) covers most of this need today; whether the rank dimension is also useful is empirically open.

- **Should the 10-layer enum be extensible by sub-RFC, or fixed for the life of v0.x?** Initial position: fixed for v0.x. Adding a layer requires renumbering precedence ranks and re-validating every conforming implementation. A future RFC may introduce a registered-layer namespace analogous to the Profiles registry (RFC 0002 §3), under which an organization could register a custom layer with a `custom:` prefix and its own precedence assignment. This is deferred until empirical demand exceeds what the 10-layer enum supports.

- **How does this interact with FactSignal (v0.1 §7)?** Initial recommendation: FactSignal is computed against the canonical set (post-supersession filter). Profile-aware scoring weights MAY incorporate `precedence_rank` (e.g. weight Company-rank factlets higher in regulated-domain queries). A future RFC may standardize a precedence-aware scoring extension.

- **Should `domain_tags` be hierarchical (parent Domain-Pack implies child)?** Initial design treats `domain_tags` as a flat list. A future RFC may introduce Domain-Pack-to-Domain-Pack composition via RFC 0004 `dependencies`, in which case the dependency graph implicitly extends the tag set.

### §9. Conformance

A v0.2-conforming implementation that claims layer support MUST:

1. Parse the `kps` array at Factbook root and the `kp_id`, `conflict_group_id`, `superseded_by` fields on individual factlets.
2. Enforce the parent-layer compatibility matrix (§3) at write time — reject any operation that would violate it. (The §3.1(b) offline-integrity-check path is permitted only as a transitional backfill measure, after which write-time enforcement applies.)
3. Enforce cycle prohibition (§3.1) at write time.
4. Compute conflict resolution per the precedence-rank table (§4). Implementations MUST NOT redefine the integer values.
5. Honor `compliance_tier='non_overridable'` (§4.2): return non-overridable factlets in retrieval regardless of supersession by lower-authority layers.
6. Apply the supersession-at-read-time filter (§5.1) by default. Honor `include_superseded=true` (§5.2) as opt-in.
7. Treat `Individual`, `Company`, `Domain-Pack` as the three valid root types (§6). Reject any non-root kp with `parent_kp_id IS NULL`.
8. Treat `domain_tags` as cross-cutting classification (§6.2), not as a parent-link substitute.
9. Preserve `display_name` on round-trip but not use it for any semantic operation (§7.4).

Conformance is tested against the portability test suite in RFC 0004 (extended for v0.2): a test Factbook with all 10 layer types instantiated, a parent_kp_id graph exercising every row of the §3 matrix, a `conflict_group_id` exercising every distinct precedence pair in §4, and a non-trivial supersession chain. A conforming implementation MUST round-trip the test Factbook with all layer, precedence, and supersession semantics preserved.

An implementation that does NOT claim layer support (a pre-RFC-0007 v0.2 implementation, or a layer-naive reader) MUST:

1. Treat `kps`, `kp_id`, `conflict_group_id`, `superseded_by`, `domain_tags`, `compliance_tier`, and `display_name` as unknown keys (preserve on round-trip per v0.1 §9.3).
2. NOT reject a layered Factbook on the basis of unknown layer-related fields.
3. SHOULD log a warning identifying the unsupported layer fields so the operator can decide whether to install layer support.

This forward-compatibility rule is identical in shape to the RFC 0002 §4 forward-compat rule for unknown Profiles.

### §10. Reference implementation

A reference implementation of this RFC is published as part of the v0.2 reference SDK (factlet-ai/reference-sdk). The reference SDK ships:

- A 10-layer SQL schema (one `kps` table with a `layer_type` enum column and a `parent_kp_id` self-referential foreign key; a `factlets` table with `kp_id`, `conflict_group_id`, `superseded_by` columns).
- A parent-layer compatibility-matrix check enforced via a CHECK constraint plus a write-time trigger.
- A recursive-CTE cycle check (§3.1) at the write path that writes parent links.
- A retrieval RPC that applies §4 precedence and §5.1 supersession filtering by default.
- Test fixtures covering all 10 layers, the full §3 matrix, the full §4 precedence table, and supersession chains of depth ≥ 3.

The SQL mapping above is one valid mapping under the §3–§6 normative clauses. Other implementations MAY map the layer model into a graph database, a document store, or an in-memory structure provided the §9 conformance criteria are met. The SQL mapping is illustrative, not normative.

Limits of the reference implementation as of v0.2:

- Lateral conflicts (a `Project` factlet vs a `Team` factlet on the same `conflict_group_id`, where neither layer is the other's ancestor per §3) are surfaced for caller arbitration; the reference SDK does not auto-resolve. This is by design (§4.1) but may be a usability gap for callers without a UI affordance for resolution.
- The recursive-CTE cycle check is O(depth) per write. For organizations with deep layer chains (Codebase → Project → Program → Division → Company is the maximum at 5), this is acceptable; deeper chains via custom layers (open question §8) would benefit from a materialized path.
- The §6.2 `domain_tags` field is enforced as syntactically valid (a JSON array of strings) but not as referentially valid (each tag pointing at an installed Domain-Pack). Reference validation is the responsibility of the Domain-Pack registry, addressed separately in RFC 0004 dependency resolution.

## Alternatives considered

### A. Single hierarchy with Company as the only root

Collapse Individual and Domain-Pack into children of Company. A user's portable preferences would be a kp under Company-rooted Individual; a Domain-Pack would be a Company-rooted child.

**Rejected because:** the portability boundary (§2 third failure mode) is not expressible. A user moving between employers cannot take an Individual kp that is structurally a child of the previous employer's Company root. Forcing the Individual kp to be re-parented on every employment change is observed to be the source of factlet loss in practice. Three orthogonal roots preserve the boundary.

### B. Flat factbook with `layer_tags` only

Tag each factlet with a free-form layer label (e.g. `layer:codebase`, `layer:company`) and let retrieval filter by tag.

**Rejected because:** tags are open-namespace and unenforceable (same failure mode as RFC 0002 §B). Two implementations diverge on conventions (`layer:codebase` vs `level:code` vs `scope:repo`). Without a normative enum and a normative parent-layer matrix, the precedence rules (§4) cannot be computed reliably across implementations.

### C. Precedence rank as a per-factlet scalar (not per-layer)

Let each factlet declare its own `precedence_rank` independent of the kp it belongs to.

**Rejected because:** per-factlet rank would let a Codebase factlet claim rank 1 (Company authority), defeating the layer model's audit value. The whole point of a layer-keyed precedence is that retrieval and audit can rely on the layer of the kp containing a factlet without inspecting each factlet's claims. Per-kp override (open question §8) is a softer alternative that retains the audit property.

### D. Read-time filtering of supersession only via flag (no default)

Require callers to opt OUT of historical factlets (`include_superseded=false`) rather than opt in.

**Rejected because:** observed retrieval patterns show that returning historical factlets by default leaks contradictions into LLM context. The default-canonical view (§5.1) is the safer behavior. Audit callers explicitly opt in to history (§5.2). The cost of the opt-in is one boolean per audit query; the cost of opt-out by default is silent factual contradiction in every retrieval.

### E. Inline layer fields on factlets (no kp grouping)

Put `layer_type` and `parent_kp_id` directly on each factlet rather than on a kp record.

**Rejected because:** kps are observed to carry several factlets each. Inlining the layer fields onto every factlet duplicates the parent_kp_id N times per kp, multiplies the storage cost, and complicates supersession (an entire kp moves to a new parent in one write, vs. N writes to N factlets). The kp record is the right granularity for organizational structure; factlets remain the granularity for individual truths.

## Migration impact

- **For Factbook authors**: no required change. Existing v0.1 and pre-RFC-0007 v0.2 Factbooks remain valid v0.2 Factbooks (treated as flat / single-kp per §7.1). Authors who want layer-specific retrieval and precedence MAY add a `kps` block and a `kp_id` on each factlet.
- **For implementations**: implementations that do not claim layer support MUST preserve the new fields on round-trip (§9 second list). Implementations that claim layer support MUST enforce the full §3-§6 semantics.
- **For domain-Profile authors**: a registered Profile (RFC 0002 §3) MAY specify a default `layer_type` for Factbooks declared under that Profile, and MAY specify per-factlet layer-type constraints (e.g. a manufacturing-Profile factlet of `factlet_type: hazard` MUST be at `layer_type: Codebase` or higher authority). These are Profile-scoped extensions of this RFC and live in the Profile's `SPEC.md`.
- **For consumers of RFC 0004 `dependencies`**: dependency declarations MAY specify `kp_id` constraints (e.g. depend only on the `kp-pci-dss-4.0` Domain-Pack kp rather than the full Domain-Pack Factbook). This is forward-compatible: pre-RFC-0007 dependency consumers ignore the new `kp_id` constraint and resolve at file granularity.
- **Breaking change?** No. All new fields are optional and additive. Layer-naive v0.2 readers ignore them per §9. Layer-aware v0.2 readers enforce them.

## Acceptance criteria

- Two maintainer approvals OR working group consensus.
- Update to `SPEC.md`: reference the RFC 0007 layer model from §15.5 (Other v0.2 base-spec extensions) — done in this change; a dedicated SPEC §16 inlining the normative tables MAY follow in a later editorial pass. Add an "Organizational layering — Resolved by RFC 0007" entry to the §13 open-questions list (no prior "Organizational layering" entry exists to toggle, so this adds the resolved record).
- Create `factlet-ai/spec/schemas/kp.schema.json` with the schema fragment in §7.3.
- Update `factlet-ai/spec/schemas/factbook.schema.json` (add `default_layer_type` and `kps` properties).
- Update `factlet-ai/spec/schemas/factlet.schema.json` (add `kp_id`, `conflict_group_id`, `superseded_by`).
- Reference SDK (factlet-ai/reference-sdk) PR adding: (a) parsing and validation of the new fields, (b) a parent-layer matrix check, (c) a cycle check, (d) a precedence-aware retrieval RPC with default `WHERE superseded_by IS NULL` filtering, (e) test fixtures covering all 10 layers, the full §3 matrix, the full §4 table, and a supersession chain of depth ≥ 3.
- RFC 0004 (Composable Factbooks via dependencies) updated to reference §6.2 `domain_tags` and §7.4 display-name handling.
- Conformance test suite extended per §9 with a multi-layer test Factbook exercising every matrix row, every precedence pair, and a non-trivial supersession chain.
