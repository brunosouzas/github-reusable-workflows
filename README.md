# github-reusable-workflows

Reusable GitHub Actions workflows for Bruno's open source repositories, pinned by tag (`@vX.Y.Z`). The private repositories use the Azure DevOps equivalent, `azure-devops-reusable-templates`.

| Workflow | What it does | Inputs | Secrets |
|---|---|---|---|
| `.github/workflows/release-please.yml` | Delivery model 2: release-please in manifest mode keeps the release pull request; merging it makes the `vX.Y.Z` tag and the GitHub release. Runs as the GitHub App `brunosouzas-release` (a pull request opened with `GITHUB_TOKEN` starts no checks, and the `v*` tag ruleset lets only the app create tags), with a token limited to the caller's repository | `app-id` (`5576835`), `skip-github-release` (`false`; `true` when the caller's own release workflow creates the tag, as the npm packages do) | `release-app-private-key` |

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
    secrets:
      release-app-private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
```

The caller needs `release-please-config.json` and `.release-please-manifest.json`, the app installed on the repository, the secret `RELEASE_APP_PRIVATE_KEY` (the app's private key, PEM; Bruno sets it), and the app as the bypass of its `v*` tag ruleset (`scripts/repo-protection.py --model <repo> --check <check> --bypass-app 5576835 --apply` in the ai-engineering-kit). Pull request titles are Conventional Commits: release-please releases only `feat` and `fix`.

## Releases

This repository is in delivery model 2 itself: `release.yml` calls the workflow above on every push to `main`; Bruno merges the release pull request. Change a workflow in a pull request, then pin a consumer to the new tag in its own pull request.
