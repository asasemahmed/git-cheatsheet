# git-cheatsheet

A single page with two parts: the git and GitHub CLI commands I use most, and a complete illustrated guide to GitHub Actions. All diagrams are Mermaid and render on GitHub.

## Contents

**Part 1: Git and GitHub CLI**
- [Where your changes live](#where-your-changes-live)
- [Everyday git](#everyday-git)
- [Undo things](#undo-things)
- [Stash](#stash)
- [GitHub CLI](#github-cli)

**Part 2: GitHub Actions**
1. [The big picture](#1-the-big-picture)
2. [Anatomy of a workflow file](#2-anatomy-of-a-workflow-file)
3. [Events (triggers)](#3-events-triggers)
4. [Jobs](#4-jobs)
5. [Steps](#5-steps)
6. [Expressions and contexts](#6-expressions-and-contexts)
7. [Passing data around](#7-passing-data-around)
8. [Secrets, variables and environments](#8-secrets-variables-and-environments)
9. [Permissions and GITHUB_TOKEN](#9-permissions-and-github_token)
10. [Matrix builds](#10-matrix-builds)
11. [Caching and artifacts](#11-caching-and-artifacts)
12. [Reusable workflows](#12-reusable-workflows)
13. [Custom actions](#13-custom-actions)
14. [Runners](#14-runners)
15. [Concurrency and cancellation](#15-concurrency-and-cancellation)
16. [Service containers](#16-service-containers)
17. [Workflow commands](#17-workflow-commands)
18. [Security best practices](#18-security-best-practices)
19. [Debugging](#19-debugging)
20. [Limits worth knowing](#20-limits-worth-knowing)
21. [Frequently used actions](#21-frequently-used-actions)
22. [Quick reference](#22-quick-reference)
23. [Complete example workflows](#23-complete-example-workflows)

---

## Part 1: Git and GitHub CLI

### Where your changes live

Every git command moves changes between four places.

```mermaid
flowchart LR
    W[Working directory<br/>files you edit] -- "git add" --> S[Staging area<br/>next commit]
    S -- "git commit" --> L[Local repository<br/>your history]
    L -- "git push" --> R[(Remote<br/>GitHub)]
    R -- "git fetch" --> L
    R -- "git pull<br/>fetch + merge/rebase" --> W
    L -- "git switch / restore" --> W
    S -- "git restore --staged" --> W
```

A typical branch and pull request flow:

```mermaid
gitGraph
    commit id: "main"
    branch feature
    checkout feature
    commit id: "work"
    commit id: "more work"
    checkout main
    merge feature id: "PR merged"
    commit id: "next"
```

### Everyday git

```bash
git status                  # what changed
git switch -c my-branch     # create and switch to a branch
git add -p                  # stage changes interactively
git commit -m "message"     # commit staged changes
git commit --amend          # edit the last commit
git pull --rebase           # update without a merge commit
git log --oneline --graph   # compact history
```

### Undo things

```bash
git restore file.txt            # discard unstaged changes
git restore --staged file.txt   # unstage a file
git reset --soft HEAD~1         # undo last commit, keep changes staged
git revert <commit>             # new commit that undoes a commit
git reflog                      # find lost commits
```

### Stash

```bash
git stash push -m "wip"   # save work in progress
git stash list
git stash pop             # restore and drop the latest stash
```

### GitHub CLI

```bash
gh repo clone owner/repo
gh pr create --fill
gh pr checkout 123
gh pr merge --squash
gh issue list
```

---

## Part 2: GitHub Actions

GitHub Actions is GitHub's built-in automation platform. You describe what should happen (build, test, deploy, label an issue, publish a package) in a YAML file, and GitHub runs it on a virtual machine whenever something happens in your repository.

### 1. The big picture

Six terms cover almost everything:

| Term | What it is |
|---|---|
| **Workflow** | One YAML file in `.github/workflows/`. A repo can have many. |
| **Event** | Something that happens and starts a workflow (a push, a PR, a schedule, a button click). |
| **Job** | A group of steps that runs on one runner. Jobs run in parallel unless you chain them with `needs`. |
| **Step** | One task inside a job: either a shell command (`run`) or a reusable action (`uses`). |
| **Action** | A reusable unit of automation, like `actions/checkout`. |
| **Runner** | The machine that executes a job (GitHub-hosted VM or your own server). |

```mermaid
flowchart LR
    E[Event<br/>push, pull_request,<br/>schedule, ...] --> W[Workflow<br/>.github/workflows/ci.yml]
    W --> J1[Job: lint]
    W --> J2[Job: test]
    J1 --> J3[Job: deploy]
    J2 --> J3
    J1 -.runs on.-> R1[(Runner 1)]
    J2 -.runs on.-> R2[(Runner 2)]
    J3 -.runs on.-> R3[(Runner 3)]
```

`lint` and `test` run in parallel on separate runners. `deploy` waits for both because of `needs: [lint, test]`.

#### What happens inside one job

```mermaid
sequenceDiagram
    participant GH as GitHub
    participant R as Runner
    GH->>R: Assign job
    R->>R: Set up job (download actions)
    R->>R: Step 1: actions/checkout
    R->>R: Step 2: setup toolchain
    R->>R: Step 3: run tests
    R->>R: Post steps (cache save, cleanup)
    R-->>GH: Report result (success / failure / cancelled)
```

Steps in a job run in order, share the same filesystem, and stop at the first failure unless you say otherwise.


### 2. Anatomy of a workflow file

File location: `.github/workflows/<name>.yml` (or `.yaml`).

```yaml
name: CI                        # shown in the Actions tab

on:                             # events that trigger the workflow
  push:
    branches: [main]
  pull_request:

permissions:                    # limits what GITHUB_TOKEN may do
  contents: read

env:                            # variables for every job
  NODE_ENV: test

concurrency:                    # cancel superseded runs
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:                         # job id
    name: Unit tests            # display name
    runs-on: ubuntu-latest      # runner
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm test
```

Top-level keys:

| Key | Purpose |
|---|---|
| `name` | Display name of the workflow. |
| `run-name` | Display name of each run. Supports expressions, e.g. `Deploy by @${{ github.actor }}`. |
| `on` | The events that trigger it. Required. |
| `permissions` | Default token permissions for all jobs. |
| `env` | Environment variables for all jobs. |
| `defaults` | Default `shell` and `working-directory` for `run` steps. |
| `concurrency` | Limit or cancel parallel runs. |
| `jobs` | The jobs. Required. |


### 3. Events (triggers)

The `on:` key decides when a workflow starts. You can list one event, several, or add filters.

```yaml
on: push                        # single event
on: [push, pull_request]        # several events
on:                             # events with options
  push:
    branches: [main, "release/**"]
```

#### All events

**Code and branches**

| Event | Fires when |
|---|---|
| `push` | Commits or tags are pushed. |
| `pull_request` | A PR is opened, synchronized (new commits), reopened, and more. Runs on the PR merge commit with read-only token for forks. |
| `pull_request_target` | Like `pull_request` but runs in the context of the base branch with a write token and secrets. Dangerous with untrusted code, see [security](#18-security-best-practices). |
| `pull_request_review` | A PR review is submitted, edited or dismissed. |
| `pull_request_review_comment` | A comment on a PR diff changes. |
| `create` | A branch or tag is created. |
| `delete` | A branch or tag is deleted. |
| `fork` | Someone forks the repo. |
| `merge_group` | A PR is added to a merge queue. |
| `branch_protection_rule` | A branch protection rule changes. |

**Issues and discussions**

| Event | Fires when |
|---|---|
| `issues` | An issue is opened, edited, closed, labeled, assigned, etc. |
| `issue_comment` | A comment on an issue **or a PR** is created, edited or deleted. |
| `discussion` | A discussion is created, edited, answered, etc. |
| `discussion_comment` | A discussion comment changes. |
| `label` | A label is created, edited or deleted. |
| `milestone` | A milestone changes. |

**Releases, packages and deployments**

| Event | Fires when |
|---|---|
| `release` | A release is created, published, edited, released, etc. |
| `registry_package` | A package is published or updated in GitHub Packages. |
| `deployment` | A deployment is created. |
| `deployment_status` | A deployment's status changes. |
| `page_build` | A GitHub Pages build runs. |

**Checks and statuses**

| Event | Fires when |
|---|---|
| `check_run` | A check run is created, completed, rerequested. |
| `check_suite` | A check suite completes. |
| `status` | A commit status changes. |

**Repository**

| Event | Fires when |
|---|---|
| `public` | A private repo is made public. |
| `watch` | Someone stars the repo. |
| `gollum` | A wiki page is created or updated. |

**Manual, scheduled and chained**

| Event | Fires when |
|---|---|
| `workflow_dispatch` | You click "Run workflow" or call the API. Supports inputs. |
| `repository_dispatch` | An external system calls the REST API with a custom event type. |
| `schedule` | A cron time is reached. |
| `workflow_run` | Another workflow requested, in progress or completed. |
| `workflow_call` | Another workflow calls this one as a reusable workflow. |

#### Activity types

Many events have sub-types. Without `types`, the default types apply (for `pull_request`: `opened`, `synchronize`, `reopened`).

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review, labeled]
  issues:
    types: [opened, labeled]
  release:
    types: [published]
```

#### Filters

```yaml
on:
  push:
    branches:
      - main
      - "release/**"          # glob patterns
      - "!release/legacy"     # exclude
    tags:
      - "v*.*.*"
    paths:                    # only when these files change
      - "src/**"
      - "package.json"
    paths-ignore:             # or: skip when only these change
      - "**.md"
```

Rules:
- You cannot mix `branches` with `branches-ignore`, or `paths` with `paths-ignore`, for the same event. Use `!` negation inside one list instead.
- A `push` with `tags` and `branches` filters fires if either matches.
- If a workflow is skipped by a path filter, a required status check from it stays "pending". Use a different approach for required checks.

#### Schedule (cron)

```yaml
on:
  schedule:
    - cron: "30 5 * * 1-5"    # 05:30 UTC, Monday to Friday
```

```
┌───────────── minute (0-59)
│ ┌─────────── hour (0-23)
│ │ ┌───────── day of month (1-31)
│ │ │ ┌─────── month (1-12)
│ │ │ │ ┌───── day of week (0-6, Sunday = 0)
│ │ │ │ │
* * * * *
```

Always UTC. The shortest interval is 5 minutes. Scheduled runs only happen on the default branch, can be delayed under load, and are disabled automatically after 60 days of repo inactivity on public repos.

#### Manual runs with inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: Where to deploy
        type: choice
        options: [staging, production]
        default: staging
        required: true
      dry_run:
        description: Skip the actual deploy
        type: boolean
        default: false
      version:
        description: Version to release
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ inputs.version }} to ${{ inputs.environment }} (dry run: ${{ inputs.dry_run }})"
```

Input types: `string`, `boolean`, `choice`, `number`, `environment`.

#### Chaining workflows with workflow_run

```yaml
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    steps:
      - run: echo "CI passed, deploying"
```

#### Choosing the right PR trigger

```mermaid
flowchart TD
    A[Need to run on pull requests?] --> B{Does it need secrets<br/>or a write token on forks?}
    B -- No --> C[pull_request<br/>safe default]
    B -- Yes --> D{Will you check out<br/>the PR's code?}
    D -- No, only label/comment --> E[pull_request_target<br/>OK, trusted code only]
    D -- Yes --> F[Split in two:<br/>pull_request builds and uploads an artifact,<br/>workflow_run uses the artifact with secrets]
```


### 4. Jobs

Jobs are the units of parallelism.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    needs: build                 # wait for build
    runs-on: ubuntu-latest
    steps: [...]
```

#### Job keys

| Key | Purpose |
|---|---|
| `runs-on` | Runner label(s), e.g. `ubuntu-latest`, `windows-latest`, `macos-latest`, or `[self-hosted, linux]`. |
| `name` | Display name. |
| `needs` | Job id or list of ids that must succeed first. |
| `if` | Condition for running the job. |
| `permissions` | Token permissions for this job. |
| `environment` | Target environment (approvals, secrets). |
| `concurrency` | Concurrency group for this job. |
| `outputs` | Values exposed to downstream jobs. |
| `env` | Environment variables for the job. |
| `defaults.run` | Default `shell` and `working-directory`. |
| `steps` | The steps. |
| `timeout-minutes` | Kill the job after this long (default 360). |
| `strategy` | Matrix and fail-fast settings. |
| `continue-on-error` | Don't fail the workflow if this job fails. |
| `container` | Run steps inside a Docker container. |
| `services` | Sidecar containers (databases etc.). |
| `uses` / `with` / `secrets` | Call a reusable workflow instead of running steps. |

#### Dependencies

```mermaid
flowchart LR
    lint --> build
    test --> build
    build --> deploy_staging[deploy-staging]
    deploy_staging --> deploy_prod[deploy-production]
```

```yaml
jobs:
  lint:  { runs-on: ubuntu-latest, steps: [...] }
  test:  { runs-on: ubuntu-latest, steps: [...] }
  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps: [...]
```

If a needed job fails or is skipped, dependents are skipped too. Override with `if: ${{ always() }}`, or `if: ${{ !cancelled() }}` to still run after a failure.

#### Conditions

```yaml
if: github.ref == 'refs/heads/main'
if: github.event_name == 'pull_request' && github.event.pull_request.draft == false
if: contains(github.event.head_commit.message, '[deploy]')
if: ${{ failure() }}              # a previous step/job failed
```

#### Job outputs

```yaml
jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      value: ${{ steps.get.outputs.version }}
    steps:
      - id: get
        run: echo "version=1.4.2" >> "$GITHUB_OUTPUT"

  use:
    needs: version
    runs-on: ubuntu-latest
    steps:
      - run: echo "Version is ${{ needs.version.outputs.value }}"
```

#### Run steps in a container

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: node:20-alpine
      env:
        NODE_ENV: test
    steps:
      - uses: actions/checkout@v4
      - run: node --version
```


### 5. Steps

A step is either `run` (shell) or `uses` (action). Never both.

```yaml
steps:
  - name: Check out code
    uses: actions/checkout@v4
    with:
      fetch-depth: 0

  - name: Print info
    id: info
    run: |
      echo "Branch: $GITHUB_REF_NAME"
      echo "sha=$GITHUB_SHA" >> "$GITHUB_OUTPUT"
    shell: bash
    working-directory: ./app
    env:
      GREETING: hello
    continue-on-error: false
    timeout-minutes: 5
    if: success()
```

#### Step keys

| Key | Purpose |
|---|---|
| `name` | Label in the log. |
| `id` | Lets later steps read `steps.<id>.outputs`. |
| `uses` | An action: `owner/repo@ref`, `./local/path`, or `docker://image:tag`. |
| `run` | Shell commands. Multi-line with `|`. |
| `with` | Inputs for the action. |
| `env` | Env vars for this step. |
| `if` | Condition. Default is `success()`. |
| `shell` | `bash`, `pwsh`, `python`, `sh`, `cmd`, `powershell`, or a custom template. |
| `working-directory` | Directory for `run`. |
| `continue-on-error` | Treat failure as non-fatal. |
| `timeout-minutes` | Per-step timeout. |

#### Referencing actions

```yaml
- uses: actions/checkout@v4                     # tag (convenient)
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11   # commit SHA (safest)
- uses: actions/checkout@main                   # branch (can change under you)
- uses: ./.github/actions/my-action             # action in the same repo
- uses: docker://alpine:3.20                    # a Docker image directly
```

#### Status check functions

Use these in `if`:

| Function | True when |
|---|---|
| `success()` | All previous steps succeeded (default). |
| `failure()` | Any previous step failed. |
| `always()` | Always, even if cancelled. |
| `cancelled()` | The workflow was cancelled. |

```yaml
- name: Upload logs on failure
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: logs
    path: logs/
```


### 6. Expressions and contexts

Anything inside `${{ ... }}` is an expression, evaluated by GitHub before the step runs.

#### Contexts

| Context | What it holds | Examples |
|---|---|---|
| `github` | Event and repo info | `github.repository`, `github.ref`, `github.sha`, `github.actor`, `github.event_name`, `github.event.pull_request.number` |
| `env` | Environment variables | `env.NODE_ENV` |
| `vars` | Configuration variables | `vars.REGION` |
| `secrets` | Secrets | `secrets.NPM_TOKEN` |
| `job` | Current job | `job.status`, `job.container.id` |
| `jobs` | Reusable workflow job outputs | `jobs.build.outputs.x` |
| `steps` | Earlier steps | `steps.build.outputs.path`, `steps.build.outcome` |
| `runner` | Runner info | `runner.os`, `runner.arch`, `runner.temp` |
| `strategy` | Matrix strategy | `strategy.job-index` |
| `matrix` | Current matrix values | `matrix.node` |
| `needs` | Outputs of dependencies | `needs.build.outputs.version`, `needs.build.result` |
| `inputs` | Inputs of manual or reusable runs | `inputs.environment` |

#### Operators

`( )` `!` `<` `<=` `>` `>=` `==` `!=` `&&` `||` and property access with `.` or `[ ]`. Comparisons are case-insensitive for strings.

#### Functions

| Function | Example |
|---|---|
| `contains(search, item)` | `contains(github.event.pull_request.labels.*.name, 'bug')` |
| `startsWith(s, prefix)` / `endsWith(s, suffix)` | `startsWith(github.ref, 'refs/tags/')` |
| `format(str, ...)` | `format('Hello {0} {1}', 'a', 'b')` |
| `join(array, sep)` | `join(matrix.os, ', ')` |
| `toJSON(v)` / `fromJSON(s)` | `fromJSON(needs.setup.outputs.matrix)` |
| `hashFiles(pattern)` | `hashFiles('**/package-lock.json')` |
| `success()`, `failure()`, `always()`, `cancelled()` | status checks |

#### Handy default environment variables

| Variable | Meaning |
|---|---|
| `GITHUB_REPOSITORY` | `owner/repo` |
| `GITHUB_SHA` | Commit SHA that triggered the run |
| `GITHUB_REF` | Full ref, e.g. `refs/heads/main` |
| `GITHUB_REF_NAME` | Short ref name, e.g. `main` |
| `GITHUB_ACTOR` | User who triggered the run |
| `GITHUB_EVENT_NAME` | The event name |
| `GITHUB_WORKSPACE` | Where the repo is checked out |
| `GITHUB_RUN_ID` / `GITHUB_RUN_NUMBER` / `GITHUB_RUN_ATTEMPT` | Run identifiers |
| `GITHUB_OUTPUT` | File to write step outputs to |
| `GITHUB_ENV` | File to write env vars for later steps |
| `GITHUB_PATH` | File to prepend to `PATH` |
| `GITHUB_STEP_SUMMARY` | File to write Markdown for the run summary |
| `RUNNER_OS` / `RUNNER_TEMP` | OS name and temp directory |
| `CI` | Always `true` |


### 7. Passing data around

```mermaid
flowchart LR
    S1[Step A<br/>writes GITHUB_OUTPUT] -->|steps.a.outputs.x| S2[Step B<br/>same job]
    S1 -->|GITHUB_ENV| S3[Step C<br/>env var]
    J1[Job 1 outputs] -->|needs.job1.outputs.x| J2[Job 2]
    J1 -->|upload-artifact| ART[(Artifact)] -->|download-artifact| J2
```

| Need | Use |
|---|---|
| Between steps of the same job | `$GITHUB_OUTPUT` and `steps.<id>.outputs.<name>` |
| Env var for later steps | `echo "NAME=value" >> "$GITHUB_ENV"` |
| Add to PATH | `echo "/my/bin" >> "$GITHUB_PATH"` |
| Between jobs (small values) | Job `outputs` and `needs.<job>.outputs.<name>` |
| Between jobs (files) | `actions/upload-artifact` and `actions/download-artifact` |
| Show results on the run page | Append Markdown to `$GITHUB_STEP_SUMMARY` |

```yaml
- name: Report
  run: |
    {
      echo "## Test results"
      echo "| Suite | Result |"
      echo "|---|---|"
      echo "| unit | passed |"
    } >> "$GITHUB_STEP_SUMMARY"
```

Multi-line output values need a delimiter:

```yaml
- run: |
    {
      echo "notes<<EOF"
      git log --oneline -5
      echo "EOF"
    } >> "$GITHUB_OUTPUT"
```


### 8. Secrets, variables and environments

| | Secrets | Variables |
|---|---|---|
| Stored encrypted | Yes | No |
| Masked in logs | Yes | No |
| Access | `secrets.NAME` | `vars.NAME` |
| Use for | Tokens, passwords, keys | Region names, flags, non-sensitive config |

They can be set at three levels: **repository**, **environment**, and **organization**. The most specific level wins.

```yaml
steps:
  - run: ./deploy.sh
    env:
      API_TOKEN: ${{ secrets.API_TOKEN }}
      REGION: ${{ vars.REGION }}
```

Notes:
- Secrets are not passed to workflows triggered by `pull_request` from forks.
- Do not echo secrets. Pass them through `env` rather than inlining them in `run` text.
- Secrets are not automatically passed to reusable workflows. Use `secrets: inherit` or list them.

#### Environments

An environment (Settings, Environments) is a named deployment target such as `staging` or `production`. It can have:

- **Required reviewers** who must approve before the job starts
- **Wait timers**
- **Deployment branch rules** (only `main` may deploy to production)
- Its own secrets and variables

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - run: ./deploy.sh
```

```mermaid
sequenceDiagram
    participant W as Workflow
    participant E as Environment: production
    participant R as Reviewer
    W->>E: Job requests deployment
    E->>R: Approval needed
    R-->>E: Approve
    E-->>W: Secrets unlocked, job starts
```


### 9. Permissions and GITHUB_TOKEN

Every run gets an automatic token, `secrets.GITHUB_TOKEN`, scoped to the repo and valid only for that run. Limit it with `permissions`.

```yaml
permissions:
  contents: read          # workflow-wide default

jobs:
  release:
    permissions:
      contents: write     # this job may push tags and create releases
      pull-requests: write
```

Available scopes: `actions`, `attestations`, `checks`, `contents`, `deployments`, `discussions`, `id-token`, `issues`, `models`, `packages`, `pages`, `pull-requests`, `repository-projects`, `security-events`, `statuses`. Each can be `read`, `write` or `none`.

- Setting any permission makes all unlisted scopes `none`.
- `permissions: {}` removes all permissions.
- Prefer least privilege: start with `contents: read` and add only what a job needs.
- Changes made with `GITHUB_TOKEN` do **not** trigger new workflow runs (prevents infinite loops). Use a PAT or a GitHub App token if you need that.


### 10. Matrix builds

A matrix runs the same job across combinations of values.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false          # keep other jobs running if one fails
      max-parallel: 4
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
        include:
          - os: ubuntu-latest
            node: 22
            coverage: true      # extra variable for this one combination
        exclude:
          - os: macos-latest
            node: 18            # skip this combination
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci && npm test
```

```mermaid
flowchart TD
    M[matrix: os x node] --> A[ubuntu + 18]
    M --> B[ubuntu + 20]
    M --> C[ubuntu + 22 + coverage]
    M --> D[windows + 18]
    M --> E[windows + 20]
    M --> F[windows + 22]
    M --> G[macos + 20]
    M --> H[macos + 22]
```

Here 3 x 3 = 9 combinations, minus the excluded macOS + 18 = 8 jobs. A matrix may have up to 256 jobs.

A dynamic matrix built from JSON:

```yaml
jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set.outputs.matrix }}
    steps:
      - id: set
        run: echo 'matrix={"service":["api","web","worker"]}' >> "$GITHUB_OUTPUT"
  build:
    needs: plan
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJSON(needs.plan.outputs.matrix) }}
    steps:
      - run: echo "Building ${{ matrix.service }}"
```


### 11. Caching and artifacts

#### Cache: speed up repeated work

Restore dependencies between runs.

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-
```

Many `setup-*` actions have a built-in `cache:` input (`setup-node`, `setup-python`, `setup-java`, `setup-go`) which is simpler.

```mermaid
flowchart LR
    A[Compute key from lockfile hash] --> B{Exact match?}
    B -- Yes --> C[Restore cache, skip install]
    B -- No --> D{restore-keys match?}
    D -- Yes --> E[Restore closest cache, install the difference]
    D -- No --> F[Full install]
    C --> G[Run job]
    E --> G
    F --> G
    G --> H[Post step: save cache if key was new]
```

Caches are scoped by branch: a PR can read the default branch's cache, but not the other way round. Unused caches are evicted after 7 days and the repo limit is 10 GB.

#### Artifacts: keep files from a run

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
    retention-days: 7
    if-no-files-found: error

# in another job
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: dist/
```

| | Cache | Artifact |
|---|---|---|
| Purpose | Reuse dependencies across runs | Share or keep outputs of one run |
| Lifetime | Up to 7 days idle | 1 to 90 days (default 90) |
| Download from UI | No | Yes |
| Typical content | `node_modules`, `.m2`, pip wheels | Builds, test reports, logs |


### 12. Reusable workflows

Write a workflow once and call it from others. The called workflow uses `on: workflow_call`.

`.github/workflows/build.yml` (the callee):

```yaml
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: "20"
    secrets:
      NPM_TOKEN:
        required: false
    outputs:
      artifact:
        value: ${{ jobs.build.outputs.artifact }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact: build-${{ github.sha }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci && npm run build
```

The caller:

```yaml
jobs:
  build:
    uses: ./.github/workflows/build.yml          # same repo
    # uses: owner/repo/.github/workflows/build.yml@v1   # other repo
    with:
      node-version: "22"
    secrets: inherit

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.artifact }}"
```

```mermaid
flowchart LR
    subgraph Caller workflow
      A[job: build<br/>uses: build.yml] --> B[job: deploy]
    end
    subgraph Called workflow
      C[on: workflow_call] --> D[jobs...]
    end
    A -. inputs, secrets .-> C
    D -. outputs .-> A
```

Limits: up to 10 levels of nesting, and a caller job that uses a reusable workflow cannot also have `steps`.


### 13. Custom actions

Three kinds, all defined by an `action.yml` file.

| Type | Runs | Best for |
|---|---|---|
| **Composite** | A list of steps | Bundling shell steps and other actions |
| **JavaScript** | Node.js on the runner | Fast, cross-platform, uses the toolkit |
| **Docker container** | A container (Linux runners only) | Specific tools or languages |

#### Composite action

`.github/actions/setup-project/action.yml`:

```yaml
name: Setup project
description: Install Node and dependencies
inputs:
  node-version:
    description: Node version
    default: "20"
outputs:
  cache-hit:
    description: Whether the cache was restored
    value: ${{ steps.setup.outputs.cache-hit }}
runs:
  using: composite
  steps:
    - id: setup
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - run: npm ci
      shell: bash          # required for run steps in composite actions
```

Use it:

```yaml
- uses: actions/checkout@v4
- uses: ./.github/actions/setup-project
  with:
    node-version: "22"
```

#### JavaScript action

```yaml
name: Hello
description: Greets someone
inputs:
  who:
    required: true
runs:
  using: node20
  main: dist/index.js
```

```js
const core = require("@actions/core");
const who = core.getInput("who");
core.info(`Hello, ${who}`);
core.setOutput("greeting", `Hello, ${who}`);
```

#### Docker action

```yaml
name: Lint docs
description: Runs a linter in a container
runs:
  using: docker
  image: Dockerfile
  args:
    - ${{ inputs.path }}
```

#### Versioning your action

Tag releases (`v1.0.0`) and move a major tag (`v1`) to the latest compatible release so users can pin `@v1`.


### 14. Runners

| Runner label | OS | Notes |
|---|---|---|
| `ubuntu-latest`, `ubuntu-24.04`, `ubuntu-22.04` | Linux | Fastest and cheapest. Default choice. |
| `windows-latest`, `windows-2022` | Windows | For Windows-specific builds. |
| `macos-latest`, `macos-14`, `macos-13` | macOS | For iOS/macOS builds. Costs more minutes. |
| `ubuntu-24.04-arm`, etc. | Linux ARM64 | Native ARM builds. |
| Larger runners | Linux/Windows/macOS | More CPU/RAM, paid, configured per org. |
| `self-hosted` | Yours | Your own machine, with labels. |

Free minutes depend on your plan, and public repos on standard hosted runners are free. Windows costs 2x and macOS 10x the Linux rate on private repos.

#### Self-hosted runners

```yaml
runs-on: [self-hosted, linux, x64, gpu]
```

- You manage the OS, tools, updates and security.
- Never attach self-hosted runners to public repos that accept fork PRs, since a PR could run arbitrary code on your machine.
- Prefer ephemeral runners (one job, then wiped).


### 15. Concurrency and cancellation

`concurrency` makes sure only one run (or job) per group is active.

```yaml
# cancel the older run when a new commit arrives on the same branch
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

```yaml
# deployments: queue, never cancel mid-deploy
jobs:
  deploy:
    concurrency:
      group: production-deploy
      cancel-in-progress: false
```

```mermaid
sequenceDiagram
    participant U as You
    participant G as Group "ci-main"
    U->>G: push #1 (run starts)
    U->>G: push #2
    Note over G: cancel-in-progress: true
    G-->>G: run #1 cancelled
    G->>G: run #2 starts
```

With `cancel-in-progress: false`, only one run is active and at most one more waits; extras in between are dropped.


### 16. Service containers

Run a database or cache alongside the job.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      - run: npm test
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/postgres
          REDIS_URL: redis://localhost:6379
```

- If the job runs directly on the runner, reach services at `localhost:<mapped port>`.
- If the job uses `container:`, reach them by service name, e.g. `postgres:5432`.


### 17. Workflow commands

Special lines your scripts can print to talk to the runner.

| Command | Effect |
|---|---|
| `echo "::notice file=app.js,line=1::Message"` | Annotation, shown on the run and diff |
| `echo "::warning ::Message"` | Warning annotation |
| `echo "::error file=app.js,line=10::Message"` | Error annotation |
| `echo "::group::Title"` ... `echo "::endgroup::"` | Collapsible log group |
| `echo "::add-mask::$VALUE"` | Hide a value in the logs |
| `echo "::debug::Message"` | Debug log (needs debug logging on) |
| `echo "::stop-commands::TOKEN"` ... `echo "::TOKEN::"` | Stop and resume command processing |

Setting outputs and env vars now uses files (`$GITHUB_OUTPUT`, `$GITHUB_ENV`, `$GITHUB_PATH`), not the deprecated `::set-output` and `::set-env`.


### 18. Security best practices

1. **Least privilege.** Set `permissions: contents: read` at workflow level and widen per job.
2. **Pin third-party actions to a full commit SHA.** Tags can be moved; SHAs cannot. Dependabot can keep them updated.

   ```yaml
   - uses: some/action@a1b2c3d4e5f6...   # v2.3.1
   ```
3. **Treat untrusted input as code.** Anything from an issue title, PR title, branch name or commit message can contain shell metacharacters. Never put it directly in `run`:

   ```yaml
   # vulnerable: title becomes part of the script
   - run: echo "${{ github.event.issue.title }}"

   # safe: pass through an env var
   - run: echo "$TITLE"
     env:
       TITLE: ${{ github.event.issue.title }}
   ```
4. **Be careful with `pull_request_target` and `workflow_run`.** They run with secrets. Never check out and execute fork code in them.
5. **Prefer OIDC to long-lived cloud keys.** Request a short-lived token instead of storing credentials:

   ```yaml
   permissions:
     id-token: write
     contents: read
   steps:
     - uses: aws-actions/configure-aws-credentials@v4
       with:
         role-to-assume: arn:aws:iam::123456789012:role/deploy
         aws-region: us-east-1
   ```
6. **Use environments** with required reviewers for production secrets.
7. **Don't use self-hosted runners for public repos.**
8. **Scan your workflows** with tools like `zizmor` or `actionlint`, and enable CodeQL for Actions.
9. **Protect `.github/workflows`** with CODEOWNERS so changes need review.
10. **Restrict allowed actions** in repo or org settings (only GitHub-authored, verified creators, or an allow-list).


### 19. Debugging

| Technique | How |
|---|---|
| Re-run with debug logging | "Re-run jobs", tick "Enable debug logging" (sets `ACTIONS_STEP_DEBUG`). |
| Dump a context | `- run: echo '${{ toJSON(github) }}'` (never dump `secrets`). |
| Lint workflows locally | `actionlint` catches syntax, expression and shell mistakes. |
| Run locally | `act` runs workflows in Docker on your machine (not identical to GitHub). |
| Re-run only failures | "Re-run failed jobs" button or `gh run rerun --failed`. |
| Inspect from the CLI | `gh run list`, `gh run view <id> --log-failed`, `gh run watch`. |
| Trigger manually | `gh workflow run ci.yml -f environment=staging` |

Common mistakes:
- Wrong indentation or a tab in YAML.
- A workflow file only on a feature branch, but trigger needs the default branch (`schedule`, `workflow_dispatch` list, `issue_comment`).
- `needs` job skipped, so the dependent job is skipped silently.
- Using `${{ }}` in an `if` with a leading `!`. Write `if: ${{ !cancelled() }}` or quote it.
- Path filters preventing a required check from running.
- Forgetting `shell: bash` in composite action `run` steps.


### 20. Limits worth knowing

| Item | Limit |
|---|---|
| Job run time | 6 hours (hosted) |
| Workflow run time | 35 days |
| Matrix jobs per workflow run | 256 |
| Reusable workflow nesting | 10 levels, 50 unique reusable workflows per run |
| `workflow_dispatch` inputs | 25 top-level inputs |
| Cache size per repo | 10 GB, evicted after 7 days of no access |
| Artifact retention | 1 to 90 days |
| API requests with `GITHUB_TOKEN` | 1,000 per hour per repo |
| Concurrent jobs (Free plan) | 20 (5 for macOS) |

Limits change, so check the official docs for current numbers.


### 21. Frequently used actions

| Action | Purpose |
|---|---|
| `actions/checkout` | Check out your repo. Almost always the first step. |
| `actions/setup-node`, `setup-python`, `setup-java`, `setup-go`, `setup-dotnet` | Install a language toolchain with optional caching. |
| `actions/cache` | Cache dependencies. |
| `actions/upload-artifact` / `download-artifact` | Store and share files. |
| `actions/github-script` | Run JavaScript with an authenticated Octokit client inline. |
| `actions/labeler` | Label PRs by changed paths. |
| `actions/stale` | Close inactive issues and PRs. |
| `actions/first-interaction` | Welcome first-time contributors. |
| `actions/configure-pages`, `upload-pages-artifact`, `deploy-pages` | Deploy to GitHub Pages. |
| `actions/attest-build-provenance` | Sign build provenance. |
| `docker/setup-buildx-action`, `docker/login-action`, `docker/build-push-action` | Build and push container images. |
| `github/codeql-action` | Code scanning. |
| `softprops/action-gh-release` | Create releases with assets. |
| `dependabot/fetch-metadata` | Read Dependabot PR info for auto-merge rules. |

Inline script example:

```yaml
- uses: actions/github-script@v7
  with:
    script: |
      await github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: "Thanks for the report!"
      });
```


### 22. Quick reference

#### Minimal CI

```yaml
name: CI
on: [push, pull_request]
permissions:
  contents: read
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make test
```

#### Key syntax at a glance

```yaml
on: { push: { branches: [main] } }        # trigger
runs-on: ubuntu-latest                    # runner
needs: [a, b]                             # job order
if: github.ref == 'refs/heads/main'       # condition
strategy: { matrix: { node: [18, 20] } }  # fan-out
env: { KEY: value }                       # env vars
${{ secrets.TOKEN }}                      # secret
${{ vars.NAME }}                          # variable
${{ steps.id.outputs.name }}              # step output
${{ needs.job.outputs.name }}             # job output
echo "k=v" >> "$GITHUB_OUTPUT"            # set output
echo "K=v" >> "$GITHUB_ENV"               # set env for later steps
```

#### GitHub CLI for Actions

```bash
gh workflow list                       # list workflows
gh workflow run ci.yml -f key=value    # trigger workflow_dispatch
gh run list --workflow ci.yml          # recent runs
gh run view <run-id> --log-failed      # logs of failed steps
gh run watch <run-id>                  # follow live
gh run rerun <run-id> --failed         # re-run failed jobs
gh run download <run-id>               # download artifacts
gh run cancel <run-id>                 # cancel a run
gh secret set NAME                     # add a secret
gh variable set NAME --body value      # add a variable
```

#### Official documentation

- Workflow syntax: <https://docs.github.com/actions/reference/workflow-syntax-for-github-actions>
- Events that trigger workflows: <https://docs.github.com/actions/reference/events-that-trigger-workflows>
- Contexts: <https://docs.github.com/actions/reference/contexts-reference>
- Security hardening: <https://docs.github.com/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions>

### 23. Complete example workflows

Copy any of these into the path shown.

#### CI with a matrix

Runs tests on every push and PR, across two operating systems and two Node versions. Superseded runs are cancelled. Save as `.github/workflows/ci.yml`.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: Test (node ${{ matrix.node }}, ${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    timeout-minutes: 15
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm
      - run: npm ci
      - run: npm test

  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
```

#### Release on a version tag

Pushing a tag like `v1.2.0` builds the project, then a second job publishes a GitHub release with the build files. Save as `.github/workflows/release.yml`.

```yaml
name: Release

on:
  push:
    tags: ["v*.*.*"]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  release:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - name: Create GitHub release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh release create "$GITHUB_REF_NAME" dist/* \
            --repo "$GITHUB_REPOSITORY" \
            --generate-notes
```

#### Manual deploy with inputs

A "Run workflow" button with a dropdown for the environment and a dry-run switch. Production can require reviewer approval. Save as `.github/workflows/deploy.yml`.

```yaml
name: Manual deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        description: Where to deploy
        type: choice
        options: [staging, production]
        default: staging
      dry_run:
        description: Print the steps without deploying
        type: boolean
        default: true

permissions:
  contents: read

concurrency:
  group: deploy-${{ inputs.environment }}
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        env:
          TARGET: ${{ inputs.environment }}
          DRY_RUN: ${{ inputs.dry_run }}
        run: |
          if [ "$DRY_RUN" = "true" ]; then
            echo "Dry run: would deploy $GITHUB_SHA to $TARGET"
          else
            echo "Deploying $GITHUB_SHA to $TARGET"
            # ./scripts/deploy.sh "$TARGET"
          fi
      - name: Summary
        run: echo "Deployed ${GITHUB_SHA::7} to ${{ inputs.environment }}" >> "$GITHUB_STEP_SUMMARY"
```

#### Housekeeping: stale issues and welcome comment

One workflow, two triggers: a nightly cron closes stale issues, and opening an issue posts a greeting. Save as `.github/workflows/housekeeping.yml`.

```yaml
name: Housekeeping

on:
  schedule:
    - cron: "0 3 * * *"
  issues:
    types: [opened]

permissions:
  issues: write
  pull-requests: write

jobs:
  stale:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/stale@v9
        with:
          days-before-stale: 60
          days-before-close: 14
          stale-issue-message: This issue has had no activity for 60 days and will be closed in 14 days unless it is updated.
          exempt-issue-labels: pinned,security

  welcome:
    if: github.event_name == 'issues'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: "Thanks for opening an issue. A maintainer will take a look soon."
            });
```
