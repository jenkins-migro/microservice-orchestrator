# Jenkins to GitHub Actions Migration Report

## Repository
`microservice-orchestrator`

## Migration Date
2026-08-12

## Summary

The repository contained a single Jenkins declarative pipeline (`Jenkinsfile`) that built a
Node.js/React application and deployed the build output to Netlify. It has been converted to an
equivalent GitHub Actions workflow at `.github/workflows/ci-cd.yml`, and the original Jenkins
configuration has been archived (moved, not deleted) to `.github/ci-archive/Jenkinsfile`.

> Note: The original `Jenkinsfile` carried a header comment indicating it was sourced from
> `https://github.com/uchennaofodile/movie-land` (Educational Use) and used as a simple pipeline
> test case. This provenance note is preserved verbatim in the archived file.

## Source Pipeline Analysis

| Aspect | Detail |
|---|---|
| Pipeline type | Declarative (`pipeline { agent any ... }`) |
| Stages | `Build`, `Deploy` |
| Agent | `agent any` (no specific label or container) |
| Shared libraries | None used |
| Scripted/Groovy logic | None (pure declarative, only `sh` steps) |
| Credentials | One binding: `credentials("netlify-token")` mapped to `NETLIFY_AUTH_TOKEN` |
| Triggers | None declared in Jenkinsfile (assumed configured on the Jenkins job or webhook side) |

### Stage-by-stage mapping

| Jenkins stage | Jenkins steps | GitHub Actions equivalent |
|---|---|---|
| `Build` | `sh "npm install"`, `sh "npm run build"` | `build` job: `actions/checkout`, `actions/setup-node`, `npm install`, `npm run build`, then `actions/upload-artifact` to pass the `build/` directory to the deploy job |
| `Deploy` | env `NETLIFY_SITE_ID` (plain literal), env `NETLIFY_AUTH_TOKEN` (credential binding), `sh "npm install -g netlify-cli"`, `sh "netlify deploy --prod --dir=build --site=$NETLIFY_SITE_ID --auth=$NETLIFY_AUTH_TOKEN"` | `deploy` job (depends on `build`, gated to `push` events on `main`): `actions/download-artifact`, `actions/setup-node`, install `netlify-cli`, run `netlify deploy` using `vars.NETLIFY_SITE_ID` and `secrets.NETLIFY_AUTH_TOKEN` |

## Design Decisions

- Trigger strategy: The Jenkinsfile had no explicit trigger block, meaning triggering was
  handled outside the pipeline definition (for example, a Jenkins job configured with an SCM
  webhook). The workflow uses `push` (branch `main`), `pull_request` (branch `main`), and
  `workflow_dispatch` so pull requests get build validation without deploying, ordinary pushes to
  `main` build and deploy, and maintainers can trigger manual runs.
- Two-job split (build/deploy): Splitting the single linear pipeline into a `build` job and a
  `deploy` job (connected with `needs:` and an artifact hand-off) allows pull request builds to run
  and validate the app without ever touching deploy credentials, and allows the `deploy` job to be
  gated with an `if:` condition (push to `main` only) plus a GitHub Environment (`production`) for
  secret scoping and optional required reviewers or protection rules.
- Least privilege permissions: Top-level `permissions: contents: read`, reasserted at the job
  level. No write permissions are granted since the workflow does not push commits, tags, or
  releases.
- Pinned actions: All GitHub-authored actions (`actions/checkout`, `actions/setup-node`,
  `actions/upload-artifact`, `actions/download-artifact`) are pinned to full commit SHAs (with the
  released version noted in a trailing comment) per the migration guardrails, instead of floating
  tags, to prevent supply-chain tag-mutation attacks.
- Concurrency control: Added a `concurrency` group keyed on workflow and ref so superseded runs on
  the same branch are cancelled automatically, avoiding redundant or overlapping deployments.
- No third-party Netlify action: Rather than introducing an additional third-party marketplace
  action for Netlify deployment, the workflow preserves the original approach of installing
  `netlify-cli` via `npm install -g` and invoking `netlify deploy` directly, matching the
  Jenkinsfile steps as closely as possible while only changing the credential source. The CLI is
  pinned to major version 17 (`netlify-cli@17`) instead of an unpinned latest install, for build
  reproducibility; bump this pin deliberately when ready to adopt a newer major version.
- Secret vs. variable classification: Only the value that was wrapped in
  `credentials("netlify-token")` in Jenkins is treated as a GitHub secret (`NETLIFY_AUTH_TOKEN`),
  consistent with it being an authentication token. The literal `NETLIFY_SITE_ID` value
  (`classy-paletas-f45a67`) was a plain string in the Jenkinsfile, not a bound credential, so it is
  migrated to a GitHub Actions repository variable (`vars.NETLIFY_SITE_ID`) rather than a secret,
  keeping non-sensitive configuration out of the secrets store while still avoiding a hardcoded
  value in the workflow file.

## Required GitHub Configuration (Secrets & Variables)

Before this workflow can deploy successfully, the repository (or the `production` environment)
must be configured with:

| Name | Type | Purpose | Suggested value |
|---|---|---|---|
| `NETLIFY_AUTH_TOKEN` | Actions secret (repository or `production` environment) | Netlify personal access token, replaces Jenkins credential ID `netlify-token` | Generate from Netlify user settings, Applications, Personal access tokens |
| `NETLIFY_SITE_ID` | Actions variable (repository or `production` environment) | Target Netlify site identifier | `classy-paletas-f45a67` (value taken from the original Jenkinsfile) |

A GitHub Environment named `production` is referenced by the `deploy` job
(`environment: production`) so these values can optionally be scoped to that environment and
protected with required reviewers or wait timers if desired. If no such environment exists yet,
create it under Settings, Environments, or remove the `environment:` line if environment-level
protection is not desired.

## Validation

- YAML syntax validated with `python3 -c "import yaml; yaml.safe_load(...)"`, passed.
- `actionlint` was not available on the local runner, so actionlint validation was not run. No
  dependency was installed solely to lint this workflow.
- All action commit SHAs were independently verified to exist in their respective upstream
  GitHub repositories.
- CodeQL Actions analysis completed with zero alerts.

## Files Changed

- `Jenkinsfile`, removed from repository root (moved).
- `.github/workflows/ci-cd.yml`, new GitHub Actions workflow (added).
- `.github/ci-archive/Jenkinsfile`, archived original Jenkins pipeline (added).
- `.github/ci-archive/MIGRATION-README.md`, this report (added).

## Constraints / Out-of-scope for this session

The internal CI/CD migration knowledge base (organization `.github-private`,
resolved here as `jenkins-migro/.github-private`) could not be read: repository lookups, file
content requests, and issue or branch listings against that repository all returned
`404 Not Found` with the credentials available in this session, indicating the migration agent
GitHub App or token does not currently have read access to that internal repository. This
migration was therefore completed using the Jenkins-to-GitHub-Actions mapping knowledge, security
guardrails (SHA-pinning, least-privilege permissions, secret/variable classification, credential
handling), and report structure built into this agent own operating instructions, which mirror the
documented knowledge-base process. If knowledge-base access is restored, it would be worth
re-checking this migration against the authoritative `knowledge/actions-mapping/jenkins.md`,
`knowledge/patterns/jenkins/*.md`, and `knowledge/report-template/jenkins.md` documents for any
organization-specific conventions not captured here.

The repository currently contains no application source files, `package.json`, or dependency lock
file—only the migration configuration and README. Therefore, the `npm install` and `npm run build`
steps could not be executed locally; they preserve the original Jenkins commands and require the
application files to be present when the workflow runs. Node.js 20 is an explicit GitHub Actions
runner assumption because the Jenkinsfile used `agent any` and did not declare its Node.js version.
