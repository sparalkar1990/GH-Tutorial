# GitHub Actions Complete Tutorial

This tutorial teaches GitHub Actions from first workflow to production deployment. It includes copy-ready YAML for CI, Docker, Kubernetes, reusable workflows, caching, artifacts, secrets, environments, OIDC, matrices, and security hardening.

Official references:

- https://docs.github.com/en/actions
- https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax
- https://docs.github.com/en/actions/reference/workflows-and-actions/contexts
- https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows

## 1. What GitHub Actions Is

GitHub Actions is GitHub's automation platform. You define workflows in YAML files under `.github/workflows/`. A workflow runs when an event happens, such as a push, pull request, release, issue update, manual dispatch, or schedule.

Core terms:

- Workflow: A YAML automation file.
- Event: Something that triggers a workflow.
- Job: A group of steps that run on the same runner.
- Step: A shell command or action invocation.
- Action: A reusable automation unit.
- Runner: The machine that executes jobs.
- Context: Runtime data like `github`, `env`, `secrets`, `matrix`, and `needs`.
- Expression: Dynamic syntax inside `${{ ... }}`.

Minimal workflow:

```yaml
name: Hello Actions

on:
  push:

jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - name: Print message
        run: echo "Hello from GitHub Actions"
```

## 2. Workflow File Location

Workflows must live here:

```text
.github/workflows/<workflow-name>.yml
.github/workflows/<workflow-name>.yaml
```

Example:

```text
.github/workflows/ci.yml
```

## 3. Workflow Syntax Map

Most workflows use this shape:

```yaml
name: CI

run-name: CI for ${{ github.ref_name }} by @${{ github.actor }}

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

env:
  NODE_VERSION: "22"

defaults:
  run:
    shell: bash

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - name: Print context
        run: echo "Running on $GITHUB_REF"
```

Important top-level keys:

- `name`: Display name.
- `run-name`: Dynamic run label.
- `on`: Trigger events.
- `permissions`: Default `GITHUB_TOKEN` permissions.
- `env`: Workflow-wide environment variables.
- `defaults`: Default shell or working directory.
- `concurrency`: Prevent duplicate runs.
- `jobs`: Work to execute.

## 4. Events And Triggers

### Push

```yaml
on:
  push:
    branches:
      - main
      - "release/**"
    paths:
      - "src/**"
      - ".github/workflows/**"
```

### Pull Request

```yaml
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, ready_for_review]
```

### Manual Trigger

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod
      dry_run:
        description: Run without deploying
        type: boolean
        default: true
```

Use the inputs:

```yaml
steps:
  - run: echo "Deploying to ${{ inputs.environment }}"
```

### Schedule

GitHub schedules use UTC cron.

```yaml
on:
  schedule:
    - cron: "0 2 * * 1"
```

### Release

```yaml
on:
  release:
    types: [published]
```

### Tags

```yaml
on:
  push:
    tags:
      - "v*.*.*"
```

### Call From Another Workflow

```yaml
on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
    secrets:
      npm-token:
        required: false
```

### Multiple Events Together

```yaml
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:
```

## 5. Jobs

Jobs run independently by default.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```

Make one job wait for another with `needs`:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "build"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploy"
```

## 6. Steps

A step can run shell commands:

```yaml
steps:
  - name: Run tests
    run: |
      npm ci
      npm test
```

Or call an action:

```yaml
steps:
  - uses: actions/checkout@v4
```

Use `working-directory`:

```yaml
steps:
  - name: Test API
    working-directory: services/api
    run: npm test
```

Use a different shell:

```yaml
steps:
  - name: PowerShell step
    shell: pwsh
    run: Write-Host "Hello"
```

## 7. Runners

GitHub-hosted runner examples:

```yaml
runs-on: ubuntu-latest
runs-on: windows-latest
runs-on: macos-latest
```

Self-hosted runner:

```yaml
runs-on: [self-hosted, linux, x64]
```

Larger runner or runner group labels are configured in GitHub, then referenced by label:

```yaml
runs-on: ubuntu-22.04-large
```

## 8. Actions

Actions are reusable steps. Pin third-party actions to a trusted version.

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: actions/setup-node@v4
    with:
      node-version: "22"
      cache: npm
```

Use a local action:

```yaml
steps:
  - uses: ./.github/actions/my-action
```

## 9. Environment Variables

Workflow-level:

```yaml
env:
  APP_NAME: demo
```

Job-level:

```yaml
jobs:
  build:
    env:
      NODE_ENV: test
```

Step-level:

```yaml
steps:
  - run: echo "$MESSAGE"
    env:
      MESSAGE: hello
```

Append environment variables for later steps:

```yaml
steps:
  - run: echo "BUILD_ID=123" >> "$GITHUB_ENV"
  - run: echo "$BUILD_ID"
```

## 10. Variables And Secrets

Repository or organization variables are accessed with `vars`.

```yaml
run: echo "Region is ${{ vars.AWS_REGION }}"
```

Secrets are accessed with `secrets`.

```yaml
run: echo "Deploying"
env:
  API_TOKEN: ${{ secrets.API_TOKEN }}
```

Do not print secrets. GitHub masks many secret values, but you should still treat logs as visible.

## 11. Contexts

Common contexts:

- `github`: Repository, actor, ref, event data.
- `env`: Environment variables.
- `vars`: Repository, environment, or organization variables.
- `secrets`: Encrypted secrets.
- `inputs`: Manual or reusable workflow inputs.
- `matrix`: Current matrix values.
- `needs`: Outputs from prerequisite jobs.
- `steps`: Outputs from previous steps in the same job.
- `runner`: Runner details.
- `job`: Job details.

Example:

```yaml
steps:
  - run: |
      echo "Repository: ${{ github.repository }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "Actor: ${{ github.actor }}"
      echo "Runner OS: ${{ runner.os }}"
```

## 12. Expressions

Expressions use `${{ ... }}`.

```yaml
if: ${{ github.ref == 'refs/heads/main' }}
```

Common functions:

```yaml
if: ${{ contains(github.event.head_commit.message, '[deploy]') }}
if: ${{ startsWith(github.ref, 'refs/tags/v') }}
if: ${{ success() }}
if: ${{ failure() }}
if: ${{ always() }}
```

Ternary-like pattern:

```yaml
env:
  TARGET: ${{ github.ref == 'refs/heads/main' && 'prod' || 'dev' }}
```

Convert JSON:

```yaml
strategy:
  matrix: ${{ fromJSON(vars.TEST_MATRIX_JSON) }}
```

## 13. Conditions

Run a job only on main:

```yaml
jobs:
  deploy:
    if: ${{ github.ref == 'refs/heads/main' }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy"
```

Run a step only when previous steps fail:

```yaml
steps:
  - run: npm test
  - name: Upload logs on failure
    if: ${{ failure() }}
    uses: actions/upload-artifact@v4
    with:
      name: test-logs
      path: logs/
```

## 14. Matrix Builds

Run a job across versions and operating systems:

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: ["20", "22"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test
```

Include custom combinations:

```yaml
strategy:
  matrix:
    node: ["20", "22"]
    include:
      - node: "22"
        experimental: true
```

Exclude combinations:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: ["20", "22"]
    exclude:
      - os: windows-latest
        node: "20"
```

## 15. Job Outputs

Expose data from one job to another:

```yaml
jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.image_tag }}
    steps:
      - id: meta
        run: echo "image_tag=1.0.${{ github.run_number }}" >> "$GITHUB_OUTPUT"

  deploy:
    needs: version
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.version.outputs.image_tag }}"
```

## 16. Permissions And GITHUB_TOKEN

Use least privilege.

```yaml
permissions:
  contents: read
```

For pull request comments:

```yaml
permissions:
  contents: read
  pull-requests: write
```

For package publishing:

```yaml
permissions:
  contents: read
  packages: write
```

For OIDC cloud login:

```yaml
permissions:
  contents: read
  id-token: write
```

Use `GITHUB_TOKEN`:

```yaml
steps:
  - name: Create issue comment
    env:
      GH_TOKEN: ${{ github.token }}
    run: gh pr comment "${{ github.event.pull_request.number }}" --body "CI passed"
```

## 17. Concurrency

Cancel old runs on the same branch:

```yaml
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Serialize production deploys:

```yaml
concurrency:
  group: production
  cancel-in-progress: false
```

## 18. Caching

Node dependency cache:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with:
      node-version: "22"
      cache: npm
  - run: npm ci
```

Generic cache:

```yaml
steps:
  - uses: actions/cache@v4
    with:
      path: ~/.cache/pip
      key: pip-${{ runner.os }}-${{ hashFiles('**/requirements.txt') }}
      restore-keys: |
        pip-${{ runner.os }}-
```

## 19. Artifacts

Upload files from one job:

```yaml
steps:
  - run: mkdir dist && echo "build output" > dist/app.txt
  - uses: actions/upload-artifact@v4
    with:
      name: app-dist
      path: dist/
      retention-days: 7
```

Download files in another job:

```yaml
steps:
  - uses: actions/download-artifact@v4
    with:
      name: app-dist
      path: dist/
```

## 20. Services

Run PostgreSQL for tests:

```yaml
jobs:
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: app_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - run: npm test
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/app_test
```

## 21. Job Containers

Run all job steps inside a container:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: node:22
    steps:
      - uses: actions/checkout@v4
      - run: node --version
      - run: npm test
```

## 22. Defaults

Set shell and working directory:

```yaml
defaults:
  run:
    shell: bash
    working-directory: app
```

## 23. Timeouts And Retries

Set a job timeout:

```yaml
jobs:
  test:
    timeout-minutes: 20
    runs-on: ubuntu-latest
    steps:
      - run: npm test
```

GitHub Actions does not have a built-in retry keyword for arbitrary commands. Use a retry action or script carefully:

```yaml
steps:
  - name: Retry flaky command
    run: |
      for i in 1 2 3; do
        npm test && exit 0
        sleep 10
      done
      exit 1
```

## 24. Environments And Approvals

Environments can hold secrets, variables, protection rules, and manual approvals.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - run: echo "Deploy production"
```

Environment secret:

```yaml
env:
  PROD_TOKEN: ${{ secrets.PROD_TOKEN }}
```

## 25. Deployment Pattern

Typical CI/CD flow:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "test"

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: echo "build"

  deploy:
    needs: build
    if: ${{ github.ref == 'refs/heads/main' }}
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploy"
```

## 26. Docker Build And Push

```yaml
name: Docker

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

## 27. Kubernetes Deployment

Kubernetes deployments usually need cloud auth, a kubeconfig secret, or OIDC. This example uses a base64 kubeconfig secret named `KUBE_CONFIG_B64`.

```yaml
name: Kubernetes Deploy

on:
  workflow_dispatch:
    inputs:
      image:
        description: Container image to deploy
        required: true
        type: string

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configure kubeconfig
        run: |
          mkdir -p "$HOME/.kube"
          echo "${{ secrets.KUBE_CONFIG_B64 }}" | base64 -d > "$HOME/.kube/config"
          chmod 600 "$HOME/.kube/config"

      - name: Deploy image
        run: |
          kubectl set image deployment/my-app my-app="${{ inputs.image }}" --namespace default
          kubectl rollout status deployment/my-app --namespace default --timeout=120s
```

## 28. OIDC For Cloud Authentication

OIDC avoids long-lived cloud secrets. The cloud provider must trust your GitHub repository and branch.

AWS example:

```yaml
permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: us-east-1

      - run: aws sts get-caller-identity
```

## 29. Reusable Workflows

Reusable workflow file:

```yaml
name: Reusable Node Test

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - run: npm ci
      - run: npm test
```

Caller workflow:

```yaml
jobs:
  call-tests:
    uses: ./.github/workflows/reusable-node-test.yml
    with:
      node-version: "22"
```

Pass secrets:

```yaml
jobs:
  call-tests:
    uses: ./.github/workflows/reusable-node-test.yml
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

## 30. Composite Actions

Create `.github/actions/setup-app/action.yml`:

```yaml
name: Setup App
description: Check out code and install Node dependencies

inputs:
  node-version:
    required: true
    description: Node.js version

runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - shell: bash
      run: npm ci
```

Use it:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-app
    with:
      node-version: "22"
```

## 31. Custom JavaScript Or Docker Actions

JavaScript action metadata:

```yaml
name: My JS Action
description: Example JavaScript action
inputs:
  name:
    required: true
runs:
  using: node20
  main: dist/index.js
```

Docker action metadata:

```yaml
name: My Docker Action
description: Example Docker action
runs:
  using: docker
  image: Dockerfile
```

## 32. Workflow Commands

Set step output:

```yaml
steps:
  - id: build
    run: echo "version=1.2.3" >> "$GITHUB_OUTPUT"
  - run: echo "${{ steps.build.outputs.version }}"
```

Add markdown to the run summary:

```yaml
steps:
  - run: |
      echo "## Test Summary" >> "$GITHUB_STEP_SUMMARY"
      echo "- Passed" >> "$GITHUB_STEP_SUMMARY"
```

Mask a value:

```yaml
steps:
  - run: echo "::add-mask::sensitive-value"
```

Group logs:

```yaml
steps:
  - run: |
      echo "::group::Install"
      npm ci
      echo "::endgroup::"
```

## 33. Debugging

Enable debug logging by adding repository secrets:

```text
ACTIONS_STEP_DEBUG=true
ACTIONS_RUNNER_DEBUG=true
```

Useful debug steps:

```yaml
steps:
  - run: pwd
  - run: ls -la
  - run: env | sort
  - run: git status --short
```

Print event payload path:

```yaml
steps:
  - run: cat "$GITHUB_EVENT_PATH"
```

Do not dump full contexts that may include sensitive fields.

## 34. Security Checklist

Use these defaults:

```yaml
permissions:
  contents: read
```

Security practices:

- Pin actions to trusted versions, preferably commit SHAs for high-security repositories.
- Use least-privilege `permissions`.
- Prefer OIDC over long-lived cloud keys.
- Do not echo secrets.
- Avoid running untrusted pull request code with privileged tokens.
- Be careful with `pull_request_target`.
- Use environments for production approvals.
- Review third-party actions before use.
- Restrict self-hosted runners to trusted workflows.
- Use branch protection and required checks.

Dangerous `pull_request_target` pattern:

```yaml
on:
  pull_request_target:

jobs:
  unsafe:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - run: ./untrusted-script.sh
```

Safer pattern: use `pull_request` for untrusted code and keep token permissions read-only.

```yaml
on:
  pull_request:

permissions:
  contents: read
```

## 35. Status Badges

Add to `README.md`:

```markdown
[![CI](https://github.com/OWNER/REPO/actions/workflows/ci.yml/badge.svg)](https://github.com/OWNER/REPO/actions/workflows/ci.yml)
```

## 36. Skipping Workflows

Common commit message markers:

```text
[skip ci]
[ci skip]
[no ci]
[skip actions]
[actions skip]
```

## 37. Dependabot And Actions Updates

Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

## 38. Example Full CI Workflow

See `.github/workflows/ci.yml` in this repository for a complete workflow with lint, test matrix, build, artifacts, and deploy gating.

## 39. Example Reusable Workflow

See `.github/workflows/reusable-node-test.yml` for a workflow that can be called by another workflow.

## 40. Example Kubernetes Workflow

See `.github/workflows/kubernetes-deploy.yml` for a manual Kubernetes deployment template.

## 41. Common Mistakes

- Placing workflows outside `.github/workflows/`.
- Forgetting `actions/checkout`.
- Using secrets in pull request workflows from forks.
- Giving `GITHUB_TOKEN` write permissions everywhere.
- Assuming schedule cron uses local time instead of UTC.
- Expecting jobs to share filesystem state without artifacts or caches.
- Forgetting `needs` when jobs must run in order.
- Printing entire contexts into logs.
- Using `pull_request_target` without understanding the security model.

## 42. Learning Path

1. Create a basic `push` workflow.
2. Add `pull_request`.
3. Add lint and tests.
4. Add a matrix.
5. Add cache.
6. Upload artifacts.
7. Add `needs` and deployment gating.
8. Add environments and approvals.
9. Replace cloud keys with OIDC.
10. Extract repeated logic into reusable workflows or composite actions.

