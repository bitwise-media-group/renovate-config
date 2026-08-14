# renovate-config

Self-hosted [Renovate](https://docs.renovatebot.com/) bot for the **bitwise-media-group** organisation, plus the shared
org preset every repository inherits. This replaces Dependabot (github-actions pin bumps), the reusable
`update-tools.yaml` (daily `mise lock --bump`), and `dependabot-merge.yaml` (auto-merge of minor/patch) with a single
bot — and adds what none of them covered: bumping the `.mise` → [`toolchain`](https://github.com/bitwise-media-group/toolchain)
submodule pointer and consuming the `# renovate:` annotations already sitting above Dockerfile `ARG *_VERSION` lines.

## How it works

- **[`.github/workflows/renovate.yaml`](.github/workflows/renovate.yaml)** runs Renovate hourly with a token minted
  from the org's **"Renovate" GitHub App**. The App **installation is the scope**: autodiscover walks exactly the
  repositories the App is installed on. Rolling out to more repos means editing the installation, not this repo.
- **[`renovate-global.json5`](renovate-global.json5)** is the bot-side config: autodiscover, API-created (Verified)
  commits via `platformCommit`, submodule cloning and the `mise lock --bump` allowance for lockfile maintenance.
- **[`default.json5`](default.json5)** is the org preset, resolvable as
  `github>bitwise-media-group/renovate-config:default.json5`. The `:default.json5` filename is mandatory in every
  reference: Renovate auto-discovers only `default.json` for a bare `github>owner/repo` string (deliberately, to
  avoid try/fail API calls — [renovate#36877](https://github.com/renovatebot/renovate/discussions/36877)); the
  explicit name is what lets the preset stay JSON5 with inline comments.
  Every discovered repo inherits it with no onboarding file; policy changes here reach the whole org on the next run.

## The merge model

The org's rulesets require PR review, allow only rebase merges on `main`, and require signed commits — none of which a
stock bot satisfies. The pieces that make it work:

1. The App's installation token makes Renovate create branch commits **via the GitHub API**, so they are GitHub-signed
   and show **Verified** — satisfying `required_signatures` with no ruleset bypass needed.
2. The App is a **bypass actor on the pull-request ruleset** (see
   [`github-settings`](https://github.com/bitwise-media-group/github-settings)), so Renovate can squash-merge its own
   PRs without an approval. GitHub-native automerge is deliberately off: it could satisfy neither the approval rule nor
   the signed-commit rule (a rebase merge would land the branch commits unsigned). Renovate merges via the API on a
   later run once every check on the head is green; the server-side squash commit is web-flow signed.
3. `rebaseWhen: "conflicted"` merges behind-base PRs **without** a rebase-and-rerun cycle, and Renovate automerges at
   most one PR per base branch per run — the hourly cron is what gives same-day throughput for a queue of green PRs.

## Policy summary (the preset)

- **Cooldown**: `minimumReleaseAge: 3 days` (surfaced as the `renovate/stability-days` pending check). Exception: no
  cooldown on `bitwise-media-group/**` — our own releases ship straight through. mise tool bumps get the same 3-day
  cooldown via `MISE_MINIMUM_RELEASE_AGE`, enforced by `mise lock --bump` itself.
- **Automerge**: stable (`>=1.0.0`) minor/patch updates, and mise.lock `lockFileMaintenance` PRs. Majors and `0.x`
  wait for a human — approve, then the usual `/merge` label flow.
- **Commit types**: `chore(deps)` by default (dev-toolchain bumps don't release);
  **github-actions bumps are `fix(deps)`** because SHA pins in reusable workflows ship to consumers and must cut a
  release-please patch.
- **Grouping**: minor/patch github-actions bumps group by publisher (`actions`, `github`, `bitwise-media-group`,
  `codecov`, `google`, `goreleaser`) — majors and `0.x` fall out as individual PRs so they never block a group. Other
  ecosystems get `group:monorepos`' source-repo grouping from `config:recommended` (e.g. all otel-go core modules in
  one PR). gomod bumps run `go mod tidy` (`postUpdateOptions: gomodTidy`).
- **Submodules**: the git-submodules manager is on. A repo opts its `.mise` submodule in by setting `branch` in
  `.gitmodules` to the toolchain's full semver tag (e.g. `v2.3.0`) so updates classify as major/minor/patch; Renovate
  keeps the tag current from then on.

## Per-repo overrides

A repository can add its own `.github/renovate.json5`; it is merged on top of the org preset. To keep the preset as a
base, extend it explicitly:

```json5
{
  extends: ["github>bitwise-media-group/renovate-config:default.json5"],
  // overrides here, e.g. repos where mise tools are part of the shipped product:
  // lockFileMaintenance: { commitMessageAction: "…" }, semanticCommitType: "fix", …
}
```

To opt a repo out entirely, uninstall the App from it (or set `{ enabled: false }` in the repo's own config).

## One-time org setup

1. Create the org-owned **"Renovate" GitHub App** (webhook off). Permissions: Checks RW, Commit statuses RW, Contents
   RW, Issues RW, Pull requests RW, Workflows RW, Administration R, Members R, Dependabot alerts R. Install it on the
   target repositories (the installation is the autodiscover scope).
2. Set the org variable `RENOVATE_CLIENT_ID` and org secret `RENOVATE_PRIVATE_KEY` (all repositories — mirrors the
   FF Merge pair).
3. In [`github-settings`](https://github.com/bitwise-media-group/github-settings), add the App to the
   **pull-request** ruleset's bypass actors. Do **not** add it to the release-branch-security ruleset — its commits
   are Verified, so it never needs to bypass `required_signatures`.

## Operating the bot

- **Dry run**: `workflow_dispatch` with `dry-run: full` (and `log-level: debug`) logs everything Renovate would do —
  detected managers, resolved preset, planned PRs — without writing anything.
- **Dependency Dashboard**: each repo gets a "Dependency Dashboard" issue listing pending/open/blocked updates;
  tick a checkbox there to force retries or unblock rate-limited PRs.
- **Rollback**: disable this workflow, close open `renovate/*` PRs and the dashboard issue, and remove the App from
  the ruleset bypass list. Renovate keeps no state outside GitHub.
