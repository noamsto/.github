# Reusable workflows

Callers own the trigger and `permissions`; the called workflow cannot exceed
the caller's token. Pin `@v1`.

## auto-assign

```yaml
name: Auto-assign issues

on:
  issues:
    types: [opened]

permissions:
  issues: write

jobs:
  assign:
    uses: noamsto/.github/.github/workflows/auto-assign.yml@v1
```

| Input | Default | Meaning |
| --- | --- | --- |
| `assignees` | `noamsto` | Comma-separated logins |

## dependabot-auto-merge

Enables squash auto-merge on Dependabot PRs, except semver-major bumps outside
`exempt-ecosystems`. Pass `exempt-ecosystems: ''` to opt out of the default
`nix` exemption. Any PR whose update type Dependabot reports as empty still
auto-merges.

This workflow sets no token ceiling of its own, so the caller must always
declare `permissions:`; without it the job runs with the repository's default
token. The job inherits the caller's permissions: grant `contents: write` and
`pull-requests: write` with `GITHUB_TOKEN`, but only `contents: read` and
`pull-requests: read` with an App token, since the merge uses the App.
The App token requests only contents and pull-requests. github-actions bumps
edit workflow files, which an App can merge only if it holds `workflows`
permission; confirm that before relying on this for them.

With `GITHUB_TOKEN`:

```yaml
name: Dependabot auto-merge

on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    uses: noamsto/.github/.github/workflows/dependabot-auto-merge.yml@v1
```

With a GitHub App token, so the merge push triggers CI and release workflows.
`secrets: inherit` does not work here: the secret must be passed explicitly
as `app-private-key`.

```yaml
name: Dependabot auto-merge

on: pull_request_target

permissions:
  contents: read
  pull-requests: read

jobs:
  auto-merge:
    uses: noamsto/.github/.github/workflows/dependabot-auto-merge.yml@v1
    with:
      app-client-id: ${{ vars.VENDOR_HASH_APP_CLIENT_ID }}
    secrets:
      app-private-key: ${{ secrets.VENDOR_HASH_APP_PRIVATE_KEY }}
```

| Input / secret | Default | Meaning |
| --- | --- | --- |
| `exempt-ecosystems` | `nix` | Comma-separated ecosystems (no spaces) whose majors still auto-merge |
| `app-client-id` | empty | App client ID; set to merge with an App token |
| `app-private-key` (secret) | none | App private key; required when `app-client-id` is set |

## dependabot-vendor-hash

Dependabot bumps `go.mod`/`go.sum` but cannot know the `vendorHash` a Nix
expression pins, so each Go bump fails the `nix` check until the hash is
fixed. This workflow builds the PR, lets `determinate-nixd fix hashes` correct
the hashes and pushes the corrected `*.nix` files onto the Dependabot branch.
The App push re-triggers CI; the rerun finds the hashes right and does nothing.

Run it on `pull_request_target`: Dependabot-triggered `pull_request` events
cannot read Actions secrets. The caller owns the trigger and must grant
`contents: read`, since a called workflow cannot exceed the caller's token.

```yaml
name: Dependabot vendor hash

on:
  pull_request_target:
    types: [opened, synchronize, reopened]
    paths:
      - go.mod
      - go.sum

permissions:
  contents: read

jobs:
  vendor-hash:
    uses: noamsto/.github/.github/workflows/dependabot-vendor-hash.yml@v1
    secrets:
      VENDOR_HASH_APP_PRIVATE_KEY: ${{ secrets.VENDOR_HASH_APP_PRIVATE_KEY }}
```

Unlike `dependabot-auto-merge`, the secret here is named like the caller's, so
`secrets: inherit` also works. The explicit form is preferred: inherit hands
every caller secret to the called workflow (only the `push` job references
one), and only the explicit form can map a differently named secret. Add no
`concurrency` named `vendor-hash-<pr number>` or `vendor-hash-push-<pr
number>`: the workflow uses those groups itself.

| Input / secret | Default | Meaning |
| --- | --- | --- |
| `build-attr` | `.#default` | Flake attribute to build; it must fail on a stale hash |
| `app-client-id` | `vars.VENDOR_HASH_APP_CLIENT_ID` | App client ID, read from the caller's variables when empty |
| `VENDOR_HASH_APP_PRIVATE_KEY` (secret) | none | App private key; required when a hash needs pushing |

App setup: install a GitHub App with Contents read/write on the repo, then
set the client ID as the `VENDOR_HASH_APP_CLIENT_ID` Actions variable and the
private key as the `VENDOR_HASH_APP_PRIVATE_KEY` Actions secret. They must be
Actions secrets and variables, not Dependabot ones, because
`pull_request_target` reads the Actions set. A missing key fails the `push`
job, and so only when a hash actually changed.

Trust split. The PR is untrusted: its dependency code is compiled and tested
by the build, so the work is two jobs.

- `build` has a read-only token and no secret (the nix action's own token is
  blanked, and it uses no cache a tainted run could poison). It uploads only a
  diff of tracked `*.nix` files. If the build fails and `fix hashes` changes
  nothing, the job fails: that is not a stale hash.
- `push` is the only job that sees the App key, and it never runs nix or
  anything from the PR. It validates the diff before minting a token: in-place
  edits of tracked `*.nix` files only (no creates, deletes, renames, mode
  changes, binaries or symlinks), and with every SRI hash replaced the patched
  files must equal the originals, so only hash strings can change. The token
  carries `contents: write` on the calling repo alone, and the push is
  non-force and skipped if the branch moved.

Residual risk: the content check cannot tell a vendor hash from another
fixed-output hash, so a tainted build can still swap any SRI hash in a `.nix`
file it already knows. The PR's own CI and review still see the result.

## Cascade: cascade-notify and cascade-bump

When `main` goes green a sender notifies its consumers; each receiver bumps
the matching flake input and merges once its own CI is green on the bump.
The sender fires a `repository_dispatch` of type `upstream-updated` with
payload `{"source": "noamsto/<repo>", "rev": "<40-hex sha>"}`. A receiver
maps that source through its own `.github/upstreams.json`.

Topology today: aeye, prdash, tmux-remux -> tmux-og -> nix-config -> the
private host flake.

### GitHub App setup

The workflows use one GitHub App. Grant it Contents read/write, Pull
requests read/write and Metadata read; nothing else, since bumps only touch
`flake.lock`. Install the App on every repo in the cascade. In each such
repo set the `CASCADE_APP_ID` variable (the App ID; the client ID also
works) and the `CASCADE_APP_PRIVATE_KEY` secret. Sender repos also set the
optional `CASCADE_CONSUMERS` secret for private recipients (see below).

The called workflows read their variables from the calling repository:
GitHub resolves `vars.CASCADE_APP_ID` in the caller's context, not in
noamsto/.github. Set it in each sender and receiver repo, or as an
organization variable; nothing is needed in noamsto/.github.

"Allow auto-merge" is not needed. The App must not be a bypass actor in the
receiver's rulesets. Commits are authored as the App's bot user.

The App has no Workflows permission. If GitHub refuses the bot branch's
force-push because the update spans workflow-file changes, the run deletes
the bot branch (which closes its PR) and pushes it fresh with a new PR.
Granting the App Workflows write avoids that.

## cascade-notify

Runs in the sender. It notifies the union of the recipients in
`.github/consumers.json` (optional; absent means none) and the caller's
`CASCADE_CONSUMERS` secret, de-duplicated. Every entry must be
a `noamsto/<repo>` string matching `^noamsto/[A-Za-z0-9._-]+$`; a malformed
file or secret fails the run.

`.github/consumers.json` is an array of `noamsto/<repo>` strings:

```json
["noamsto/tmux-og"]
```

A public repo must not name a private repo in a committed file, so list
private recipients in `CASCADE_CONSUMERS` instead: a JSON array of
`noamsto/<repo>` strings, e.g. `["noamsto/private-host"]`.

`CASCADE_CONSUMERS` is a secret: repository variables are never masked, so a
private recipient would show up in public run logs, job names and check
names. The workflow masks each recipient name before it is used, and with no
recipients it logs a warning and still exits green.

Caller snippet:

```yaml
name: Cascade notify

on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]

permissions:
  contents: read

jobs:
  notify:
    if: >-
      github.event.workflow_run.conclusion == 'success' &&
      github.event.workflow_run.event == 'push' &&
      github.event.workflow_run.head_branch == 'main'
    uses: noamsto/.github/.github/workflows/cascade-notify.yml@v1
    with:
      rev: ${{ github.event.workflow_run.head_sha }}
    secrets:
      CASCADE_APP_PRIVATE_KEY: ${{ secrets.CASCADE_APP_PRIVATE_KEY }}
      CASCADE_CONSUMERS: ${{ secrets.CASCADE_CONSUMERS }}  # optional
```

`secrets: inherit` also works because the secret name matches, but it hands
every caller secret to the called workflow, so the explicit form is
preferred.

When `rev` is the triggering `workflow_run`'s head sha, the workflow requires
that run to be a successful `push` on `main`; a different `rev` is trusted as
a bot landing. It mints one token per recipient, scoped to that repo; one
failing recipient does not stop the others.

A repo whose landing job merges with `GITHUB_TOKEN` must send its own
events: those merges start no workflows. Add a job after the landing job,
`needs:` it, and pass the landed sha from an output. The job may live in a
workflow triggered by `workflow_run`, `schedule`, `workflow_dispatch` or
`push`:

```yaml
  notify:
    needs: land
    uses: noamsto/.github/.github/workflows/cascade-notify.yml@v1
    with:
      rev: ${{ needs.land.outputs.sha }}
    secrets:
      CASCADE_APP_PRIVATE_KEY: ${{ secrets.CASCADE_APP_PRIVATE_KEY }}
      CASCADE_CONSUMERS: ${{ secrets.CASCADE_CONSUMERS }}  # optional
```

| Input / secret / variable | Default | Meaning |
| --- | --- | --- |
| `rev` | none | 40-hex commit on `main` whose CI passed |
| `CASCADE_APP_PRIVATE_KEY` (secret) | none | App private key |
| `CASCADE_CONSUMERS` (secret, optional) | none | Private recipients |

## cascade-bump

Runs in the receiver, triggered by the dispatch. It maps the sending repo
through the receiver's own `.github/upstreams.json`, an object from
`noamsto/<repo>` to the root flake input name:

```json
{"noamsto/aeye": "aeye", "noamsto/prdash": "prdash"}
```

Caller snippet:

```yaml
name: Cascade bump

on:
  repository_dispatch:
    types: [upstream-updated]

permissions:
  contents: read
  checks: read
  statuses: read

jobs:
  bump:
    uses: noamsto/.github/.github/workflows/cascade-bump.yml@v1
    secrets:
      CASCADE_APP_PRIVATE_KEY: ${{ secrets.CASCADE_APP_PRIVATE_KEY }}
```

Add no workflow- or job-level `concurrency`: the workflow already coalesces
per input, and a caller-level group would make one input's pending run
replace another's. The receiver's CI must run on the bot branch's push or on
`pull_request`; if no check reports within about 10 minutes the run fails.

Unknown sources, a source that does not look like `noamsto/<repo>`, and
malformed payloads are logged and ignored (exit 0). A missing or invalid
`.github/upstreams.json`, or an entry that is not a direct root input of
`flake.lock`, fails the run.

### Merge

The bump job mints the App token twice: once before the push and again
after the CI wait. With no required status checks, GitHub auto-merge would
merge at once, and nix-config and the private host flake have none. So the
job waits on the pushed commit's full check rollup, settles 60s, mints a
fresh App token (installation tokens expire after an hour), checks the
default branch has not moved, and squash-merges with `--match-head-commit`.
The receiver must allow squash merges.

If the default branch moved during the wait, the run sends the same
`upstream-updated` event to its own repo (same source and rev, so the event
shape is unchanged) and exits. Red CI leaves the PR open for a human.

Accepted residual risks:

- the default branch can move between the final check and the merge;
- two different-input bumps can merge back to back, each tested only on its
  own;
- a `pull_request`-only check that registers more than 60s after every other
  check finished green can be missed.

The job holds a runner while CI runs: billed minutes on a private repo,
bounded by `timeout-minutes`. Private upstream flake inputs are not
supported, since nix only gets this repo's `GITHUB_TOKEN`.

| Input / secret | Default | Meaning |
| --- | --- | --- |
| `timeout-minutes` | `120` | Bump job ceiling, covering the CI wait |
| `CASCADE_APP_PRIVATE_KEY` (secret) | none | App private key |
