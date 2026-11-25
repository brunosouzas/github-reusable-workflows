# MuleSoft delivery on GitHub Actions

Reusable workflows that deliver MuleSoft applications and libraries from public repositories to an Anypoint organisation (Exchange and CloudHub 2.0). GitHub runs reusable workflows only from `.github/workflows/`, so the entry points live there with a `mulesoft-` prefix, and this folder holds their documentation.

| Workflow | For | What it does |
|---|---|---|
| `.github/workflows/mulesoft-library.yml` | a connector or module (`mule-extension`) | Pull request: build and tests. Merge into `main`: the Maven release (tag `vX.Y.Z`, `main` at the next patch SNAPSHOT), the tag published to Exchange, a GitHub release |
| `.github/workflows/mulesoft-app.yml` | an application (`mule-application`) | The whole path below |
| `.github/workflows/mulesoft-deploy.yml` | called by `mulesoft-app.yml` only | Publish a version to Exchange, deploy it from Exchange to one environment, check it |

## The branch strategy

There is no `develop`. Four places: local, TEST (what `main` integrated), UAT (what will go to production) and PROD (tags only).

```text
feature/x ──PR──▶ main ──────────────▶ TEST            (every merge)
                    │
        Criar release (picks merged PRs onto the latest tag)
                    ▼
              release/X.Y.Z ─────────▶ UAT             (every commit; UAT fixes come by PR)
                    │
                 Release (Maven release, as the release app)
                    ▼
                 tag vX.Y.Z ─────────▶ UAT ──approval──▶ PROD
                    │                                     │
        back-merge PR into main ◀─────────────────────────┘ release/X.Y.Z deleted
```

| Event | Who | What runs |
|---|---|---|
| Pull request into `main` or `release/*` | developer | `build`: build with `-DskipMunitTests`, Maven cache, no secret, no deploy |
| Merge into `main` | developer, after review | `TEST`: the SNAPSHOT to Exchange, deploy and check in TEST |
| Run workflow on `main`, action `criar-release` (version, pull requests, base) | tech lead | `release/X.Y.Z` from the latest `v*` tag with the squash commit of each chosen pull request, in merge order; the release pull request (never merged: it is the release's dossier); the pull request that moves `main` to the next minor |
| Commit on `release/X.Y.Z` (Criar release, or a fix merged by pull request) | developer | `UAT`: the release's SNAPSHOT to Exchange, deploy and check in UAT |
| Run workflow on `release/X.Y.Z`, action `release` | tech lead, after the tester's ok | `Release`: `release:prepare` as the release app (tag `vX.Y.Z`, the release at `X.Y.(Z+1)-SNAPSHOT`), then the back-merge pull request: the fixes merged into the release, cherry-picked onto a branch from `main` |
| The tag's push | the release app | `UAT (the tag)`: the tag to Exchange, deploy and check in UAT; then `PROD`, which waits for the approval of the environment `prod`; then the GitHub release, and the release branch is deleted (which closes the release pull request) |
| Run workflow on a `v*` tag, action `auto` | ops | `PROD` only, after the same approval: a rollback to an earlier tag, or a retry |
| Merge the back-merge | tech lead | `TEST` again |

The Maven release's development commit starts with `[skip ci]`, so its push runs nothing; the release commit, the one the tag points to, carries no marker, so the tag's push starts the run that ends in PROD. Configure the application's `maven-release-plugin` so:

```xml
<configuration>
  <tagNameFormat>v@{project.version}</tagNameFormat>
  <scmReleaseCommitComment>chore(release): v@{releaseLabel}</scmReleaseCommitComment>
  <scmDevelopmentCommitComment>[skip ci] chore(release): next development iteration after v@{releaseLabel}</scmDevelopmentCommitComment>
  <pushChanges>true</pushChanges>
</configuration>
```

A library has no environment: merging its pull request is releasing it, so both of its Maven release commits carry `[skip ci]`. A library declares its Exchange publish goals (`exchange-mule-maven-plugin`) in a Maven profile named `exchange`, so `mvn install` works for anyone without an Anypoint account.

The squash message of every merge is written by hand, one line per change, with no skip-CI marker anywhere (`[skip ci]`, `[ci skip]`, `[no ci]`, `[skip actions]`, `[actions skip]`, `skip-checks: true`): a squash commit that carries one deploys nothing, and Criar release and the back-merge pick squash commits.

## The groupId

Git holds a canonical groupId (input `group-id`, default `com.brunosouzas.mulesoft`). Exchange takes an asset only under its organisation's ID, so the publish and deploy jobs replace the project's groupId with `ANYPOINT_ORG_ID` in their checkout only. Nothing commits the organisation's ID, so a new trial organisation needs no change in Git.

An application that depends on a library of its own author builds that library from its released tag (input `install-from-source`, one `owner/repo pom-property` per line, for example `brunosouzas/mule4-circuit-breaker circuit.breaker.version`). Anyone can then build the application, a fork's pull request included, without an Anypoint account.

## Why no MUnit

MUnit runs on the Mule Enterprise runtime, which comes from the MuleSoft Enterprise repository with a customer credential tied to a MuleSoft subscription. That credential never goes to a public repository, not even as a secret: using an employer's licence outside its business is not ours to decide. The workflows build with `-DskipMunitTests`; MUnit runs on the developer's machine (`mvn clean test`) and is a pull request checklist item. In a company, store the Enterprise repository credential in the build job's environment and remove `-DskipMunitTests`.

## Security

- **Forks.** Every check runs on `pull_request`, never `pull_request_target`, so a fork's pull request gets a read-only token and no secret, and only the `build` job runs. Every secret lives in an environment, and each environment admits only its refs: `test` admits `main`; `uat` admits `release/*` and `v*` tags; `prod` admits `v*` tags; `release` admits `main`, `release/*` and `v*` tags.
- **Production.** The environment `prod` has a required reviewer. One maintainer both starts the Release and approves PROD, so "Prevent self-review" stays off; a team turns it on.
- **The release app.** The commits, tags, branches and pull requests of a release are made by a GitHub App with a token limited to the repository (contents and pull requests). A pull request opened with `GITHUB_TOKEN` starts no checks, and the app is the only bypass of the rulesets that protect `main`, `release/*` and `v*` tags. The app cannot push a change to `.github/workflows/`, so such a change enters a release only through a pull request into it.
- **Runners.** GitHub-hosted only (`ubuntu-24.04`). A self-hosted runner on a public repository would run a stranger's code on your machine.
- **Logs are public.** Every secret is passed by name and read from `env:`; a bearer minted from it is masked before use. Maven reads the Exchange credential from the environment, never from a file in the repository.
- **Departures from github-actions GHAD-3.** The Maven release and the Exchange publication build in the job that holds the credential, because the plugins publish what they build. Those jobs restore no cache and run only on refs that reviewed pull requests or the release app change. The deploy itself builds nothing: `mule:deploy` deploys the version that Exchange holds.

## Setup

For each repository: `ANYPOINT_*` comes from the Anypoint organisation, `RELEASE_*` from the release app.

1. **Anypoint:** the environments (for example `TEST`, `UAT`, `PROD`) and a Connected App acting on its own behalf (client credentials) with Exchange Contributor and, in those environments, the Runtime Manager roles to deploy and read applications.
2. **The release app:** a GitHub App with Contents and Pull requests read and write and no webhook, installed on the repository. Keep its private key.
3. **Repository variables:** `ANYPOINT_ORG_ID`, `ANYPOINT_CLIENT_ID`, `RELEASE_APP_CLIENT_ID`.
4. **Environments:**

   | Environment | Deployment refs | Variable | Secrets | Reviewer |
   |---|---|---|---|---|
   | `release` | `main`; an application also `release/*` and tag `v*` | none | `ANYPOINT_CLIENT_SECRET`, `RELEASE_APP_PRIVATE_KEY` | none |
   | `test` (application) | `main` | `ANYPOINT_ENVIRONMENT` | `ANYPOINT_CLIENT_SECRET` | none |
   | `uat` (application) | `release/*`, tag `v*` | `ANYPOINT_ENVIRONMENT` | `ANYPOINT_CLIENT_SECRET` | none |
   | `prod` (application) | tag `v*` | `ANYPOINT_ENVIRONMENT` | `ANYPOINT_CLIENT_SECRET` | the owner |

5. **Rulesets:** `main` and `release/*` take changes only by pull request with the check `<caller job> / build`, and cannot be deleted or force-pushed; `v*` tags can be created only by the app. The release app is the bypass of all three.
6. **Merges:** squash only, with the pull request's title and description as the commit.
7. **Forks:** Settings → Actions → "Require approval for all external contributors".
8. **An application:** `deployment/test.yaml`, `uat.yaml` and `prod.yaml`, each with `applicationName`, `target`, `runtimeVersion`, `replicas`, `vCores` and `healthPath`, and the POM's `cloudhub2Deployment` reading the `deploy.*` properties the deploy passes (`deploy.environment`, `deploy.applicationName`, `deploy.target`, `deploy.runtimeVersion`, `deploy.replicas`, `deploy.vCores`, `deploy.clientId`, `deploy.clientSecret`).
9. **The first release after a migration:** Criar release with `base` set to the migration's commit on `main`, because the older tags lack these workflows.

## What a company adds

These workflows are the core of the delivery. A company usually adds:
- MUnit with its Enterprise credential (above);
- a delivery register of every deploy and release;
- the API Manager policies of each application, applied from a file at each deploy;
- the publication of API specs through Design Center;
- a lock that stops `main` and an open release from publishing the same SNAPSHOT.
