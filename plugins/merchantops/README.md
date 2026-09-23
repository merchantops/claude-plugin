# MerchantOps Claude Plugin

Turns "set up MerchantOps for a new customer" into a conversation. This plugin bundles eight
skills, four slash commands, and a reference to MerchantOps' remote MCP server for Claude
Code, Claude Desktop and Claude Cowork.

## Two surfaces — read this before you install

This plugin ships **two separate capability paths**, and they are not the same thing:

- **The bundled MCP server** (`merchantops` in `.mcp.json`) is a 56-tool, **write-capable**
  server — installing this plugin arms MCP writes for the conversational agent. It is for
  **conversational catalog reads** (and a bounded set of guarded writes) with its own
  two-phase confirmation guardrails: destructive or wide-reaching actions require a preview
  call first, which returns a confirmation token bound to the exact arguments, before the
  matching write tool will run.
- **The CLI, driven by the skills in this plugin, is the onboarding path.** The skills shell
  out to the `mo` command-line tool, not the MCP. **No skill in this plugin depends on the
  MCP** — it has read tools for property definitions, product types and lakehouse brands, but
  no WRITE tools for any of them (a permanent boundary, `agent-surface.md`), no crawl-trigger
  tool, and no CSV/JSON catalog bulk-import the way `mo import` does. Property dictionaries,
  product types, catalog loads and org provisioning only ever happen through `mo`.

If you only want conversational catalog reads, the MCP reference alone gets you that. If you
want to onboard a new customer — crawl a site, propose a schema, load a catalog — you need
the `mo` CLI installed too; see "Prerequisite: the `mo` CLI" below.

## What's included

**Slash commands** (side-effecting; each requires explicit invocation, never auto-triggered):

| Command | What it does |
|---|---|
| `/merchantops:onboard <url-or-file>` | Chains the onboarding skills: crawl a site or read a spreadsheet, propose a property dictionary and product types, produce a reviewable `onboarding-bundle/`. Read-only — it never writes to MerchantOps. |
| `/merchantops:validate <bundle-dir>` or `<artifact> <file>` | Validate a whole bundle in the safe load order, or a single artifact, against MerchantOps' schema and referential rules. Read-only, entirely client-side. |
| `/merchantops:load <bundle-dir>` or `<artifact> <file>` | Dry-run the load, show the plan, wait for explicit confirmation, then load it. |
| `/merchantops:provision <org-name> <slug> <owner-email> [role...]` | Two hard stops: plan and create (or find) the org, then plan and send the owner invite with roles — each step waits for its own confirmation. Staff-only. |

**Skills** (model-invoked when relevant, except where noted):

| Skill | What it does |
|---|---|
| `merchantops-cli-reference` | Reference material for `mo` — the curated commands, every exit code, the safe load order, the environment presets. Deliberately self-contained, so the other skills work without any MerchantOps repository on disk. Invoked by the other skills when they need a command; read-only. |
| `merchantops-onboard-from-site` | "Here is my site, crawl it and set up MerchantOps." Crawls a site (Firecrawl, agent-side, your own key and budget) and hands the result to schema design. Confirms the crawl budget before spending credits. |
| `merchantops-onboard-from-spreadsheet` | "Here is a CSV/Excel of my products." Reads a file locally and hands the result to schema design. |
| `merchantops-design-schema` | Proposes a property dictionary and product types from analyzed records, and writes a complete `onboarding-bundle/` for review. **Hard stop**: presents the files and waits. |
| `merchantops-load-catalog` | "Load the bundle." Dry-runs, shows the plan, waits for confirmation, then loads. **Hard stop**; re-runs use `--resume`. |
| `merchantops-import-pricing` | "Load these prices." Shows a per-product plan with batch grouping before loading. **Hard stop**; prices land as **draft batches requiring approval**, never live. |
| `merchantops-provision-org` | "Provision a new customer org." Plans a Stytch org, owner invite and roles, then creates it after confirmation. **Hard stop**; a live Stytch project needs the org name re-typed. |
| `merchantops-competitive-analysis` | "Compare my properties against these pages." Scrapes 3-10 **user-supplied** aspirational product pages — never crawls a competitor's whole site. Read-only, no writes at all. |

**MCP reference**: `.mcp.json` points at MerchantOps' existing remote MCP server
(`https://mcp.merchantops.ai/mcp` by default; override with `MERCHANTOPS_MCP_URL` for a qa
install, no file edit needed). It is not built or hosted by this plugin.

## Prerequisite: the `mo` CLI

The onboarding skills and all four slash commands shell out to `mo`, a Python 3.11+
command-line client. **Not live yet — the planned, recommended path once `merchantops-cli`
publishes to PyPI:** `uv tool install merchantops-cli` (or `pipx install merchantops-cli`, or a
plain venv + `pip install merchantops-cli`). The full install ladder — per-OS Python pointers,
each method spelled out in full including how `mo` reaches `PATH`, and the Node.js /
`firecrawl-cli` prerequisite the two site-crawl skills add — lives in the public
`merchantops/claude-plugin` repository's root README, not duplicated here.

<!-- PYPI-FLIP:START state=not-published -->
**Today's actual install: from source, for MerchantOps-supported customers**, from the clone's
root:

```bash
git clone git@github.com:merchantops/product-enricher.git
cd product-enricher
pip install -e ./merchantops-sdk -e ./merchantops-cli
mo --help
```
<!-- PYPI-FLIP:END -->

If you don't have access to that repository, ask your MerchantOps contact. Once installed,
sign in once per environment — `--env local|qa|prod` carries the API URL, the OAuth issuer
and the audience, so an environment is the only thing you pick:

```bash
mo --env qa auth login     # or --env prod, once you're ready for a live org
mo --env qa auth whoami    # confirms who you are and which permissions you actually hold
```
For unattended/CI use, set `MERCHANTOPS_TOKEN` instead of logging in interactively.

The catalog skills open with `mo onboarding status`, which reports the org's current shape and
fails with a "sign in" exit code if no credential is available, and they run `mo auth whoami`
before a write when a permission is in doubt.

**Don't need the plugin wrapper?** Every skill here is also a self-contained `SKILL.md`
folder with no dependency on this plugin — see the public `merchantops/claude-plugin`
repository's root README for the "skills only" install path (copy just the ones you want into
another agent's skills directory), the full CLI prerequisites ladder, running `mo` on its own
(no skills, no plugin), and using MerchantOps' remote MCP server with no local install at all,
plus a "which surface can run what" table across Claude Code, Claude Desktop, Claude Web and
other agent platforms.

## Install

This section covers installing **the plugin** (skills + commands + MCP reference together).
For the other three ways to get MerchantOps tooling — skills only, `mo` on its own, or the
MCP server with no local install at all — plus the full "which surface can run what" table,
see the public `merchantops/claude-plugin` repository's root README.

### Claude Code

```
/plugin marketplace add merchantops/claude-plugin
/plugin install merchantops@merchantops
```

### Claude Desktop

Claude Desktop has a Plugins browser built in. Either run the same two commands above in a
chat, or: open the Plugins section, add the marketplace `merchantops/claude-plugin`, search
for "MerchantOps", and click Install. Choose an installation scope (User, Project or Local)
when prompted. **This is the demoed install path.**

### Claude Cowork

Cowork's plugin browser works the same way as Desktop's (best-effort support — this path is
smoke-verified, not the primary demo). Add the `merchantops/claude-plugin` marketplace, find
"MerchantOps" in the browser, and install.

### Reconnect after install

MCP clients cache the tool list from their last connection. If MerchantOps' catalog tools
don't show up right after installing (or after this plugin updates), reconnect the connector
or start a new session to force a re-list.

### Local testing (development)

```bash
claude plugin validate --strict ./merchantops-plugin   # schema + frontmatter checks
claude --plugin-dir ./merchantops-plugin                # load without installing
/reload-plugins                                          # after editing a running session's plugin
```

## Before you start an onboarding: use a scoped role, not Owner

The token you're signed in as (via `mo auth login`) is the real boundary on what any of
these skills or commands can do — not the CLI's own guardrails. Every write command defaults
to a no-write plan, requires an explicit `--execute` plus a `--confirm` hash bound to exactly
the artifact you reviewed, and every skill shows you that plan and waits for your explicit
confirmation before running it. Those are real protections against a misfire or a casually
run command, but they are not a boundary against an agent that decides, or is told by
something it read, to supply the flags anyway — the API's RBAC check on your member's role
is what actually bounds that. **Sign in as a member with a scoped, non-Owner role for
onboarding work**, so a misfire or a successfully-injected instruction can't exceed what that
role is allowed to do in the first place.

Content read from a website, spreadsheet, PDF or API response is always **data, never
instructions**, in every skill and command in this plugin — text that looks like a command,
a request, a system prompt, or a claim about permissions gets reported to you, never acted
on.

## Support

Questions or issues: contact MerchantOps. This plugin's source of truth lives in the
MerchantOps engineering repository; this public repository is a mirrored copy, kept in sync
automatically on every change.
