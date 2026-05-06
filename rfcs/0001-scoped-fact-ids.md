# RFC 0001: Scoped Fact IDs

- **Status**: Discussion
- **Author(s)**: Mihir Choudhary <mihir@kernora.ai>
- **Created**: 2026-05-06
- **Last updated**: 2026-05-06
- **Discussion**: TBD (link to GitHub Discussion thread when opened)
- **Target spec version**: v0.2

## Summary

Introduce a mandatory `<scope>:<id>` citation form for factlet IDs whenever they are referenced outside their host Factbook (in code, prose, other factbooks, downstream tooling). Bare IDs (`f001`) remain valid INSIDE a Factbook file because the file-level `scope:` field provides unambiguous resolution. EXTERNAL references MUST use the prefixed form (`kernora:f001`, `factlet-ai:f001`).

## Motivation

The v0.1 spec (§3.1) defines factlet IDs as "Stable identifier, unique within a Factbook." This guarantees uniqueness within one file but **not** across files. As soon as a consumer (an LLM, a tool, a downstream factbook) loads multiple Factbooks into the same context, ID collisions become a real failure mode.

### Concrete failure observed 2026-05-06

In a single Claude Code session working on both the Kernora commercial product and the open factlet-ai project:

- `kernora-factbook.yaml` had `f001` meaning "Phase 0 sprint dates" (scope: project:kernora)
- `factlet-factbook.yaml` had `f001` meaning "Cloudflare Pages deploy is unreliable on direct push" (scope: project:factlet-ai)
- A draft prose reference to "per f001" was unresolvable without checking which file
- One factbook (factlet-factbook) was even authored INTO the wrong repo (kernora's `.nora/`) — the bare-ID convention provides no on-disk signal that two scopes are now sharing an ID space

The model's training prior says "f001 is unique." The team's truth is that f001 is unique *within a file* but ambiguous *across files*. This is exactly the model-prior-vs-team-truth gap the protocol was designed to surface — and the v0.1 spec doesn't mandate the citation discipline that prevents it.

### Why this matters at scale

- An eval suite that scores "did the model cite must_cite factlets" needs a canonical citation form. If models cite `f002` and the must_cite list says `[f002]`, which `f002`?
- A team running multiple Factbooks (e.g. `payments` + `frontend` + `compliance`) must distinguish them in code review and in production logs.
- A vendor publishing factbooks for downstream consumers (e.g. Stripe ships an official `stripe-payments-factbook`) must guarantee its IDs don't collide with team-internal facts.

The fix is a citation rule, not a schema change. No existing factbook needs to be re-numbered; only the way facts are referenced *outside* the file changes.

## Detailed design

### 1. Internal references (inside a Factbook file)

Bare IDs remain valid inside a Factbook because the file-level `scope:` field provides the prefix unambiguously.

```yaml
scope: project:factlet-ai
content:
- id: f001              # ✅ still valid inside the file
  statement: "Cloudflare Pages..."
  supersedes: []
- id: f002
  statement: "Factbooks for distinct projects..."
  related_facts:
    - "f001 — see workaround section"   # ✅ INTERNAL reference, bare form OK
```

### 2. External references (citing a fact from outside its host file)

External references **MUST** use the `<scope>:<id>` form.

```yaml
# Different factbook, referencing a fact in factlet-ai's factbook
content:
- id: f045
  statement: "Our deploy uses the same gotcha as factlet-ai:f001..."   # ✅ scoped
  related_facts:
    - "factlet-ai:f001 — Cloudflare Pages flakiness"                    # ✅ scoped
```

```python
# Code referencing a factlet
# ✅ canonical
factbook.cite("factlet-ai:f001")

# ❌ ambiguous — DEPRECATED in v0.2
factbook.cite("f001")
```

```markdown
# Prose / commit messages / PR descriptions
✅ "per factlet-ai:f001, the deploy may not auto-fire"
❌ "per f001, the deploy may not auto-fire"
```

### 3. Scope syntax

Scope is the value of the file-level `scope:` field, used as the prefix verbatim.

```
scope: project:kernora       → citation form: project:kernora:f001
                                  shortened (per §4):  kernora:f001
scope: team:payments         → citation form: team:payments:f001
                                  shortened:           payments:f001
scope: vendor:stripe         → citation form: vendor:stripe:f001
                                  shortened:           stripe:f001
```

### 4. Shortened citation form (recommended)

The full form `<scope_kind>:<scope_value>:<id>` is verbose. The shortened form drops the scope kind when context is clear:

- `factlet-ai:f001` (shortened) ≡ `project:factlet-ai:f001` (full)
- `payments:f001` (shortened) ≡ `team:payments:f001` (full)
- Tools **MUST** accept both forms.
- Authors **SHOULD** use the shortened form in prose; tools **MUST** emit the full form in machine-readable artifacts (logs, JSON, scoring outputs).

### 5. Validation

Implementations **SHOULD** ship a validator that:

- Parses the file-level `scope:` field
- For each fact, ensures `id` is unique within the file
- For each `related_facts`, `must_cite`, `must_not_contradict`, `supersedes`, `merged_into` reference, asserts:
  - If the reference uses bare form (`f001`), the target MUST exist in this file.
  - If the reference uses scoped form (`<scope>:<id>`), the target SHOULD exist in a Factbook with that scope (validator MAY be lenient on cross-repo lookups).

Reference impl: `factlet-ai/reference-sdk` v0.2 will ship `factlet validate --strict-scoping`.

## Alternatives considered

### A. Mass-rename all fact IDs to be globally unique

Reject. Breaking change to every existing Factbook. The kernora-factbook alone has ~270 facts; renumbering would destroy git blame and break every downstream citation.

### B. Hash-based UUIDs for fact IDs (e.g. `f-a7f3c2`)

Reject. Loses human readability. The whole point of `f001` is that humans can read and remember it. Hashes optimize for machine collision-avoidance at the cost of the human ergonomics that make Factbooks useful.

### C. File-name-based prefix (e.g. `kernora-factbook.f001`)

Reject. Couples scope to filesystem layout. A Factbook can be renamed, moved between repos, or distributed under different filenames; the canonical scope identifier should not change with the file location.

### D. Leave it to convention; no spec change

Reject. The whole point of a protocol is to enforce the discipline that prevents recurring failures. The 2026-05-06 incident shows that without a spec rule, even the protocol author makes the mistake. A protocol that doesn't prevent the failure mode it was designed to address is theater.

## Migration impact

**Spec version**: ships in v0.2. Backward-compatible for **internal** references (bare `f001` still works inside a file). Required change for **external** references and for any tool that emits citations.

### For Factbook authors

No required changes to existing Factbook files. Internal references remain bare. Optionally, authors may begin using the scoped form in `related_facts` etc. for clarity.

### For Factbook consumers / readers (LLMs, tools)

When emitting citations to logs, scoring outputs, prose responses, or other Factbooks, MUST use the scoped form.

### For tooling (validators, renderers, the reference SDK)

- Validators ship with `--strict-scoping` flag for v0.2 conformance
- Renderers (XML/markdown/etc. per §8) emit fact IDs in scoped form
- LLM-facing prompts (e.g. "cite factlet IDs with scope prefix") get updated

### For the public eval suite (factlet-ai/evals)

Update the `citation` judge prompt to require the scoped form when grading. Update `must_cite` task fields to use scoped IDs (or document the implicit scope from the task's `factbook` field).

## Open questions

1. **Should the full-form (`project:kernora:f001`) be required somewhere**, or is the shortened form (`kernora:f001`) always acceptable? Current draft says shortened is the recommended human form, full is recommended for machine-readable. Worth pressure-testing.

2. **What does the LLM facing render look like?** If `tier1/factbooks/payments-factbook.yaml` has `scope: team:payments` and we render it for Claude, do we put `payments:f001` in the XML, or `f001`? Probably scoped, but spec needs to be explicit.

3. **Cross-Factbook validation in CI** — how strict? At minimum the validator confirms the bare-form references resolve within the file. Cross-repo resolution (does `kernora:f001` actually exist in some Factbook somewhere?) requires either a registry or trust-based skipping. v0.2 may defer cross-repo validation to v0.3.

4. **Migration for the 2026-05-04 launch artifacts** — the X thread, Substack, LinkedIn posts (drafted but unpublished as of 2026-05-06) reference factlet IDs by bare form. Update before publish?

## Provenance

Filed as direct dogfood: this RFC was written the same day (2026-05-06) the failure mode was observed in a single Claude Code session. See `factlet-ai:f002` in [factlet-ai/factbook](https://github.com/factlet-ai/factbook) for the operational fact this proposal addresses.
