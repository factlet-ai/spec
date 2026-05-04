# Factlet Protocol — specification

Open specification for portable, vendor-neutral AI grounding. The five-primitive protocol (factlet, FactMap, Factbook, FactSignal, low-FactSignal warning) defines how private team knowledge flows into LLM answers and how to measure when grounding coverage runs out.

## Read the spec

- **[SPEC.md](SPEC.md)** — the v0.1 specification (RFC 2119 normative language, ~14 sections)
- **[schema/factlet.schema.json](schema/factlet.schema.json)** — JSON Schema for individual factlet records
- **[schema/factbook.schema.json](schema/factbook.schema.json)** — JSON Schema for the Factbook container
- **[examples/payments-factbook.yaml](examples/payments-factbook.yaml)** — complete worked example

## Contribute

- **Discussion / question** → open a [GitHub Discussion](https://github.com/factlet-ai/spec/discussions)
- **Substantive change** (new field, new primitive, breaking schema) → open an RFC PR using [rfcs/0000-template.md](rfcs/0000-template.md)
- **Spec text typo or ambiguity** → open an Issue with section number and proposed clarification
- **Security issue** → see [SECURITY.md](SECURITY.md), do not open a public issue

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full RFC process.

## Status

**v0.1 draft.** v0.2 ships within 90 days incorporating community feedback. Pre-v1.0 minor versions MAY include breaking changes; after v1.0, breaking changes require a major version bump.

## Related repositories

- **[factlet-ai/reference-sdk](https://github.com/factlet-ai/reference-sdk)** — reference implementations in Python and TypeScript
- **[factlet-ai/registry](https://github.com/factlet-ai/registry)** — community-contributed example Factbooks across domains
- **[factlet.ai](https://factlet.ai)** — website + getting-started prompts
- **[factlet.ai/llms.txt](https://factlet.ai/llms.txt)** — LLM-discovery file

## License

MIT — see [LICENSE](LICENSE).
