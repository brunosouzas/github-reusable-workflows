# github-reusable-workflows

Reusable GitHub Actions workflows for Bruno's open source repositories, pinned by tag (`@vX.Y.Z`). The private repositories use an Azure DevOps equivalent.

| Workflow | What it does | Inputs | Secrets |
|---|---|---|---|
| `.github/workflows/release-please.yml` | Delivery model 2: release-please in manifest mode keeps the release pull request; merging it makes the `vX.Y.Z` tag and the GitHub release. Runs as the caller's own release GitHub App (one app per repository: a pull request opened with `GITHUB_TOKEN` starts no checks, and when release-please makes the tag, the `v*` tag ruleset lets only the app create it), with a token limited to the caller's repository, in the caller's environment `release-please` | `environment` (`release-please`), `skip-github-release` (`false`; `true` when the caller's own release workflow creates the tag, as the npm packages do) | `RELEASE_APP_PRIVATE_KEY`, passed by name; it resolves from the environment `release-please`. `RELEASE_APP_CLIENT_ID` is read from the environment's variables |
| `.github/workflows/pr-title.yml` | Fails a pull request whose title is not a Conventional Commit (`<type>[(<scope>)][!]: <description>`), since the squash merge makes the title the commit release-please reads. No third-party action | none | none |

## Use

```yaml
name: Release PR
on:
  push:
    branches: [main]
permissions: {}
jobs:
  release-please:
    uses: brunosouzas/github-reusable-workflows/.github/workflows/release-please.yml@v0.1.0
    with:
      skip-github-release: true   # only when the repository's own release workflow makes the tag
    secrets:
      RELEASE_APP_PRIVATE_KEY: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
```

```yaml
name: Title
on:
  pull_request:
    types: [opened, edited, synchronize, reopened]
permissions: {}
jobs:
  pr-title:
    uses: brunosouzas/github-reusable-workflows/.github/workflows/pr-title.yml@v0.1.0
```

The caller of `release-please.yml` needs:

- `release-please-config.json` and `.release-please-manifest.json`;
- its own release GitHub App (Contents, Pull requests and Issues read and write, no webhook), installed on that repository alone;
- the environment `release-please`, with deployment branches limited to `main` and no reviewer, holding the variable `RELEASE_APP_CLIENT_ID` (the app's Client ID) and the secret `RELEASE_APP_PRIVATE_KEY` (its private key, PEM; Bruno sets it). The reusable job declares the environment, so both resolve to its values: it reads the variable directly, and the caller passes the secret by name (`secrets: RELEASE_APP_PRIVATE_KEY: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}`), which GitHub resolves from that environment even though the secret exists only there. Not passed, it is empty and the job stops at its first step;
- when release-please makes the tag, the app as a bypass of its `v*` tag ruleset (creations restricted to the app and the admin).

The caller of `pr-title.yml` sets the squash merge's default commit message to the pull request title (`gh api -X PATCH repos/<owner>/<repo> -f squash_merge_commit_title=PR_TITLE -f squash_merge_commit_message=PR_BODY`) and makes `pr-title / conventional-commit` a required check (the name holds when the caller's job id is `pr-title`). The scope is lowercase; GitHub's default revert title (`Revert "..."`) fails: retitle it `revert: <description>`. Commit messages and pull request titles are Conventional Commits: release-please reads the commits that reach `main` and releases `feat`, `fix`, `perf` and `revert`; other types release nothing (a `Release-As: x.y.z` line in a commit body forces a version).

After a release that never got its tag (the caller's release workflow rejected or failed), release-please opens no new release pull request until the tag `v<version>` exists: push the tag by hand to release that version; the next push to `main` marks the release pull request tagged.

## Releases

This repository is in delivery model 2 itself: `release.yml` calls the workflow above on every push to `main`, as its own app `github-reusable-workflows-release`, and `title.yml` checks its pull request titles; Bruno merges the release pull request. Change a workflow in a pull request, then pin a consumer to the new tag in its own pull request.
