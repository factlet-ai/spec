# Profile Registry

Registered profiles known to the Factlet Protocol v0.2. Each profile defines domain-specific extensions on top of the base spec.

| Profile | Status | Spec | RFC | Maintainer |
|---|---|---|---|---|
| [`software-engineering`](software-engineering/SPEC.md) | Stable (v0.2) | [SPEC.md](software-engineering/SPEC.md) | [RFC 0005](../rfcs/0005-software-profile-phase-enum.md) | factlet-ai/spec working group |
| [`manufacturing`](manufacturing/SPEC.md) | Draft (v0.1) | [SPEC.md](manufacturing/SPEC.md) | [RFC 0006](../rfcs/0006-manufacturing-profile.md) | factlet-ai/spec working group |

## Adding a profile

1. Copy [`.template/`](.template/) to `<your-name>/`.
2. Author the profile spec in `<your-name>/SPEC.md`.
3. Define your factlet schema additions in `<your-name>/factlet.schema.json` (and `factbook.schema.json` if you extend the Factbook root).
4. Add at least one worked example in `<your-name>/examples/`.
5. Open a sub-RFC referencing the new profile + acceptance criteria.
6. Add your profile to the table above.

Profile names MUST match `^[a-z][a-z0-9-]{0,63}$` and SHOULD use kebab-case.
