# gmd-seed-gh-workflow

Shared, reusable GitHub Actions for Salesforce CI/CD. Consumer repos call these
instead of re-implementing auth/deploy/retrieve logic in their own workflows.

## Structure

- `actions/` — composite actions, the reusable building blocks (auth, CLI setup,
  deploy, retrieve). Can be called on their own, outside the full workflows below.
- `.github/workflows/` — reusable workflows (`workflow_call`) that orchestrate the
  actions above into complete jobs. Must stay flat — GitHub only discovers
  `workflow_call` workflows directly inside `.github/workflows/`, not in subfolders.

## Target architecture

`gmd-seed-wfl-sf-validate.yml` and `gmd-seed-wfl-sf-deploy.yml` follow the
pipeline design in `documentation/GDM-DevOps-CICD.pdf` (GDM, 2026-06-11):
JWT Bearer Flow auth, full `force-app` source deploys (not manifest/delta),
a PMD quality gate, and a validate→quick-deploy pattern for PROD. **This is
the target design, not what's live today** — production deploys currently
go out via Salesforce Change Sets; this pipeline is the replacement being
built.

Environments per GDM: **QA** (sandbox, validate PR + deploy on merge to
`develop`), **UAT** (sandbox Full/Partial, homologação, manual dispatch),
**PROD** (production, quick-deploy + Required reviewers), **HOTFIX**
(sandbox clone of PROD, Capgemini patches). Branch model: `main` (immutable
PROD reference) / `develop` (Cloud Gaia integration) / `release/*` (cut from
develop) / `feature/*` (per dev, from develop) / `hotfix/*` (Capgemini, from
main).

## Available workflows

Every workflow requires an `environment` input naming a [GitHub
Environment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
configured in the **calling** repo (e.g. `qa`, `uat`, `prod`, `hotfix`).
This is also where you attach approval/branch-protection rules per
environment (e.g. Required reviewers on `prod`).

| Workflow | Purpose |
|---|---|
| `gmd-seed-wfl-sf-validate.yml` | PMD scan + check-only deploy (`sf project deploy validate`) against a target org, no changes made |
| `gmd-seed-wfl-sf-deploy.yml` | Deploy `force-app` into a target org; optionally validate-then-quick-deploy instead of re-running tests |
| `gmd-seed-wfl-backpromote.yml` | Opens a PR from `main` back into `develop` after a PROD deploy, so hotfixes sync back into the feature line |
| `gmd-seed-wfl-sf-retrieve.yml` | Retrieve metadata from an org, upload as a build artifact for review (baseline bootstrap / drift detection, not part of the promotion flow) |
| `gmd-seed-wfl-pr-title-check.yml` | Validates PR title format (default: ticket-number prefixed, e.g. `[PROJ-123] ...`) |

`gmd-seed-wfl-sf-validate.yml` and `gmd-seed-wfl-sf-deploy.yml` authenticate
via `actions/sf-auth-jwt` (JWT Bearer Flow). The original SFDX Auth URL
option is still available as `actions/sf-auth` for workflows that haven't
moved to JWT yet — see "Authenticating" below.

Callers must pass `secrets: inherit` so the environment's secrets resolve
automatically based on the `environment` input, instead of every caller
having to explicitly map differently-named secrets per env.

### Approval gate (deploy only)

`gmd-seed-wfl-sf-deploy.yml` doesn't implement approval in YAML — it relies on
the environment's own **Required reviewers** protection rule (Settings →
Environments → `<environment>` → Deployment protection rules). Since the job
declares `environment: ${{ inputs.environment }}`, GitHub pauses the entire
job — before any step, including the "starting" notification — until someone
approves. Enable this per environment where you want a human gate (e.g.
`production`, not necessarily `staging`).

### Authenticating

Two options, both available as standalone actions:

| Action | Method | Secrets |
|---|---|---|
| `actions/sf-auth-jwt` | JWT Bearer Flow (Connected App + X.509 cert), no interactive login | `SFDX_USERNAME`, `SFDX_CLIENT_ID`, `SFDX_INSTANCE_URL` (Environment secrets, one set per env) + `SFDX_JWT_KEY` (Repository secret, or Environment secret if the Connected App differs per org) |
| `actions/sf-auth` | SFDX Auth URL | `SF_AUTH_URL` (same secret name, different value per environment) |

`gmd-seed-wfl-sf-validate.yml` and `gmd-seed-wfl-sf-deploy.yml` are wired to
`sf-auth-jwt`, matching the GDM target design. If you have a caller repo
still set up with an `SF_AUTH_URL` secret, either keep it on the old
workflow revision or swap the auth step to `sf-auth` in a fork/copy of
these workflows — both actions stay in this repo so nothing is lost.

### Quality gate (PMD)

`gmd-seed-wfl-sf-validate.yml` runs `actions/sf-code-analyzer` before
authenticating. GDM's real pipeline runs this with `continue-on-error: true`
today — it reports violations but doesn't block the PR; `check-only` deploy
validation is the actual hard gate. `severity-threshold` defaults to `5`
(PR/QA); pass `3` for the stricter UAT gate. Quick win from the GDM doc:
require this check in branch protection and set `pmd-continue-on-error:
false` once the team is ready to block on it.

### Quick-deploy (PROD)

`gmd-seed-wfl-sf-deploy.yml` takes a `use-quick-deploy` input. When `true`,
it runs `sf project deploy validate` (with tests) and then
`sf project deploy quick` on the resulting job id, instead of running tests
twice. Salesforce's quick-deploy window is 24h from the validate. This is
the GDM-recommended pattern for PROD; leave it `false` (default) for QA/UAT,
which deploy directly.

### Teams notifications (deploy only)

`gmd-seed-wfl-sf-deploy.yml` posts to Teams on start, success, and failure via
`actions/teams-notify`, using a `TEAMS_WEBHOOK_URL` secret (same
per-environment scoping as the other secrets). Skipped entirely if that
secret isn't set on the environment. This expects a **Teams "Workflows"
app** incoming webhook (Adaptive Card payload) — the current Teams-native
webhook mechanism. If your Teams channel instead uses a legacy "Office 365
Connector" webhook, the payload format differs and `actions/teams-notify`
would need a small adjustment (`MessageCard` schema instead of Adaptive
Card).

Each card shows a status pill + colored title, a fact list (environment,
branch, the pull request that triggered the merge, actor, repository,
Salesforce-reported start/completion times), component and Apex test
counts, and a link to the Salesforce deploy status page. On failure, the
actual Salesforce error is surfaced inline rather than requiring a log
dive. The PR reference is looked up automatically from the triggering
commit — no caller input needed, empty if the push wasn't a PR merge.

### Example: validate PR on QA

```yaml
name: CI

on:
  pull_request:
    branches: [develop]
    paths: ['force-app/**']

jobs:
  validate:
    uses: bes-innovation/gmd-seed-gh-workflow/.github/workflows/gmd-seed-wfl-sf-validate.yml@v1
    with:
      environment: qa
      target-org: qa
      severity-threshold: "5"
    secrets: inherit
```

### Example: deploy to QA on merge

```yaml
name: Deploy to QA

on:
  push:
    branches: [develop]
    paths: ['force-app/**']

jobs:
  deploy:
    permissions:
      contents: read
      pull-requests: read  # needed for the reusable workflow's PR lookup (Teams card)
    uses: bes-innovation/gmd-seed-gh-workflow/.github/workflows/gmd-seed-wfl-sf-deploy.yml@v1
    with:
      environment: qa
      target-org: qa
    secrets: inherit
```

> A caller job's default token permissions can't grant more than the
> organization's default Actions token policy allows. If that's set to
> "Read repository contents" only, `gmd-seed-wfl-sf-deploy.yml` will fail
> to resolve with `pull-requests: none` unless the caller job declares
> `permissions:` explicitly, as above.

### Example: deploy to PROD with quick-deploy + backpromotion

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
    paths: ['force-app/**']

jobs:
  deploy-prod:
    permissions:
      contents: read
      pull-requests: read
    uses: bes-innovation/gmd-seed-gh-workflow/.github/workflows/gmd-seed-wfl-sf-deploy.yml@v1
    with:
      environment: prod
      target-org: prod
      use-quick-deploy: true
    secrets: inherit

  backpromote:
    needs: deploy-prod
    permissions:
      contents: write
      pull-requests: write
    uses: bes-innovation/gmd-seed-gh-workflow/.github/workflows/gmd-seed-wfl-backpromote.yml@v1
```

### Example: PR title check

```yaml
name: CI

on:
  pull_request:
    types: [opened, edited, synchronize]
    branches: [development, staging, main]

jobs:
  pr-title:
    uses: bes-innovation/gmd-seed-gh-workflow/.github/workflows/gmd-seed-wfl-pr-title-check.yml@v1
    with:
      pr_title_regexp: '^(\[PROJ-\d+\])+ (.*)[\n\r]*$'   # swap PROJ for the real ticket-system key(s) once confirmed
```

No secrets required — this one doesn't touch Salesforce or need repo checkout,
it just validates the title string from the PR event.

## Available actions

| Action | Purpose |
|---|---|
| `actions/sf-cli-setup` | Installs the Salesforce CLI on the runner |
| `actions/sf-auth-jwt` | Authenticates via JWT Bearer Flow (Connected App + certificate) — GDM target |
| `actions/sf-auth` | Authenticates via an SFDX Auth URL secret |
| `actions/sf-code-analyzer` | Installs Salesforce Code Analyzer and runs a PMD scan with a severity gate |
| `actions/sf-deploy-metadata` | Deploys or validates the `force-app` source against a target org |
| `actions/sf-quick-deploy` | Quick-deploys a previously validated job id (skips re-running tests) |
| `actions/sf-retrieve-metadata` | Retrieves a manifest's metadata from a target org |
| `actions/teams-notify` | Posts a start/success/failure status card to a Teams channel via webhook |

Use an action directly if you don't need the full orchestrated workflow, e.g.:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: bes-innovation/gmd-seed-gh-workflow/actions/sf-cli-setup@v1
  - uses: bes-innovation/gmd-seed-gh-workflow/actions/sf-auth-jwt@v1
    with:
      username: ${{ secrets.SFDX_USERNAME }}
      client-id: ${{ secrets.SFDX_CLIENT_ID }}
      instance-url: ${{ secrets.SFDX_INSTANCE_URL }}
      jwt-key: ${{ secrets.SFDX_JWT_KEY }}
```

## Versioning

Consumers pin a tag (`@v1`), not a branch. Bump the tag when making a breaking
change to an action's inputs or a workflow's required secrets; patch/minor fixes
can move the existing major tag forward.

## Setting up JWT Bearer Flow secrets

1. Generate a self-signed certificate (GDM recommends rotating it periodically):
   ```
   openssl req -x509 -newkey rsa:2048 -keyout server.key -out server.crt -days 365 -nodes
   ```
2. Upload `server.crt` to a Connected App in the target org (Setup → App
   Manager → New Connected App → Enable OAuth Settings → Use digital
   signatures), with the `api` OAuth scope at minimum, and pre-authorize a
   dedicated integration user (not a personal login) via a Permission Set.
3. Populate secrets per GitHub Environment (QA, UAT, PROD, HOTFIX):
   - `SFDX_USERNAME` — the integration user's username
   - `SFDX_CLIENT_ID` — the Connected App's Consumer Key
   - `SFDX_INSTANCE_URL` — `https://test.salesforce.com` for sandboxes, `https://login.salesforce.com` for PROD
4. Populate `SFDX_JWT_KEY` (contents of `server.key`) as a **Repository**
   secret shared across environments, or as an **Environment** secret if
   each org has its own Connected App/cert.
5. Delete `server.key`/`server.crt` locally once uploaded — never commit them.
6. Enable **Required reviewers** on the `prod` Environment (Settings →
   Environments → prod → Deployment protection rules).

## Generating an SFDX Auth URL secret

Only needed if you're using the `sf-auth` (non-JWT) action:

```
sf org auth show-sfdx-auth-url --target-org <alias> --no-prompt
```

Store the output as a repo (or environment) secret — never commit it, never
print it in a workflow log.
