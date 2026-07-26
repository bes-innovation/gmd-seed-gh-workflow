# Project provisioning

Drop a `<repo-name>.yaml` file in this folder and merge it to `main` —
`.github/workflows/gmd-seed-bootstrap-project-wfl.yml` picks up any changed
file here automatically and provisions the repo: creates it from
[`salesforce-project-template`](https://github.com/bes-innovation/salesforce-project-template),
creates the listed GitHub Environments, and applies branch protection.

This runs **automatically on push to `main`** — no manual trigger step.
Review the config carefully before merging; a typo in `repoName` creates
a repo with that typo'd name (repo creation is otherwise idempotent —
re-running against an existing repo just skips creation and re-applies
environments/branch protection).

See `_bootstrap-project-definition.yaml` for the schema (files starting
with `_` are ignored by the bootstrap workflow — reference only, never
provisioned).

## Schema

```yaml
repoName: my-new-project        # required -- becomes bes-innovation/<repoName>
description: "Short description" # optional
visibility: public               # optional, "public" or "private" (default: public)

environments:                    # optional, list of GitHub Environments to create
  - name: qa
    requiredReviewers: false
  - name: uat
    requiredReviewers: false
  - name: prod
    requiredReviewers: true       # NOTE: creates the environment, but actual reviewer
                                   # assignment still needs a person's/team's numeric
                                   # GitHub ID -- not automated yet, see "Manual
                                   # follow-ups" below.

branchProtection:                 # optional, list of branch rules to apply
  - branch: development
    requiredStatusChecks: ["validate"]
    requiredApprovingReviewCount: 1
  - branch: main
    requiredStatusChecks: []
    requiredApprovingReviewCount: 1
```

## Manual follow-ups (not automated)

The bootstrap workflow provisions the repo shell, environments, and
branch protection. It does **not** and cannot automate:

- Setting `SFDX_USERNAME` / `SFDX_CLIENT_ID` / `SFDX_INSTANCE_URL` /
  `SFDX_JWT_KEY` / `TEAMS_WEBHOOK_URL` secrets on each environment
  (real per-org credentials — see the template repo's README for the
  full checklist).
- Assigning actual required reviewers to an environment marked
  `requiredReviewers: true` (Settings → Environments → `<name>` →
  Required reviewers, needs a specific person/team).

The workflow run's summary lists these as next steps every time it
provisions a project.

## Prerequisite

The bootstrap workflow needs an `ORG_ADMIN_TOKEN` secret on this repo --
a PAT with `repo` + org repo-creation rights, since the default
`GITHUB_TOKEN` can't create new repositories in the org. Not needed to
edit configs here, only for the workflow to actually run.
