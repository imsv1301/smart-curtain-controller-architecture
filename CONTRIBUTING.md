# Contributing

This repository documents the architecture of a proprietary firmware system developed at CosmicX. The firmware source is not included — contributions here are limited to documentation.

---

## What you can contribute

- Corrections to technical descriptions (wrong register addresses, incorrect GPIO numbers, typos)
- Improvements to diagrams or text explanations in `docs/` or `architecture/`
- Translations of documentation into other languages
- Suggestions for additional documentation topics (open an issue)

---

## What this repo does not accept

- Firmware source code of any kind
- PCB designs, schematics, or Gerber files
- Bill of Materials or component sourcing information
- Any CosmicX proprietary material

---

## Process

1. Open an issue describing what you want to change and why
2. Fork the repository
3. Make changes in a branch named `fix/description` or `docs/description`
4. Open a pull request against `main` with a clear description

For small corrections (typos, broken links), a pull request without an issue is fine.

---

## Writing style

Documentation in this repo aims to be:

- **Specific over vague** — "GPIO21, active LOW" rather than "the enable pin"
- **Concrete over abstract** — measurements and register values wherever possible
- **Direct** — no preamble, no "In this section we will explore..."

If you're not sure whether something is accurate, note the uncertainty explicitly rather than writing around it.

---

## License

Contributions are accepted under the MIT license. See [LICENSE](./LICENSE).
