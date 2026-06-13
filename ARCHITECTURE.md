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

![The full system map: on the local Mac, AI sessions and the tools CLI drive severino-vault-mcp, which reads and writes the Severino Labs vault and syncs a docs manifest plus the shared schema to Severino HQ on the private tailnet; the vault's published subset is snapshotted into the jseverino.com Astro repo alongside branding-engine assets; a git push triggers the Cloudflare Pages build serving jseverino.com with D1 behind it, reviewed by sitedrift against live; the tools CLI and the MCP both conform to the cordon command-surface contract, one shared spec](diagrams/architecture.png)

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

A fully sanitized, runnable model of this structure ships in the MCP's
[sample vault](https://github.com/joeseverino/severino-vault-mcp/tree/main/examples/sample-vault):
the same top-level shape — `01 Projects` / `02 Infrastructure` / `03 Runbooks`
for indexed operational docs, plus templates, reference, the `05 Writeups` /
`06 Pages` publishing pipeline, and an archive — with the frontmatter data model,
the Quick Index navigation hub, and the sensitivity tiers, every host and command
replaced by `*.example` placeholders. It is a concrete look at how the vault is
organized (a typed knowledge base, not loose notes) without exposing the real
one, and the MCP's own test suite runs against it.

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
- **Two faces, one code path.** The same FastMCP-free service functions are
  exposed as MCP tools for AI sessions *and* as plain CLI subcommands for
  scripts, returning one JSON result shape that the `site` CLI and its terminal
  UI both parse. When the `site` CLI validates a writeup it runs the exact
  functions an AI session would, so validation, writes, schema, and atomic file
  replacement all live once and cannot drift between the interactive and
  scripted paths.
- **Reaches the edge, read-only, same gate.** Operational tools list
  contact-form submissions and CSP violation reports from Cloudflare D1 via
  wrangler, so triage happens in the same session as everything else. Contact
  PII is redacted by default — names abbreviated, emails masked, message
  previewed — and full release takes the same explicit, audited unlock as a
  `restricted` doc body. The sensitivity gate covers operator PII, not just the
  vault.

### [Severino HQ](https://github.com/joeseverino/severino-hq) — the private ops app

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

The frontmatter contract HQ validates against isn't HQ's own copy — it's the
MCP's schema, emitted as JSON and committed into HQ, so the importer can never
reject a value the MCP just wrote. One definition, enforced on both sides of
the sync.

Deployment is a gated pipeline, not an SSH session. A push to `main` (via git
or `hq ship`) runs the checks in GitHub Actions — lint, tests on two Python
versions, a `check --deploy` posture gate, and pip-audit — then builds a
container image, scans it with Trivy, and **only on green** does a self-hosted
runner on the homelab pull the scanned image from GHCR and restart the
container. The runner dials out to GitHub; nothing inbound is ever opened,
there is no SSH in the deploy path at all, and a red commit physically cannot
reach the box.

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

Each tool emits its whole command surface as one JSON spec — `-h`, the machine
`--describe`, an interactive explorer, and even the README command reference and
shell completions all derive from it (the same emitter that answers `-h` writes
the docs, so they can't drift), with per-command *effect* metadata (read →
deploy blast-radius) so an AI agent can risk-gate before acting. `tools describe
--repos` federates the MCP into the same contract, so one document describes the
surface across repos. That contract is now its own language-agnostic spec —
**cordon** (below).

![tools describe --tui: a full-screen two-pane explorer listing all 17 tools and 72 commands; the right pane shows the selected vault command with its effect classed remote_write over the network and a copy-ready vault sync invocation, all rendered from each tool's describe_spec declaration](docs/images/tools-describe-tui.png)

*`tools tui`: 72 commands across 17 tools, rendered from the one
`describe_spec()` each tool declares — down to the per-command effect chip
(here `remote_write · network`), the same signal an AI agent reads to risk-gate
before running.*

### [cordon](https://github.com/joeseverino/cordon) — the shared contract

The command surface above isn't a `tools`-only convention — it's a standalone,
language-agnostic spec. **cordon** defines one JSON contract for describing a
command-line tool: declare it once, render every view (help, completions, docs,
the agent-readable spec), and have every command carry its *effect* — a fixed
blast-radius ladder (`read → local_write → vault_write → remote_write →
deploy`) plus `network` / `interactive` tags — so an automated agent can
risk-gate before it acts.

Two independent emitters, in two languages, already conform to it: the `tools`
CLI (Bash, rendered from a `describe_spec` DSL) and `severino-vault-mcp` (Python,
by introspecting its argparse parser). One JSON Schema validates both — hosted at
its own `$id`,
[`jseverino.com/schemas/cordon-v4.json`](https://jseverino.com/schemas/cordon-v4.json),
and shipped with language-agnostic conformance fixtures so a third
implementation in any language is just "pass the fixtures."

**Why:** most "describe your CLI" formats answer *what flags exist*; none answer
*what happens if I run this*. Making `effect` a required field — and the one
signal both a runtime gate and an AI session stop on — is what lets me hand an
agent the whole toolchain without it mistaking a production deploy for a read.

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

**An HQ code change.** `hq ship -m "fix dashboard"` runs local checks,
commits, and pushes. The push triggers the gated GitHub Actions pipeline —
lint, tests, deploy-posture check, pip-audit, image build, Trivy scan — and
only when every gate is green does the homelab's self-hosted runner pull the
scanned image and restart the container. No step is manual, and a red commit
can't ship.

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
- **Schema single-source.** The core frontmatter enums (doc types,
  environments, sensitivities, ID prefixes) are defined once in the MCP and
  emitted as JSON. Severino HQ validates its importer against that exact output,
  and the vault's own schema doc is checked against it too, so the MCP can never
  write a value HQ would reject. `hq schema --check` fails on any drift across
  all three.
- **Install drift.** The MCP ships a `--fingerprint` flag hashing its
  installed sources; doctor compares it against the source repo so a stale
  install can't silently disagree with the code.
- **Seam drift.** Doctor commands verify every CLI-to-npm-script seam
  resolves, plus security headers, contrast ratios, and dependency audits.
- **Command-surface drift.** Each tool's human help and its machine-readable
  JSON render from one spec, so they can't diverge; round-trip, bash/zsh
  byte-parity, spec↔dispatch, and effect-enum guards enforce it in CI. The
  contract itself is the **cordon** spec, and both the Bash and Python emitters
  validate against its one schema (`tools check` runs the federated `describe
  --repos` through it), so two languages can't drift on what the contract means.
- **Supply chain.** The site repo runs SHA-pinned GitHub Actions: CodeQL,
  dependency review, SBOM generation, OpenSSF Scorecard, link checking, and
  scheduled Lighthouse runs, keeping the code-scanning dashboard at zero
  open alerts. Code scanning (CodeQL), dependency updates (Dependabot), and
  dependency audits now run across the Python and tooling repos too, and every
  repo's `main` is branch-protected — merges need a green CI run and resolved
  review threads.

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
5. **Every seam between two systems gets a check.** Parity scripts, a
   single-source schema with `--check` guards, doctor commands, and fingerprints
   make integration drift loud instead of silent.
6. **AI is grounded or it is silent.** Assistants answer from real docs
   through the MCP, or say the doc doesn't exist — never from generic memory.

---

The build stories behind these systems, with the screenshots, configs, and
step-by-step decisions, are published at
[jseverino.com](https://jseverino.com).
