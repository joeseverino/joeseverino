# How My Systems Fit Together

Everything I run hangs off one idea: **a private Obsidian vault is the single
source of truth**, and every other system (AI tooling, a private ops app, the
public site, the CLI suite) derives from it rather than duplicating it.

> The vault holds the knowledge; an MCP server lets AI tools recall and edit it
> safely; a private Django app turns it into structured operational records;
> an Astro site publishes the public subset; a CLI toolchain ties the loop
> together with repeatable, checked commands.

This document is the map: every component, how they talk, and, most
importantly, the whys. Each component links to its public repo; the deeper
build stories live on [jseverino.com](https://jseverino.com).

## The Map

![The full system map: on the local Mac, AI sessions and the tools CLI drive severino-vault-mcp, which reads and writes the Severino Labs vault; the vault syncs a docs manifest to Severino HQ on the private tailnet and publishes the public subset into the jseverino.com Astro repo alongside branding-engine assets; a git push triggers the Cloudflare Pages build serving jseverino.com with D1 behind it, reviewed by sitedrift against live](diagrams/architecture.png)

<sup>Diagram source: [`diagrams/architecture.mmd`](diagrams/architecture.mmd),
pre-rendered with `diagrams/render.sh` so every browser sees the same
pixels.</sup>

## The Pieces

### Severino Labs vault — the source of truth

An Obsidian vault of runbooks, infrastructure notes, decision records, and
write-ups. Every doc carries YAML frontmatter with a stable `doc_id`, a
sensitivity tier (`public` / `internal` / `sensitive` / `restricted`), a
`system`, and typed relationships to projects and assets. The vault itself is
a Git repository mirrored to a private GitHub remote for offsite backup, with
the infrastructure and runbook directories encrypted via git-crypt; even the
private mirror stores those bodies as ciphertext.

**Why:** plain markdown plus frontmatter is the one format every consumer
below can read: Obsidian renders it, the MCP indexes it, HQ imports it, the
site publishes it. The frontmatter isn't decoration; it's the data model.
Nothing else is allowed to become a second source of truth.

### [severino-vault-mcp](https://github.com/joeseverino/severino-vault-mcp) — the recall layer

A local stdio MCP server. No network port; it runs as a child process of the
AI client. It exists to kill one failure mode: an AI assistant generating a
generic tutorial when I have my own four-line runbook.

- **Sensitivity gate in code, not prompts.** The doc's frontmatter tier
  decides what the MCP releases. `restricted` bodies are withheld unless an
  explicit local unlock (env flag + configured hash + interactive prompt on
  the Mac) approves the release. An AI session cannot talk its way past it.
- **Designed for small local models too.** A one-call `get_runbook` tool does
  search + read together, because two-step tool choreography gave smaller
  models (LM Studio and friends) a chance to corrupt the doc ID between
  search and read. That came out of a real observed failure, not theory.
- **Two faces, one code path.** The same Python functions are exposed as MCP
  tools for AI sessions *and* as plain CLI subcommands for scripts. When the
  `site` CLI validates a writeup, it runs the exact functions an AI session
  would, so validation logic cannot drift between the interactive and
  scripted paths.
- **Reaches the edge, read-only.** Operational tools list contact-form
  submissions and CSP violation reports from Cloudflare D1 via wrangler, so
  triage happens in the same session as everything else.

### Severino HQ — the private ops app

A Django 5 app (SQLite + gunicorn) running as a Docker Engine container on the
homelab server, with named volumes for data, media, and exports. Reachable
only over the LAN and the WireGuard tailnet — never the public internet.
Sign-in is OIDC SSO against a self-hosted Pocket ID instance, with the Django
password form kept as the break-glass path.

HQ tracks projects, assets, expenses, receipts, and an index of every vault
doc. The split is deliberate: **the vault keeps the prose, HQ keeps the
structure.** `hq sync` regenerates HQ's records from vault frontmatter:
metadata, relationships, and pointers only. Runbook bodies and secrets never
enter HQ, so its database is never the interesting target.

Deployment is one command: `hq ship` runs local checks, commits, pushes, has
the server pull over a **read-only deploy key** (the server can never push),
rebuilds the container, runs a remote Django check, and re-syncs the docs
index. My primary SSH keys never exist on the server.

### [jseverino.com](https://github.com/joeseverino/jseverino.com) — the public subset

An Astro static site on Cloudflare Pages. The vault doubles as its private
CMS with a strict one-way flow, **vault → repo → edge**:

1. Write in Obsidian. Drafts are invisible by default; `published: true` is
   an explicit gate.
2. `site publish` snapshots the public subset into the repo: strips
   vault-only metadata, optimizes images locally with Sharp (AVIF/WebP), and
   records intrinsic dimensions in a manifest so every image ships with
   explicit width/height. Zero layout shift, by construction.
3. Cloudflare Pages builds from the committed snapshot; the edge serves
   static files with strict headers.

The only dynamic surface is deliberate: Pages Functions provide a per-request
CSP nonce middleware and a D1-backed, Turnstile-protected contact form. HTML
is intentionally not edge-cached because the nonce changes per request.
Security over cache, chosen knowingly.

### [tools](https://github.com/joeseverino/tools) — the control plane

A personal CLI suite (`site`, `hq`, `vault`, `inbox`, `encrypt`, `backup`, …)
sharing one scaffold, help convention, and bats test suite. The rule: any
operation that means copying several commands out of a note becomes one
command here. The CLIs are the deterministic control plane; AI is optional
everywhere, automation is not. Even the README is kept honest: every measured
claim in it has a benchmark script that asserts it in CI.

### [branding-engine](https://github.com/joeseverino/branding-engine) — one brand source

A published [npm package](https://www.npmjs.com/package/branding-engine) that
renders favicons, vector marks, wordmarks, social cards, brand sheets, and web
tokens from one color and monogram, with real font-outline rendering rather
than text in an SVG. The brand is 100% config; the engine is generic. One
source edit regenerates every brand surface, and the generated output
directory is fully disposable: delete it, rebuild, get byte-identical assets
back.

### [sitedrift](https://github.com/joeseverino/sitedrift) — the review layer

A published [npm package](https://www.npmjs.com/package/sitedrift) for
reviewing DEV against LIVE on the same route: split, overlay/diff, synced
navigation, response deltas, SEO checks. Every site and brand change gets
compared against unchanged production before it ships. On branch previews it
installs a review shell automatically; on `main` it provably exits without
touching the production build.

## How a Change Moves Through the System

**An operational question.** I ask an AI session "how do I renew the homelab
cert?" It calls the MCP, which ranks vault docs and returns my actual
runbook. The answer is the runbook's own commands, not a generated openssl
tutorial. If the doc is `restricted`, the body never leaves the machine
without an explicit local unlock.

**A write-up, draft to production.** `site new-writeup` scaffolds a vault
folder; I write in Obsidian; a terminal UI (`site manage`) edits frontmatter
through the MCP's code path so every write is validated. Publishing runs a
gate that refuses to ship on missing images, unknown technology tags, or
unresolved references, then snapshots, optimizes, commits, and lets
Cloudflare Pages build. `hq sync` mirrors the new piece into HQ's content
records. One source file ended up in four systems, and no step was manual
bookkeeping.

![site manage detail view: frontmatter fields of one writeup edited in place, with read-only relation fields deferring to Obsidian](https://raw.githubusercontent.com/joeseverino/jseverino.com/main/docs/images/site-cli/manage-writeup-detail.png)

*The `site manage` TUI editing one writeup's frontmatter: every save goes
through the MCP's validated code path, never raw YAML edits.*

**An HQ code change.** `hq ship -m "fix dashboard"`: checks, commit, push,
server pulls read-only, container rebuilds, remote Django check, docs
re-sync. One command, and the failure of any step stops the rest.

**A brand change.** Edit one value in the brand config, regenerate, and
review the diff with sitedrift against live production. The comparison
isolates exactly the pixels that were supposed to change (favicon, wordmark,
palette, social cards) before anything merges.

![sitedrift Overlay/Diff comparing a red-brand Cloudflare preview against live production: only the changed brand pixels are lit, everything identical stays black](https://raw.githubusercontent.com/joeseverino/jseverino.com/main/docs/images/sitedrift-brand-demo/red-vs-live-diff.png)

*A full red-brand test deployment diffed against unchanged production:
lit = changed, black = identical. The only lit pixels are the brand.*

## Keeping It Honest

Integration drift is the failure mode of multi-system setups, so every seam
gets a check instead of a convention:

- **Schema parity.** The writeup frontmatter contract is asserted across five
  surfaces (vault schema doc, site Zod schema, MCP tool signature, MCP CLI
  flags, and the TUI editor) by a parity script that fails CI on any
  mismatch.
- **Install drift.** The MCP ships a `--fingerprint` flag hashing its
  installed sources; doctor compares it against the source repo so a stale
  install can't silently disagree with the code.
- **Seam drift.** Doctor commands verify every CLI-to-npm-script seam
  resolves, plus security headers, contrast ratios, and dependency audits.
- **Supply chain.** The site repo runs SHA-pinned GitHub Actions: CodeQL,
  dependency review, SBOM generation, OpenSSF Scorecard, link checking, and
  scheduled Lighthouse runs, keeping the code-scanning dashboard at zero
  open alerts.

## The Network Underneath

Nothing private listens on the public internet. Everything above rides this
layer:

![The infrastructure layer: a Tailscale tailnet with tailnet lock connects admin devices, a homelab host acting as subnet router and residential exit node, and a cloud VPS acting as a datacenter exit node; the homelab VM runs AdGuard Home, Nginx Proxy Manager fronting HQ, AdGuard, and Portainer with TLS, Pocket ID signing into HQ and Portainer, and Severino HQ in Docker Engine; the VPS runs Caddy fronting Uptime Kuma plus a Portainer agent; AdGuard forwards upstream over encrypted Cloudflare DoH; an offline private root CA VM issues internal TLS certificates](diagrams/network-layer.png)

The tailnet is a WireGuard mesh with tailnet lock: a new node can't join
without being co-signed by a designated signing device, so admin-console
compromise alone is not enough. The **homelab host** doubles as a subnet
router (any tailnet device reaches LAN-only services from anywhere) and a
residential exit node; the **cloud VPS** is a second exit node with
datacenter egress. The host's Docker Engine VM runs the private core.
AdGuard Home answers every tailnet device's DNS over the tunnel and forwards
upstream over encrypted DoH, and the DNS rewrite *is* the routing decision. Nginx Proxy Manager is the TLS front door
for the VM's web UIs (HQ, AdGuard, Portainer), serving certificates issued by
the offline root CA. Pocket ID provides OIDC sign-in to both HQ and
Portainer, and Portainer manages the VPS's containers through its agent.

Monitoring deliberately runs *outside* the homelab: Uptime Kuma on the VPS,
fronted by Caddy, watches everything over the tailnet, so it catches the
network-level failures an internal monitor would go down with. The only
public surface in the entire picture is the static site on Cloudflare's
edge.

## Design Principles

1. **One source of truth.** Every system derives from the vault; none of them
   re-author what it owns.
2. **Policy lives in code and metadata, not prompts.** The sensitivity gate
   and publish gate are enforced by tools, so they hold regardless of what an
   AI session is told.
3. **Private by default, public by explicit gate.** Drafts, metadata, and
   operational detail stay home; `published: true` is a decision, not a
   default.
4. **Repeated operations become one command.** If a task needs a runbook's
   worth of shell commands, it gets folded into the CLI suite.
5. **Every seam between two systems gets a check.** Parity scripts, doctor
   commands, and fingerprints make integration drift loud instead of silent.
6. **AI is grounded or it is silent.** Assistants answer from real docs
   through the MCP, or say the doc doesn't exist — never from generic memory.

---

The build stories behind these systems, with the screenshots, configs, and
step-by-step decisions, are published at
[jseverino.com](https://jseverino.com).
