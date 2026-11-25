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
`exempt-ecosystems`. Callers need `contents: write` and `pull-requests: write`
even with `pull_request_target`. Repos that never want nix majors merged
(tmux-og, aeye) pass `exempt-ecosystems: ''`.

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
  contents: write
  pull-requests: write

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
