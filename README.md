# Joe Severino

Technical Solutions Engineer at World Wide Technology. Most of my projects are built around real systems I run myself: a self-hosted operations hub, local AI tooling with explicit safety boundaries, zero-trust homelab infrastructure, private PKI, and the vault-to-website pipeline behind my portfolio.

## Featured Projects

- **[severino-hq](https://github.com/joeseverino/severino-hq)** - Self-hosted operations hub that derives its picture of my infrastructure from the credentials it holds. Every change goes through one gated path shared by the web UI, API, MCP, and CLI.
- **[jseverino.com](https://github.com/joeseverino/jseverino.com)** - My site and writeups, published from a private Obsidian vault with Astro on Cloudflare Pages. Strict nonce CSP, edge-tested, and released with signed provenance.
- **[severino-vault-mcp](https://github.com/joeseverino/severino-vault-mcp)** - Local-first MCP server that gives AI tools sensitivity-gated access to an operations vault, built on [vault-engine](https://github.com/joeseverino/vault-engine), its reusable core on PyPI.
- **[sitedrift](https://github.com/joeseverino/sitedrift)** - [npm package](https://www.npmjs.com/package/sitedrift) for reviewing DEV against LIVE on the same route, with a Cloudflare Pages addon, an MCP interface, and SLSA provenance.
- **[tools](https://github.com/joeseverino/tools)** - Personal macOS CLI suite where one declaration per tool renders its help, completions, docs, and machine-readable spec.
- **[cordon](https://github.com/joeseverino/cordon)** - Command-surface contract: declare a CLI once, render every view from it, and give each command a blast radius an agent can check before it acts.
- **[cert-generator](https://github.com/joeseverino/cert-generator)** - CLI that issues TLS certificates from a private root CA kept on an offline VM.

## How It Fits Together

A private Obsidian vault holds the knowledge and the content. Severino HQ holds
the infrastructure, read from the providers themselves.

![On the Mac, the vault MCP, used by AI sessions and the tools CLI, reads and writes the Obsidian vault and sends its docs to Severino HQ on the homelab; HQ reads and manages Cloudflare zones and Access with a scoped token; the vault's published subset becomes jseverino.com on Cloudflare Pages](docs/diagrams/system-map.png)

<sup>Diagram source: [`docs/diagrams/system-map.fig`](docs/diagrams/system-map.fig),
pre-rendered with [`brand figure`](https://github.com/joeseverino/branding-engine).</sup>

The pieces, how they connect, and why are in
**[ARCHITECTURE.md](ARCHITECTURE.md)**.

**Certifications:** CCNA, CompTIA Security+, ISC2 Certified in Cybersecurity (CC)
