# Factlet Protocol — specification

Open specification for portable, vendor-neutral AI grounding. Read the full protocol at [factlet.ai/protocol](https://factlet.ai/protocol).

## Status

**v0.1 draft.** Open RFC process — see [Discussions](https://github.com/factlet-ai/spec/discussions) for active proposals and [`rfcs/`](rfcs/) for accepted RFCs. v0.2 ships within 90 days incorporating feedback.

## Scope

The five-primitive protocol:

- **Factlet** — one atomic truth about your private information
- **FactMap** — the structured collection of factlets covering one body of work
- **Factbook** — a packaged FactMap, versioned and portable
- **FactSignal** — coverage strength at a query, measured in bars (0-5)
- **Low-FactSignal warning** — runtime callback when a model is about to answer in a dead zone

This repository contains the specification text, JSON Schema, and the RFC change log. Implementations live in [`factlet-ai/reference-sdk`](https://github.com/factlet-ai/reference-sdk). Example factbooks live in [`factlet-ai/registry`](https://github.com/factlet-ai/registry).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the RFC process. All participation requires adherence to the [Code of Conduct](CODE_OF_CONDUCT.md).

## Security

Vulnerability disclosure: see [SECURITY.md](SECURITY.md).

## License

MIT — see [LICENSE](LICENSE).
