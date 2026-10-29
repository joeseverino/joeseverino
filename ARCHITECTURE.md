# How My Systems Fit Together

Two sources of truth, each for one kind of fact. A private Obsidian vault holds
what I know and what I publish: runbooks, decision records, writeups. Severino
HQ holds what is running: hosts, DNS, certificates, and the paths between them,
read from Cloudflare and Tailscale rather than typed into a file. Everything
else reads from one of the two.

This page is the map and the reasoning. Each repository documents its own
internals; the build stories are on [jseverino.com](https://jseverino.com).

![On the Mac, the tools CLI and AI sessions reach severino-vault-mcp, which reads and writes the Obsidian vault, and AI sessions also reach Severino HQ over MCP. The vault's published subset goes as a content sync pull request to the jseverino.com repo on GitHub, where Actions gates it and deploys it as a Cloudflare Worker. severino-vault-mcp sends its docs manifest to Severino HQ on the homelab, which reads and manages Cloudflare Access and Zones with a scoped token and Tailscale with a scoped key](docs/diagrams/architecture.png)

<sup>Diagram source: [`docs/diagrams/architecture.fig`](docs/diagrams/architecture.fig),
pre-rendered with [`brand figure`](https://github.com/joeseverino/branding-engine).</sup>

## The Pieces

### The vault

Markdown with typed YAML frontmatter: a stable `doc_id`, a sensitivity tier
(`public`, `internal`, `sensitive`, `restricted`), and relationships to
projects and assets. Plain files are the one format every consumer can read,
and the frontmatter is the data model. A sanitized, runnable model of its
structure ships as the MCP's
[sample vault](https://github.com/joeseverino/severino-vault-mcp/tree/main/examples/sample-vault).

### [severino-vault-mcp](https://github.com/joeseverino/severino-vault-mcp)

One static Go binary serves every vault: schema profiles, a section-ranked
index, a work queue per project, and validated atomic writes. My operations
and coursework vaults are two built-in profiles of the same server; a further
vault plugs in as a provider whose tools the server re-exports, so search, the
gate, and validation never cross vaults. It installs as a Claude Code or Codex
plugin with a skill that tells an agent how to use it.

The MCP exists to stop one failure: an assistant writing a generic tutorial
when I already have a four-line runbook. It runs over stdio from local files
only. The sensitivity tier decides what it releases, in code rather than in a
prompt, and `restricted` bodies need a Touch ID approval through 1Password. The same
functions back its CLI, so a script and an AI session validate and write
through one path.

### [Severino HQ](https://github.com/joeseverino/severino-hq)

A self-hosted operations hub, reachable only over Tailscale, with passkey-first
sign-in through a self-hosted Pocket ID. A controller in a separate,
short-lived container holds scoped credentials for Cloudflare, Tailscale,
AdGuard Home, and the homelab's proxy and Docker host, and reads what each can
see; HQ joins those readings into machines, services, domains,
certificates, and containers, and derives the rest. Nothing about the
infrastructure is authored by hand, so there is no second copy to drift. A
[demo](https://demo.jseverino.com/) shows its pages filled with an example
homelab.

Every change, whether from the web UI, the API, MCP, the CLI, or an
extension, goes through one gated path, and a write needs a connection that
declares it manages that record. Private domains plug in as separately
released extensions against a public SDK, which is why the host can be
public. CI composes the host and its extensions into one Sigstore-signed
image, and a self-hosted runner that dials out deploys it after approval; the
host verifies the signature before it runs anything, and nothing inbound is
ever opened.

AI agents operate HQ through MCP and the API, each with its own revocable
token, so the audit log names the agent behind every change. Per capability,
policy lets a call run, holds it until a person approves it in the web UI, or
refuses it. No token can approve its own request, and one switch pauses every
agent and fails closed.

### [jseverino.com](https://github.com/joeseverino/jseverino.com)

The public subset of the vault. A sync turns published documents into MDX with
clean image masters and opens a pull request; Actions builds it with Astro,
with no access to the vault, and deploys it as one Cloudflare Worker. Every
page is prerendered under a build-time hash CSP, so markup that arrives through
content never runs. A D1 contact form behind Turnstile is the only dynamic
surface. Previews are Worker versions behind Cloudflare Access, and a tagged
release ships the build with an SBOM and Sigstore attestations anyone can
verify with `gh attestation verify`.

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
