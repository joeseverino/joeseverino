# Joe Severino

Cybersecurity graduate student at Georgia Tech focused on network security, security engineering, and the production tooling that keeps secure systems maintainable.

**Certifications:** CCNA, CompTIA Security+, ISC2 Certified in Cybersecurity (CC)

Most of my projects are built around real systems I run myself: WordPress hardening, TLS automation, private PKI, DNS filtering, local AI tooling with safety boundaries, and homelab infrastructure.

## Featured Projects

- **[severino-vault-mcp](https://github.com/joeseverino/severino-vault-mcp)** - Local-first MCP server that gives AI assistants safe access to an Obsidian operations vault. Layered CI security tooling (CodeQL, pip-audit, OSSF Scorecard, Dependabot), documented threat model, and a four-tier sensitivity gate for credential-adjacent content.
- **[branding-engine](https://github.com/joeseverino/branding-engine)** - Published [npm package](https://www.npmjs.com/package/branding-engine) that generates a full brand kit — favicons, marks, wordmarks, social cards, and CSS tokens — from one compact mark and a single accent color. Extracted from a one-off script in [jseverino.com](https://jseverino.com), which now depends on it to drive its own marks, favicon, and theme color. Node-only core with real font-outline rendering and a CI release pipeline.
- **[jseverino.com](https://github.com/joeseverino/jseverino.com)** - Public Astro portfolio deployed on Cloudflare Pages from a private Obsidian vault. Includes vault-to-content sync, static publishing checks, RSS/sitemap generation, Cloudflare security headers, and a D1-backed contact form protected by Turnstile.
- **[severino-labs-security-layer](https://github.com/joeseverino/severino-labs-security-layer)** - Custom WordPress security plugin for application hardening, file integrity monitoring, security event logging, browser security headers, and passkey-first login customization.
- **[tools](https://github.com/joeseverino/tools)** - Personal macOS CLI suite: age-based file encryption with Keychain-cached unlock, vault sync, dotfile backup, DNS latency diagnostics, and a bridge between an Obsidian vault and a private Django docs index.
- **[zsh-git-deploy-workflow](https://github.com/joeseverino/zsh-git-deploy-workflow)** - Zero-dependency Git deploy pipeline for solo devs. One bootstrap command wires SSH keys, GitHub integration, and per-project shell commands. `acmepush "fix: message"` stages, commits, pushes, and deploys without CI, YAML, or dashboards.
- **[tls-cert-renewal-automation](https://github.com/joeseverino/tls-cert-renewal-automation)** - Automates Let's Encrypt wildcard certificate renewal and cPanel deployment using Certbot, Cloudflare DNS validation, and cPanel UAPI.

## Focus Areas

- Network security
- Infrastructure automation
- Static site publishing
- WordPress hardening
- TLS and PKI
- DNS filtering
- Local-first AI tooling with explicit safety boundaries
- Secure deployment workflows
- Homelab engineering

## Links

- Portfolio: https://jseverino.com
- LinkedIn: https://www.linkedin.com/in/joeseverino
