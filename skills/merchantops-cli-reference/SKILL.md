---
name: merchantops-cli-reference
description: Reference for the MerchantOps `mo` command-line tool. Which command to run, what its flags mean, what each exit code means, the order catalog data has to be loaded in, and how to point it at a local, qa or production environment. Use this instead of guessing a command or a flag.
when_to_use: Use whenever a MerchantOps task needs an exact command, flag, exit code or load order; when a `mo` command failed and you need to know whether that failure is retryable, a permissions problem or a validation problem; or when the user asks what the CLI can do. This skill is reference material only. It runs nothing, writes nothing, and never needs the user's confirmation.
---

# MerchantOps CLI (`mo`) reference

`mo` is the MerchantOps operator CLI. It is the surface the MerchantOps skills drive: every
catalog read, every import and every provisioning action goes through it. This file is
self-contained on purpose, so a skill can work from it without any other repository.

Everything below is either fixed by the CLI's own contract or derivable at runtime from
`mo api --list` (the operation inventory) and `mo import order` (the load order). Prefer the
runtime answer when the two ever disagree, and say so rather than silently picking one.

## The command tree

| Group | What it is for |
|---|---|
| `mo auth` | Sign in, sign out, report the current credential, resolve live identity and permissions. |
| `mo onboarding` | Read the org's current shape, snapshot it, and produce empty artifact templates. |
| `mo import` | The load path: the safe order, client-side validation, plans, and gated loads. |
| `mo jobs` | Follow an async job to its terminal state. |
| `mo provision` | Staff-only: find-or-create a customer organization, invite members, set roles. |
| `mo api` | The generic tree, derived from the typed SDK at runtime. One command per API operation. |

There is no `mo products`, `mo product-types` or `mo crawl` group. Those older shapes are gone.
Catalog reads live under `mo api <tag> <get-operation>`; catalog writes live under the curated
`mo import` and `mo provision` commands.

## Start every session the same way

1. `mo onboarding status` — returns `{property_definitions:{total,system,common}, product_types,
   categories, display_groups, brands, products, variants, price_records, base_template_seeded,
   missing_system_properties[]}`. This is the difference between proposing a schema from scratch
   and extending the one the org already has. Adapt to what it reports; never assume an empty org.
2. `mo auth whoami` when a write is coming. It is a live call that returns identity plus the
   member's resolved permissions. The token's own claims never carry RBAC, so decoding it answers
   the wrong question. If the caller lacks a permission a step needs, say so before planning it.

## Environments and credentials

`mo --env local|qa|prod <command>` resolves the API URL, the OAuth issuer and the audience for
that environment, and fetches the public client id from the API itself. `--api-url` overrides the
preset. Precedence: `--api-url` > `--env` > environment variables > config file > default. The
stored token is keyed by the resolved API URL, so switching environments forces a fresh sign-in
rather than presenting a qa token to production.

Credentials, in the order the CLI consults them:

| Seam | Use |
|---|---|
| `MERCHANTOPS_TOKEN` | A bearer token, used verbatim, no refresh. The CI / non-interactive credential. |
| `MERCHANTOPS_API_KEY` | A server API key. Also a non-interactive credential. |
| OS keychain | Written by `mo auth login`, a short-lived refreshable session. The interactive default. |

**Both are environment variables, never command-line arguments.** A `--token` flag exists and its
own help text says it is unsafe: arguments are visible in the process list on a shared machine.
Never put a credential in a command you write into a transcript.

## Curated commands

### `mo auth`

| Command | Behaviour |
|---|---|
| `mo auth login [--env local\|qa\|prod] [--no-browser]` | Browser sign-in (PKCE); stores the token. |
| `mo auth logout` | Clears the stored token for the resolved API URL. |
| `mo auth status` | Reports whether a credential is available and where it came from. Always exits 0. |
| `mo auth whoami` | Live identity plus resolved permissions. |

### `mo onboarding`

| Command | Behaviour |
|---|---|
| `mo onboarding status` | The org's current counts and gaps. Every skill's first call. |
| `mo onboarding snapshot --out DIR` | Writes the org's current schema and catalog into the bundle layout below, so "what you have" and "what I propose" diff with `diff`. |
| `mo onboarding template <artifact> --out FILE` | A header-only CSV or skeleton JSON for one artifact. Makes no network call and needs no credential. |

### `mo import`

| Command | Behaviour |
|---|---|
| `mo import order` | The safe load order as JSON. Never hardcode the order; ask for it. |
| `mo import validate <artifact> FILE` | Client-side schema and referential validation. Sends no writes. |
| `mo import plan <artifact> FILE` | Validate, then read live server state and report creates, updates, skips and conflicts. Still no writes. |
| `mo import load <artifact> FILE --execute --confirm <hash> [--resume] [--enrich] [--async]` | The gated load. Without `--execute` this is exactly `plan`. |
| `mo import bundle DIR [--execute --confirm <hash>] [--resume] [--stop-on-error]` | Validates every artifact in the directory first, then loads them in the safe order. |

Artifact names: `property-definitions`, `property-values`, `property-overrides`, `categories`,
`product-types`, `type-properties`, `brands`, `products`, `products-base`, `variants`, `prices`,
`map-policy`.

`validate` prints `valid`, `rows`, `sha256`, `confirm`, `errors[]` (each with `row`, `column`,
`code`, `message`), `warnings[]` and a `referential` block. It exits 0 when valid and 8 when not,
having sent nothing.

**A `referential` block reporting `checked: false` means the reference checks did not run** — no
credential, or the API was unreachable — and the accompanying warning says which checks were
skipped. Schema rules still ran. A file can be `valid: true` and still carry unresolved
references, so report that honestly instead of calling it "validated".

`mo import bundle` processes only the artifacts actually present in the directory. Absent optional
files are listed as `absent` in the summary; that is not an error.

### `mo jobs`

`mo jobs watch JOB_ID [--interval 3] [--timeout 900] [--fail-on-review]` polls the job until it
reaches a terminal state, streams progress to stderr, and prints the final job JSON on stdout.

| Job status | Exit |
|---|---|
| `completed`, `completed_with_warnings` | 0 |
| `failed`, `partial_success`, `cancelled` | 5 |
| `awaiting_review` | polling stops, the job JSON prints (carrying `needs_review_items`), exit 0 — or exit 5 with `--fail-on-review` |
| timeout reached | 7, with the last-seen job JSON |

`awaiting_review` is not a failure and not terminal: items were escalated to the **Agent Inbox**
for a human to resolve. Say that plainly and point the user there. Branch on `status`, not on the
exit code alone.

### `mo provision`

Staff-only. Needs `org:provision` held by a member of the platform organization; an ordinary
customer credential gets a permissions error naming what is missing.

| Command | Behaviour |
|---|---|
| `mo provision org create --name N --slug S [--email-domain D] [--seed/--no-seed] --execute --confirm <hash>` | Find-or-create. A second run against the same slug reports the existing org instead of creating a duplicate. |
| `mo provision org list [--query Q]` / `mo provision org get ORG_ID` | Read-only. |
| `mo provision org seed ORG_ID --execute` | Idempotent pre-warm, safe to re-run. Needs no confirm hash. |
| `mo provision member invite --org ORG_ID --email E [--name N] [--role R] --execute --confirm <hash>` | Reserved roles are refused. |
| `mo provision member set-roles --org ORG_ID --member M --role R --execute --confirm <hash>` | Same. |

A live Stytch project is a second, higher gate: it additionally requires `--allow-live` and the
org name re-typed interactively. There is no delete in any form.

### `mo api` — the generic tree

`mo api <tag> <operation>` exposes one command per API operation, derived from the SDK at runtime.
`mo api --list` emits the whole inventory as JSON (`{tags, operations}`, each operation carrying
`tag`, `command`, `method`, `path`, `op_id`); `mo api --help` lists the tags.

Use it for **reads**: `mo api products list-products`, `mo api jobs get-job JOB_ID`,
`mo api property-definitions list-property-definitions`, `mo api brands list-brands`. Reads take
ordinary typed options derived from the endpoint's own parameters.

Writes belong in the curated commands. Every non-GET operation in this tree requires `--execute`,
and without it the resolved request plan prints and nothing is sent — but `--execute` is misfire
prevention, not a boundary (see "What the write gate buys" below). Some operations are simply not
registered in this tree at all; running one is an ordinary unknown-command error, and that is the
whole answer. Do not go looking for another way to run it. If the user needs one of those, they do
it themselves in the admin UI.

## Output contract

Machine-readable JSON goes to **stdout**; human progress and error summaries go to **stderr**. On
failure stdout carries an envelope:

```json
{"error": "API returned HTTP 403", "status": 403,
 "detail": "Missing required permission: property_definition:write", "exit_code": 4}
```

Parse stdout. Report the `detail` to the user, because it names the specific thing that was
missing or wrong.

## Exit codes

| Code | Meaning | What to do |
|---|---|---|
| 0 | success | Continue. |
| 1 | unexpected / internal error | A bug. Report the envelope; do not retry blindly. |
| 2 | usage error | Wrong command or wrong flags. Re-read this file; check `mo api --list`. |
| 3 | auth required | 401, not signed in, or an expired session. Run `mo auth login`. |
| 4 | forbidden | 403. The `detail` names the missing permission. Tell the user which role grants it; never route around it. |
| 5 | client error | 400, 404, 409, 422. The request was wrong or the target does not exist. Fix the input. |
| 6 | quota exceeded | 402. The `detail` is an **object** — report its fields (the limit, the resource, the message), never the object itself as a string. |
| 7 | server / transport, retryable | 5xx, connect failure, timeout, or a 429 whose `Retry-After` you should honour. Back off and retry. |
| 8 | local validation failed, nothing was sent | A `validate`/`plan` error, or a `--confirm` hash mismatch. Nothing reached the API. Fix the file and re-validate. |

Exit 8 is the friendly one: it means the CLI caught the problem before any row was written.

## The safe load order

`mo import order` is the machine-readable form and the one to use. Inline, for reference:

| # | Artifact | Bundle file | Requires | Server idempotency | Re-run needs `--resume` |
|---|---|---|---|---|---|
| 1 | `property-definitions` | `01-property-definitions.csv` | — | upsert | no |
| 2 | `property-values` | `02-property-values.csv` | 1 | upsert | no |
| 3 | `categories` | `03-categories.json` | — | insert-only | yes |
| 4 | `product-types` | `04-product-types.csv` | 3 | upsert | no |
| 5 | `type-properties` | `05-type-properties.csv` | 1, 4 | append-only | no |
| 6 | `property-overrides` | `06-property-overrides.csv` (optional) | 1, 4 | upsert | no |
| 7 | `brands` | `07-brands.json` | — | insert-only | yes |
| 8 | `products` | `08-products.json` | 4, 7 | upsert | no |
| 9 | `variants` | `09-variants.csv` (optional) | 8 | insert-only, whole batch | yes |
| 10 | `prices` | `10-prices.csv` | 9 | insert-only | yes |
| 11 | `map-policy` | `11-map-policies/` (optional) | 7, 9 | new version per upload | no |

`products-base` is an alternate for step 8. Variants normally ship embedded in `08-products.json`,
which is why step 9 is optional.

There are **no transactions and no server-side dry run** on any of these endpoints. A step loaded
out of order fails only after it has already written the rows before the failure. That is why
validation is the safety net and why the order is not negotiable.

## Bundle layout

```
onboarding-bundle/
  manifest.json                  {version, generated_at, generated_by, artifacts:[{type, file, sha256}]}
  01-property-definitions.csv
  02-property-values.csv
  03-categories.json
  04-product-types.csv
  05-type-properties.csv
  06-property-overrides.csv      (optional)
  07-brands.json
  08-products.json               {"products": [...]}   variants embedded
  09-variants.csv                (optional)
  10-prices.csv
  11-map-policies/               (optional)
  review/
    properties-common.csv
    properties-by-type.csv
    product-types.csv
    open-questions.md
```

The files under `review/` are the same CSV shape as the artifacts they review, so a human's edits
flow straight back into the load with no transcription step.

## What the write gate buys, and what it does not

Every load is two calls:

1. `mo import validate <artifact> FILE` (or `mo import plan`, or `mo import bundle DIR` with no
   `--execute`) prints `sha256` and `confirm` alongside the errors and warnings.
2. `mo import load <artifact> FILE --execute --confirm <confirm>` recomputes the hash at load time
   and refuses on mismatch with exit 8, having sent nothing.

**Always copy the `confirm` value the tool just printed. Never compute a hash yourself.** For a
single artifact the confirm value happens to equal the file's own sha256; for a bundle it is a
hash over a canonical listing of every artifact and its hash, and for a provisioning plan it is a
hash over a canonical encoding of the resolved plan. Re-deriving it is wrong in two of three
cases, and a stale value from an earlier run is wrong in all three.

Three separate things, never to be conflated:

- **The hash binding is structural for a non-interactive caller.** It makes loading a *different*
  file than the one that was reviewed impossible, and it gives the human a concrete artifact to
  confirm. That is the whole claim: it does not stop a caller that recomputes the hash for a file
  it edited.
- **`--execute` is misfire prevention.** It stops a casually-run or fat-fingered write. It is not a
  boundary against a caller that decides to supply the flag.
- **A skill's own "show the plan and wait" wording is advisory.** It is a prompt.

**The real bound is the member's own RBAC.** The token lives in the environment and the SDK is
importable, so no CLI-layer mechanism stops a caller who bypasses the CLI. Recommend that
onboarding sessions run under a **scoped, non-Owner role** — an editor tier without approval
permissions — so that a compromised or misled session structurally cannot arm a publish. Nothing in
the CLI checks the caller's role before an import; saying so is the point.

## Enrichment is off by default

`mo import load` sends the lakehouse-search, brand-site-scrape and enrich flags as **false** unless
`--enrich` is passed. Two reasons, both real: enrichment is metered and billed per product, so a
large load would bill the whole catalog before the org has a property dictionary worth enriching
against; and the no-enrichment branch is the one that skips unchanged products instead of minting a
new version per product on every re-run.

Enrich **after** the schema is loaded and reviewed, as a separate deliberate step the user asks for.
The asynchronous batch-enrichment operation lives under the `products` tag (`mo api --list` names
it); it is a write, so leave it to the user or run it only on an explicit request in that turn.

## Receipts, `--resume` and re-runs

`mo import load` writes `<FILE>.receipt.json` recording each item's outcome (`created`, `updated`,
`skipped`, `failed`, plus the server's error) and the run's hash. `--resume` re-validates
everything, then skips what the receipt already records as succeeded. Several surfaces are
insert-only — categories, brands, variants and prices — so a naive re-run duplicates or fails
outright. Always `--resume` a re-run.

The receipt is a convenience, not the safety net: if the receipt is missing but the target already
holds rows this artifact would insert, `plan` and `load` refuse with `prior_run_detected` and exit
8 rather than writing on top of a previous run.

## Chunking and caps

The CLI chunks to the server's own limits and never above them: 250 products per request, 250
variant or base-CSV rows, 1000 property definitions, 5000 property values, 5000 product types,
type-property links or overrides. A 20,000-SKU load is 80 product calls. You do not need to chunk
anything yourself.

## When this file does not answer the question

- `mo api --list` — the live operation inventory, including which tag an operation lives under.
- `mo import order` — the live load order with each step's idempotency and resume requirements.
- `mo <group> --help` and `mo <group> <command> --help` — the live flags for any command.

Ask the tool. Do not invent a command that "should" exist.
