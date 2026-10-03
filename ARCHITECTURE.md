# How My Systems Fit Together

Two sources of truth, each for one kind of fact. A private Obsidian vault holds
what I know and what I publish: runbooks, decision records, writeups. Severino
HQ holds what is running: hosts, DNS, certificates, and the paths between them,
read from Cloudflare and Tailscale rather than typed into a file. Everything
else reads from one of the two.

This page is the map and the reasoning. Each repository documents its own
internals; the build stories are on [jseverino.com](https://jseverino.com).

![On the Mac, the vault MCP, used by AI sessions and the tools CLI, reads and writes the Obsidian vault and sends its docs to Severino HQ on the homelab; HQ reads and manages Cloudflare zones and Access with a scoped token; the vault's published subset becomes jseverino.com on Cloudflare Pages](docs/diagrams/system-map.png)

<sup>Diagram source: [`docs/diagrams/system-map.fig`](docs/diagrams/system-map.fig),
pre-rendered with [`brand figure`](https://github.com/joeseverino/branding-engine).</sup>

## The Pieces

### The vault

Markdown with typed YAML frontmatter: a stable `doc_id`, a sensitivity tier
(`public`, `internal`, `sensitive`, `restricted`), and relationships to
projects and assets. Plain files are the one format every consumer can read,
and the frontmatter is the data model. A sanitized, runnable model of its
structure ships as the MCP's
[sample vault](https://github.com/joeseverino/severino-vault-mcp/tree/main/examples/sample-vault).

### [vault-engine](https://github.com/joeseverino/vault-engine) and the MCP servers

vault-engine is the domain-agnostic core, published on PyPI: schema profiles,
a frontmatter index, ranked search, a task ledger, and atomic writes.
[severino-vault-mcp](https://github.com/joeseverino/severino-vault-mcp)
serves my operations vault through it, and
[severino-edu-mcp](https://github.com/joeseverino/severino-edu-mcp) serves my
coursework vault through the same engine with a different profile. A second
vault is a second profile, not a fork.

The MCP exists to stop one failure: an assistant writing a generic tutorial
when I already have a four-line runbook. It runs over stdio from local files
only. The sensitivity tier decides what it releases, in code rather than in a
prompt, and `restricted` bodies need an explicit local unlock. The same
functions back its CLI, so a script and an AI session validate and write
through one path.

### [Severino HQ](https://github.com/joeseverino/severino-hq)

A self-hosted operations hub, reachable only over Tailscale, with passkey-first
sign-in through a self-hosted Pocket ID. A controller holds scoped Cloudflare
and Tailscale credentials and reads what each can see; HQ joins those readings
into machines, services, and domains, and derives the rest. Nothing about the
infrastructure is authored by hand, so there is no second copy to drift.

Every change, whether from the web UI, the API, MCP, or the CLI, goes through
one gated path, and a write needs a connection that declares it manages that
record. Private domains plug in as separately released, signed extensions
against a public contract, which is why the host can be public. Code reaches
it only through CI, a scanned image, and a self-hosted runner that dials out,
so nothing inbound is ever opened.

### [jseverino.com](https://github.com/joeseverino/jseverino.com)

The public subset of the vault. A sync turns published documents into MDX with
clean image masters and opens a pull request; Cloudflare Pages builds it with
Astro and serves it with a per-request CSP nonce. A D1 contact form behind
Turnstile is the only dynamic surface. Previews answer only to a Cloudflare
Access service token, and a tagged release ships the build with an SBOM and
Sigstore attestations anyone can verify with `gh attestation verify`.

### [tools](https://github.com/joeseverino/tools) and [cordon](https://github.com/joeseverino/cordon)

`tools` is my CLI suite. Any operation that takes several commands copied out
of a note becomes one command. Each tool is declared once, and its help, shell
completions, docs, and machine-readable spec render from that declaration, so
none of them can disagree.

cordon is that declaration format as a standalone, language-agnostic contract,
with a JSON Schema at
[`jseverino.com/schemas/cordon-v4.json`](https://jseverino.com/schemas/cordon-v4.json)
and conformance fixtures. Its distinguishing field is `effect`: every command
states its blast radius, from `read` to `deploy`, so an agent can tell a
production deploy from a lookup before it runs anything.

### [sitedrift](https://github.com/joeseverino/sitedrift) and [branding-engine](https://github.com/joeseverino/branding-engine)

Two published npm packages the site depends on. sitedrift reviews a change
against unchanged production on the same route (split, overlay, diff, SEO) and
installs itself on branch previews. branding-engine renders every brand
surface and every diagram on these pages from one brand definition.

## The Network Underneath

Nothing private listens on the public internet. Every host is on a Tailscale
tailnet with tailnet lock, so a new device joins only when a signing device
co-signs it. SSH logins use 12-hour certificates signed by a CA whose key
stays in 1Password and signs through Touch ID; servers prove themselves with
host certificates in turn. AdGuard Home answers DNS for every tailnet device,
internal services carry certificates from a root CA on an offline VM
([cert-generator](https://github.com/joeseverino/cert-generator)), and
monitoring runs on a separate VPS so it survives a homelab outage. The public
surface is the site on Cloudflare's edge.

## Design Principles

1. **One owner per fact.** The vault owns knowledge, HQ owns infrastructure,
   and neither keeps a copy of the other's facts.
2. **Policy lives in code, not prompts.** Sensitivity, publishing, and
   blast-radius gates are enforced by tools, whatever an AI session is told.
3. **Private by default, public by explicit gate.** `published: true` is a
   decision, not a default.
4. **Repeated operations become one command.**
5. **Every seam between two systems gets a check.** A shared schema, a
   contract with conformance fixtures, or a drift guard, so a mismatch fails
   loudly instead of silently.
6. **Verifiable over trusted.** Signed commits and tags, pinned dependencies,
   and attested releases, so a claim can be checked rather than believed.
