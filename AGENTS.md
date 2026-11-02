# github-reusable-workflows: project context

Reusable GitHub Actions workflows for Bruno's open source repositories, consumed by tag. Read [README.md](README.md).

## Technology

Stack: GitHub Actions, YAML
Delivery: B package
Model: 2 release PR
Integration branch: `main`
Data: none
Article: none

- Build: not applicable
- Test: actionlint (the `CI` workflow)
- Run locally: not applicable
- Deliver and verify: a merge updates the release pull request; Bruno merges it, which makes the `vX.Y.Z` tag and the GitHub release: the `Release` run is green and the tag exists; consumers move to it in their own pull requests

## Rules for this repository

- Board area: `setup` (Azure Boards, project `brunosouzas`).
- OpenSpec: no: every change takes the small path.
- Every action is pinned by commit SHA with its tag in a comment; every tool by version and checksum.
- Public repository: never a credential, never a private repository name or path that is not already public.
- Merge: Bruno (shared pipeline templates).
