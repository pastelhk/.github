# pastelhk/.github

​
Shared GitHub Actions, reusable workflows and starter templates for the Pastel
organisations (`pastelhk`).
​

> **This repository is public.** Never commit secrets, internal hostnames or
> client names. Hosts are referenced through `vars.*`, never as literals.
> ​

## How to use

​
Reusable workflows live under `.github/workflows` and are referenced with `uses:`:
​

```yaml
jobs:
  scan:
    uses: pastelhk/.github/.github/workflows/code-scan.yml@v1
    permissions:
      contents: read
      checks: write
      pull-requests: write
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      PRIVATE_NPM_REGISTRY_PASSWORD: ${{ secrets.PRIVATE_NPM_REGISTRY_PASSWORD }}
```

​
Composite actions live under `.github/actions`:
​

```yaml
- uses: pastelhk/.github/.github/actions/npm-init@v1
  with:
    restore-packages: true
```

​
Starter templates are in `workflow-templates/` and are copied into a repository
once. They are **not** inherited and do **not** receive updates.
​

### Conventions that apply to every caller

- **`permissions:` is mandatory on the calling job.** A called workflow can only
  _downgrade_ the token it receives, never elevate it. Omit the block and the job
  inherits the repository default, which is read-only in hardened organisations —
  the PR comment fails silently and the test reporter reds the job.
- **Secrets are passed explicitly, not inherited.** `secrets: inherit` would hand
  this public repository every secret the caller can see. It also only works
  within a single organisation, so it would break `pasteltech` callers.
- **Do not add your own `concurrency:` block.** Cancellation is owned by the
  called workflow. Setting it in both places causes unpredictable cancellations.
- **`@v1` is a moving tag**, retagged on each release, so callers pick up fixes
  automatically. Pin to a commit SHA if a repository needs immutability.
- **Keep the job id stable.** Branch protection matches on
  `<job-id> / <job-name>`, so the stubs below deliberately fix the job id.
  ​

## Reusable workflows

### `code-scan.yml`

​
Runs the standard repository health checks: lock file, package lint, format,
types, lint, tests, audit, hoisting and SonarQube. Posts a single sticky PR
comment that is updated in place and keeps a short run history.
​
**Required status check name:** `scan / Code Scan` (with the stub below).
​
**Inputs**
​

| Input          | Type    | Default | Description                       |
| -------------- | ------- | ------- | --------------------------------- |
| `run-packages` | boolean | `true`  | Run `npm run check:packages`      |
| `run-format`   | boolean | `true`  | Run `npm run check:format`        |
| `run-types`    | boolean | `true`  | Run `npm run check:types`         |
| `run-lint`     | boolean | `true`  | Run `npm run check:lint`          |
| `run-tests`    | boolean | `true`  | Run `npm run check:test`          |
| `run-audit`    | boolean | `true`  | Run `npm audit --omit=dev`        |
| `run-sonar`    | boolean | `true`  | Run SonarQube scan                |
| `run-hoisting` | boolean | `true`  | Verify workspace package hoisting |
| `pr-comment`   | boolean | `true`  | Post/update a sticky PR comment   |
| `timeout`      | number  | `12`    | Job timeout in minutes            |

**Secrets and variables**

| Name                            | Type     | Required        | Description                                       |
| ------------------------------- | -------- | --------------- | ------------------------------------------------- |
| `PRIVATE_NPM_REGISTRY_PASSWORD` | secret   | If private deps | Password or token for the private npm registry    |
| `SONAR_TOKEN`                   | secret   | If `run-sonar`  | SonarQube token                                   |
| `PRIVATE_NPM_REGISTRY_URL`      | variable | No              | Private npm registry host (org level)             |
| `PRIVATE_NPM_REGISTRY_USERNAME` | variable | No              | Private npm registry username (org level)         |
| `SONAR_HOST_URL`                | variable | No              | SonarQube host URL (org level)                    |
| `SONAR_PROJECT_KEY`             | variable | If `run-sonar`  | SonarQube project key — **repository level only** |

SonarQube and the private registry steps are skipped when the corresponding
variables are not set.

> If a repository has private dependencies but no
> `PRIVATE_NPM_REGISTRY_PASSWORD`, `npm install` fails with an **E404** that
> looks like a missing package rather than an auth error. Check the secret first.
> ​
> **Caller stub**
> ​

```yaml
name: Code Scan
​
on:
  pull_request:
  push:
    branches: [develop]
​
jobs:
  scan:
    uses: pastelhk/.github/.github/workflows/code-scan.yml@v1
    permissions:
      contents: read # checkout + Sonar blame (fetch-depth: 0)
      checks: write # dorny/test-reporter check run
      pull-requests: write # sticky PR comment (drop if pr-comment: false)
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      PRIVATE_NPM_REGISTRY_PASSWORD: ${{ secrets.PRIVATE_NPM_REGISTRY_PASSWORD }}
```

​
On `push` runs there is no pull request, so the sticky comment is skipped. The
check still appears under the same name. This is expected, not a failure.
​
**Least-privilege variants**
​

| Configuration       | Permissions needed                                        |
| ------------------- | --------------------------------------------------------- |
| Default             | `contents: read`, `checks: write`, `pull-requests: write` |
| `pr-comment: false` | drop `pull-requests: write`                               |
| `run-tests: false`  | drop `checks: write`                                      |
| Both disabled       | `contents: read` only                                     |

Fork pull requests receive a read-only token regardless of what the caller  
declares, so the comment and check-run steps degrade on external contributions.

### `dependabot-auto-merge.yml`

​
Approves and enables auto-merge for Dependabot pull requests.
​
**Inputs**
​

| Input         | Type    | Default | Description                           |
| ------------- | ------- | ------- | ------------------------------------- |
| `allow-minor` | boolean | `false` | Also auto-merge minor version updates |

**Caller stub**

```yaml
name: Dependabot auto-merge
​
on:
  pull_request_target:
​
jobs:
  auto-merge:
    # REQUIRED. pull_request_target runs with a fully privileged token in the
    # base repository context. Without this gate, any pull request from any
    # author triggers a privileged run.
    if: github.actor == 'dependabot[bot]'
    uses: pastelhk/.github/.github/workflows/dependabot-auto-merge.yml@v1
    permissions:
      contents: write
      pull-requests: write
```

​
Prerequisites that fail silently if missed:
​

- **Allow auto-merge** must be enabled in the repository settings, otherwise
  enabling auto-merge is a no-op.
- A `GITHUB_TOKEN` approval does **not** satisfy a branch protection rule that
  requires a review from a person or from CODEOWNERS. The PR will sit waiting.
- Never add `secrets: inherit` to a `pull_request_target` workflow.
  ​

### `publish.yml`

​
Publishes packages and pushes any generated tags and versions.
​
**Inputs**

| Input     | Type   | Default         | Description                                                  |
| --------- | ------ | --------------- | ------------------------------------------------------------ |
| `args`    | string | `""`            | Extra arguments passed to `npm run publish`                  |
| `runs-on` | string | `ubuntu-latest` | Runner label. Use `macos-latest` only when Xcode is required |

**Secrets and variables**

| Name                                    | Type     | Required | Description                                  |
| --------------------------------------- | -------- | -------- | -------------------------------------------- |
| `PUBLISH_APP_PRIVATE_KEY`               | secret   | Yes      | GitHub App private key for publishing        |
| `PUBLISH_APP_ID`                        | variable | Yes      | GitHub App ID for publishing                 |
| `PRIVATE_NPM_REGISTRY_PASSWORD`         | secret   | No       | Private npm registry password                |
| `PRIVATE_NPM_REGISTRY_PUBLISH_USERNAME` | variable | No       | Private npm registry username for publishing |
| `PRIVATE_NPM_REGISTRY_URL`              | variable | No       | Private npm registry host                    |

**Caller stub**

```yaml
name: Publish
​
on:
  workflow_dispatch:
    inputs:
      args:
        description: 'Publish arguments'
        required: false
        default: ''
​
jobs:
  publish:
    uses: pastelhk/.github/.github/workflows/publish.yml@v1
    permissions:
      contents: write
    with:
      args: ${{ inputs.args }}
    secrets:
      PUBLISH_APP_PRIVATE_KEY: ${{ secrets.PUBLISH_APP_PRIVATE_KEY }}
      PRIVATE_NPM_REGISTRY_PASSWORD: ${{ secrets.PRIVATE_NPM_REGISTRY_PASSWORD }}
```

## Composite actions

### `npm-init`

Sets up Node from `.nvmrc`, enables Corepack when the project declares
`packageManager`, writes registry authentication and optionally installs
dependencies.
​

| Input               | Default  | Description                                                       |
| ------------------- | -------- | ----------------------------------------------------------------- |
| `shell`             | `bash`   | Shell used for the action's `run` steps                           |
| `restore-packages`  | `'true'` | Run `npm ci` after setup                                          |
| `registry-url`      | —        | Private registry URL. Scheme optional; auth is skipped when empty |
| `registry-username` | —        | Private registry username                                         |
| `registry-password` | —        | Private registry password or token                                |

**Contract to be aware of:** the action writes `.npmrc` to `$RUNNER_TEMP` and
exports `NPM_CONFIG_USERCONFIG` into `$GITHUB_ENV`. Registry auth therefore
applies to **every subsequent step in the calling job**, not just steps inside
the action, and the repository working tree is never modified. Do not "simplify"
this by writing `.npmrc` into the checkout — that reintroduces false positives in
the lock-file drift check.  
​

### `scan-comment`

Creates and updates the sticky pull request comment used by `code-scan.yml`.
Not intended to be called directly.

## Starter workflow templates

​
Files in `workflow-templates/` appear under **Actions → New workflow** in
repositories across the organisation. Selecting one copies it into that
repository's `.github/workflows/` directory. There is no link back to this
repository afterwards, so later fixes here do **not** reach copies.

> **Limitation:** workflow templates created by an organisation can only be used
> in **public** repositories unless the organisation is on GitHub Enterprise
> Cloud. Most client repositories are private, so the templates may not appear
> there. This restriction does not affect reusable workflows, which work from a
> public `.github` into private callers on any plan.
> ​

- `code-scan.yml` — thin caller stub for the reusable code scan.
- `pr-labeler.yml` — label PRs by path and monorepo package. The target
  repository must also contain `.github/labeler.yml`, because `actions/labeler`
  reads its configuration from the repository it runs in.
- `pr-dependency-check.yml` — check PR descriptions for linked dependent PRs.
  ​
  Each template requires a matching `<name>.properties.json` in the same directory.
  ​

## Config templates

These are **not** workflows and must not be copied into `.github/workflows/`.

| File                       | Copy to                  | Purpose                                                 |
| -------------------------- | ------------------------ | ------------------------------------------------------- |
| `templates/dependabot.yml` | `.github/dependabot.yml` | Dependabot config for npm, GitHub Actions and Terraform |
| ​                          |

## Organisation configuration

​
**Actions variables (`vars`) — organisation level**
​
`PRIVATE_NPM_REGISTRY_URL`, `PRIVATE_NPM_REGISTRY_USERNAME`,
`PRIVATE_NPM_REGISTRY_PUBLISH_USERNAME`, `SONAR_HOST_URL`, `PUBLISH_APP_ID`
​
**Actions variables (`vars`) — repository level**
​
`SONAR_PROJECT_KEY` — must be unique per repository. Setting this at
organisation level makes every repository report into a single SonarQube
project, silently merging coverage and quality gates across clients.
​
**Actions secrets**
​
`PRIVATE_NPM_REGISTRY_PASSWORD`, `SONAR_TOKEN`, `PUBLISH_APP_PRIVATE_KEY`
​
**Dependabot secrets**
​
`PRIVATE_NPM_REGISTRY_USERNAME` and `PRIVATE_NPM_REGISTRY_PASSWORD` must also be
configured as **Dependabot secrets**. Dependabot does not read Actions secrets.
​
Both `pastelhk` and `pasteltech` need the same variables and secrets.
​

## Versioning

- `v1` is a moving major tag; patches and backwards-compatible changes are
  retagged onto it.
- Breaking changes to inputs, secrets or the required check name get a new major
  tag and a note in this README.
- Use `@develop` only while smoke-testing a change.
