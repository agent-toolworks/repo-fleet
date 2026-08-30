# repo-fleet

An opinionated environment for working across many repositories at once.

**Start with [GETTING-STARTED.md](GETTING-STARTED.md).**

Two levels, and the distinction is the whole design:

- **The fleet** — every repo cloned side by side under one root, all on their
  default branch. Read-mostly: it exists to be searched and to be the base for
  worktrees, so the daily refresh can hard-reset without losing anything.
- **A ticket workspace** — a directory per ticket holding **git worktrees** (not
  fresh clones) of only the repos that ticket changes, each on a branch named
  for the ticket.

```sh
claude plugin marketplace add agent-toolworks/plugins
claude plugin install fleet-workspace@agent-toolworks
claude plugin install code-search@agent-toolworks
```

Two plugins, one catalog. They are **peers, not layers** — neither depends on
the other in code. They share a convention: `FLEET_ROOT`, `TICKETS_ROOT`, and
the directory layout above.

| Plugin | Lives in | Does |
|---|---|---|
| `fleet-workspace` | this repo | clones the fleet, creates and tears down ticket workspaces |
| `code-search` | [code-search-fleet](https://github.com/agent-toolworks/code-search-fleet) | one search interface over five engines, which understands the two-level layout |

`code-search` is genuinely useful on its own — point `FLEET_ROOT` at any
directory of repos — and is installable from its own repo. It is listed here so
that adopting the environment is one `marketplace add` rather than two.

## What it does

```sh
fleet-init --config --fleet-root ~/code/fleet --tickets-root ~/tickets
fleet-init --clone acme/inventory-api acme/checkout-service acme/web-monorepo
fleet-init --cron                 # daily refresh line for crontab -e

new-ticket PROJ-123 inventory-api checkout-service
close-ticket PROJ-123
```

But you mostly do not run these. They are **Claude Code skills** — describe the
intent and the agent runs them:

> "Start ticket PROJ-123, it changes the inventory API and the checkout service."

## Why worktrees, and why search has to span both levels

A ticket costs no re-clone of a large repo, and the fleet stays the single copy
on disk. More importantly, searching **only** a ticket workspace makes renaming
a shared route look safe — because the caller that breaks lives in a repo the
ticket does not contain:

```
searching: PROJ-123 (2 repo(s), your branch) + fleet (8 repo(s), main)
checkout-service/src/main/kotlin/.../InventoryClient.kt:12
web-monorepo/packages/admin-ui/src/restockTool.ts:5
```

That second hit is the justification for the whole arrangement.

## Customising it

Nothing company-specific is baked into the scripts. Two extension points, both
covered in [GETTING-STARTED.md](GETTING-STARTED.md):

- **Branch naming** — `BRANCH_PREFIX`, or `BRANCH_TEMPLATE="users/zvi/{ticket}"`
  when the convention is not a prefix.
- **`post-create` hooks** — an executable run once per repo after its worktree
  is created, either machine-wide (`~/.config/repo-fleet/hooks/post-create`) or
  owned by a repo (`<repo>/.fleet/post-create`). This is where `npm ci`, a
  package restore, or copying a local `.env` belongs.

`fleet-init --status` shows the config, the cloned repos, every hook it can see,
and live ticket workspaces — including hooks that exist but are not executable,
the usual reason one silently does nothing.

## Verifying

```sh
scripts/verify-workspace          # 30 checks against a throwaway fleet
```

It builds its own fleet of small git repos with real bare origins, then drives
the real scripts against it — so `origin/HEAD` detection, worktrees, and "not in
origin/main" behave as they do in production. Nothing touches your config:
`FLEET_CONFIG_DIR`, `FLEET_ROOT` and `TICKETS_ROOT` are redirected into a
temporary directory, which is why `FLEET_CONFIG_DIR` is overridable at all.

The guards below are the reason it exists. They are the part that can lose
work, and a guard nobody tests is one that quietly stops working.

## Safety properties

- `new-ticket` validates every repo name **before creating anything**, so a typo
  cannot leave a half-built workspace. Re-running it **resumes** rather than
  restarts.
- `close-ticket` refuses to remove a worktree with uncommitted changes, or one
  holding commits not in the default branch, and exits non-zero when blocked.
  `--force` overrides.
- `refresh-fleet` skips a fleet repo with local modifications rather than
  resetting it — that means someone worked directly in the fleet, which the
  model forbids.
- A failing `post-create` hook warns and continues: a workspace that exists but
  is not fully provisioned beats no workspace.
- Environment variables override `fleet.env`, so a one-off
  `BRANCH_PREFIX=hotfix/ new-ticket …` or a second fleet for one command works.

## License

[Apache-2.0](LICENSE). Chosen over MIT for the explicit patent grant, which is
what makes a corporate legal review a formality rather than a conversation —
a fleet of repos side by side is an organisation's layout, not an individual's,
so the install instructions invite exactly the audience that has to ask. Without
a LICENSE file the default is all rights reserved, which blocks not just use but
vendoring, internal mirroring, and redistribution through an internal plugin
marketplace.
