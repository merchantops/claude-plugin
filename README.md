# MerchantOps — Claude Plugin & Skills

This repository is the public distribution point for MerchantOps' Claude plugin and its
underlying skills. It is a **mirrored copy** — the source of truth is MerchantOps' private
engineering repository, kept in sync automatically on every change.

- **The plugin itself** lives at [`plugins/merchantops/`](plugins/merchantops/) and has its own
  [README](plugins/merchantops/README.md) — install steps, what each slash command and skill
  does, and the two-surfaces (MCP vs. CLI) framing.
- **The eight skills**, as plain `SKILL.md` folders with no plugin wrapper, are mirrored again
  at the repo root under [`skills/`](skills/), for agent platforms that read skill folders
  directly instead of installing a plugin.
- **[`LICENSE`](LICENSE)** — Apache 2.0.

This README covers everything **outside** the plugin architecture: four independent ways to
get MerchantOps tooling, depending on what your platform or shell supports. Pick the one that
matches where you're working — you don't need all four.

## Install path (a): Plugin via marketplace

The packaged install, for platforms with plugin support. Gets you the skills, the four slash
commands, and a reference to MerchantOps' remote MCP server, all in one step.

**Claude Code:**
```
/plugin marketplace add merchantops/claude-plugin
/plugin install merchantops@merchantops
```

**Claude Desktop:** open the Plugins browser, add the marketplace `merchantops/claude-plugin`,
search for "MerchantOps", and install. (Same two slash commands above also work in a chat.)

**What it contains:** the eight skills, four slash commands, and a bundled reference to the
remote MCP server (catalog reads plus a guarded set of writes). That reference points at
`https://mcp.merchantops.ai/mcp` by default; set the `MERCHANTOPS_MCP_URL` environment
variable before starting your client to point the installed plugin's connector at a different
MerchantOps environment instead — no file edit needed.

**What it does NOT contain:** the `mo` CLI itself. The onboarding skills and slash commands
shell out to `mo` — see path (c) below to get it. Installing the plugin alone gives you
conversational catalog reads (and guarded writes) via MCP; it does not by itself let you run
an onboarding.

## Install path (b): Skills only, no plugin

Each skill is a self-contained folder — `SKILL.md` plus, where relevant, a `templates/`
subfolder — with no dependency on the plugin wrapper. Copy or symlink the ones you want into
your agent's skills directory:

```bash
# Claude Code, for example:
cp -R skills/merchantops-cli-reference ~/.claude/skills/
cp -R skills/merchantops-onboard-from-site ~/.claude/skills/
# … or symlink instead of copy, to track updates
```

Any agent platform that reads a directory of `SKILL.md` files the same way can use these —
this is the path for a non-Claude agent, or for using only one or two skills without the
plugin's slash commands or MCP reference. Every skill except `merchantops-cli-reference`
(reference material only — its own frontmatter says "it runs nothing, writes nothing") shells
out to `mo`, so it needs `mo` on `PATH` (path (c), below) — including
`merchantops-competitive-analysis`, which still runs `mo onboarding status` and
`mo onboarding snapshot` before comparing anything, even though it writes nothing.
`merchantops-onboard-from-site` and `merchantops-competitive-analysis` additionally use the
bundled `firecrawl` CLI, under your own key and budget, to fetch pages. Every skill that reads
a crawled page, a spreadsheet or a PDF treats that content as data to report on, never as
instructions to follow.

## Install path (c): CLI only, no plugin, no skills

`mo` is a Python ≥3.11 command-line client (`merchantops-cli`, over the generated
`merchantops-sdk`). It's what every onboarding skill and slash command actually runs, and it's
also usable entirely on its own from a terminal.

**Prerequisites, by how deep you're going — pick the rung that matches you:**

1. **Plugin only** (install path (a), above) — nothing to install locally.
2. **The `mo` CLI** — Python 3.11 or newer, then `mo` itself:
   - **Python 3.11+:** macOS — `brew install python@3.11` (or the
     [python.org installer](https://www.python.org/downloads/)); Linux —
     `apt install python3.11` (or your distro's package manager); Windows — the
     [python.org installer](https://www.python.org/downloads/), checking "Add python.exe to
     PATH" during setup.
   - **Installing `mo`:**
     - **Recommended:** [`uv`](https://docs.astral.sh/uv/) — `uv tool install merchantops-cli`
       — installs `mo` into its own isolated environment and puts it on `PATH` in one step,
       with no virtualenv to manage by hand.
     - **`pipx`** — `pipx install merchantops-cli`, the same isolation model as
       `uv tool install`; run `pipx ensurepath` once if `mo` isn't found afterward (it adds
       `~/.local/bin`, pipx's install target, to `PATH`).
     - **A plain venv** — `python3 -m venv ~/.venvs/mo`, then `~/.venvs/mo/bin/pip install
       merchantops-cli`, then either call `~/.venvs/mo/bin/mo` by its full path, or add
       `~/.venvs/mo/bin` to `PATH` yourself (e.g. `export PATH="$HOME/.venvs/mo/bin:$PATH"` in
       your shell profile) so plain `mo` resolves.
3. **Site-crawl / competitive-analysis skills** (`merchantops-onboard-from-site` *and*
   `merchantops-competitive-analysis`) — everything in step 2, plus Node.js and the
   `firecrawl-cli` package, plus your own Firecrawl API key. No other skill needs either, and
   `pip install merchantops-cli` on its own never pulls a Node toolchain — crawling runs
   agent-side, under your own key and budget, never through MerchantOps infrastructure.
4. **MCP only** (install path (d), below) — nothing to install locally.

<!-- PYPI-FLIP:START state=published -->
**Installing from source instead** — for MerchantOps-supported customers or contributors who
want an editable checkout rather than the PyPI package above: ask your MerchantOps contact for
access to the engineering repository, then, from the clone's root:

```bash
git clone git@github.com:merchantops/product-enricher.git
cd product-enricher
pip install -e ./merchantops-sdk -e ./merchantops-cli
mo --help
```
<!-- PYPI-FLIP:END -->

**Sign in:**
```bash
mo --env prod auth login       # browser-based OAuth; or --env qa for a test org
```
For unattended/CI use, set `MERCHANTOPS_TOKEN` instead of logging in interactively.

**Command reference, exit codes, the safe load order:** see
[`skills/merchantops-cli-reference/SKILL.md`](skills/merchantops-cli-reference/SKILL.md) in
this repository — it's written to be self-contained, so it's useful with no other MerchantOps
documentation on disk.

## Install path (d): MCP only, no local install at all

MerchantOps' remote MCP server needs nothing installed locally — just a client that speaks
MCP:

```
https://mcp.merchantops.ai/mcp
```

Add it as a connector in any MCP-capable client — Claude Desktop, Claude Web, ChatGPT, or
another agent with remote-MCP support — using that URL (or a different MerchantOps
environment's endpoint, if your client lets you point a connector at one), and authenticate
with OAuth in that client; there's no local credential file or environment variable to
manage. This surface is **catalog reads plus a guarded, two-phase-confirmed set of writes
only**: it has read tools for property definitions, product types and lakehouse brands, but no
WRITE tools for any of them (that boundary is permanent, `agent-surface.md`), no crawl-trigger
tool, and no CSV/JSON catalog bulk-import the way `mo import` does — so it cannot drive an
onboarding. Use it for conversational catalog work; use paths (a)-(c) for onboarding a new
customer.

## Which surface can run what

| Surface | Plugin (skills + commands + MCP) | Skills only | CLI only | MCP only |
|---|---|---|---|---|
| Claude Code | ✅ everything | ✅ | ✅ | ✅ |
| Claude Desktop | ✅ (the demoed install) — onboarding skills need a session that can run local shell commands | ✅ | ✅ | ✅ |
| Claude Web (claude.ai) | plugin skills/commands cannot run — no local shell for `mo` (install support depends on the client) | — | — | ✅ MCP only |
| ChatGPT / other MCP clients | — | — | — | ✅ MCP only |
| Other agent platforms | — (plugin format is Claude-specific) | ✅ if the platform reads `SKILL.md` folders | ✅ if the agent has a shell | ✅ if the client supports remote MCP |

---

Questions or access requests: contact MerchantOps. Licensed Apache 2.0 — see
[`LICENSE`](LICENSE).
