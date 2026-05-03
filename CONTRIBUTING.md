# Contributing to the Factlet Protocol specification

Thank you for your interest in shaping the protocol. The spec is owned by the community via an open RFC process.

## Quick paths

- **Question or discussion** → open a [GitHub Discussion](https://github.com/factlet-ai/spec/discussions). Most spec questions belong here, not in issues.
- **Bug or ambiguity in the published spec** → open an [Issue](https://github.com/factlet-ai/spec/issues) with the section number and proposed clarification.
- **Substantive change** (new field, new primitive, breaking schema change) → open an **RFC PR** in `rfcs/`. See template in `rfcs/0000-template.md`.
- **Security issue** → see [SECURITY.md](SECURITY.md). Do not open a public issue.

## RFC process

1. Fork the repo and copy `rfcs/0000-template.md` to `rfcs/00xx-short-title.md` (use the next free number).
2. Fill in motivation, design, alternatives considered, and migration impact.
3. Open a PR. The RFC is "in discussion" until two maintainers approve OR a working-group meeting reaches consensus.
4. Accepted RFCs land in the spec text under the appropriate section, and the RFC file is moved to `rfcs/accepted/`.
5. Rejected RFCs are kept in `rfcs/rejected/` with the rejection reason recorded — useful future reference.

## Spec text style

- Use **MUST / SHOULD / MAY** per [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).
- Cite section and field names exactly as defined; no synonyms.
- Provide at least one machine-readable example (JSON Schema or YAML) for any new field.

## Versioning

The spec follows semantic versioning at the document level: `v<major>.<minor>`. Pre-v1.0, breaking changes may occur in any minor release; after v1.0, breaking changes require a major version bump.

## Code of Conduct

Participation requires adherence to the [Code of Conduct](CODE_OF_CONDUCT.md).
