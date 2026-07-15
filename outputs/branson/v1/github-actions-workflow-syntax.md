# Workflow syntax for GitHub Actions

**Source:** <https://docs.github.com/actions/reference/workflows-and-actions/workflow-syntax>

**TL;DR** — A workflow is a YAML file (`.yml`/`.yaml`) stored in `.github/workflows/`, made up of one or more jobs. Top level sets the trigger (`on`), token `permissions`, workflow `env`, `defaults`, `concurrency`, and the `jobs` map. Each job picks a runner (`runs-on`), optionally declares `needs`/`if`/`strategy`/`container`/`services`, and runs a sequence of `steps`; a step either runs a shell command (`run`) or an action (`uses`).

> **Fidelity note.** This page is built from server-rendered `reusables` includes. Many keys below have no body in the source file — only a heading. Those are listed under **"Keys defined by include"** with their identifier preserved so the outline is complete; consult the live page for their full text. Every section with visible prose or a code example is distilled in full.

**At a glance — top-level keys:** `name` · `run-name` · `on` · `permissions` · `env` · `defaults` · `concurrency` · `jobs`
**Job keys:** `name` · `permissions` · `needs` · `if` · `runs-on` · `environment` · `concurrency` · `outputs` · `env` · `defaults` · `steps` · `timeout-minutes` · `strategy` · `continue-on-error` · `container` · `services` · `uses` · `with` · `secrets`
**Step keys:** `id` · `if` · `name` · `uses` · `run` · `working-directory` · `shell` · `with` · `env` · `continue-on-error` · `timeout-minutes` · `background` · `wait` · `wait-all` · `cancel` · `parallel`

## YAML basics

Workflow files use YAML and must end in `.yml` or `.yaml`, stored in the repository's `.github/workflows` directory.

## `run-name`

The name for workflow runs generated from the workflow, shown in the Actions tab. If omitted or whitespace-only, the run name defaults to event-specific info (e.g. commit message for `push`, PR title for `pull_request`). Can include expressions and reference the `github` and `inputs` contexts.

```yaml
run-name: Deploy to ${{ inputs.deploy_target }} by @${{ github.actor }}
```

## `on` triggers

`on` defines what triggers the workflow (bodies are includes — see live page). Event-specific keys that **do** carry content in the source:

### `on.schedule`

Defines a time schedule for workflows. See [events-that-trigger-workflows#schedule](https://docs.github.com/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule).

### `on.workflow_call`

Defines the inputs and outputs for a **reusable workflow**, and maps the secrets available to the called workflow.

#### `on.workflow_call.inputs`

Optional inputs passed from the caller. Each input **requires** a `type`. Missing `default` values default to `false` (boolean), `0` (number), `""` (string). Reference inputs inside the called workflow via the `inputs` context. Passing an input not specified in the called workflow is an error.

```yaml
on:
  workflow_call:
    inputs:
      username:
        description: 'A username passed from the caller workflow'
        default: 'john-doe'
        required: false
        type: string

jobs:
  print-username:
    runs-on: ubuntu-latest
    steps:
      - name: Print the input name to STDOUT
        run: echo The username is ${{ inputs.username }}
```

- **`on.workflow_call.inputs.<input_id>.type`** — required; one of `boolean`, `number`, or `string`.

#### `on.workflow_call.outputs`

A map of outputs for a called workflow, available to all downstream jobs in the caller. Each output has an identifier, optional `description`, and a `value` set to a job output from within the called workflow.

```yaml
on:
  workflow_call:
    outputs:
      workflow_output1:
        description: "The first job output"
        value: ${{ jobs.my_job.outputs.job_output1 }}
      workflow_output2:
        description: "The second job output"
        value: ${{ jobs.my_job.outputs.job_output2 }}
```

#### `on.workflow_call.secrets`

A map of secrets usable in the called workflow, referenced via the `secrets` context. Passing an unspecified secret is an error. To pass a secret to a **nested** reusable workflow, use `jobs.<job_id>.secrets` again.

```yaml
on:
  workflow_call:
    secrets:
      access-token:
        description: 'A token passed from the caller workflow'
        required: false

jobs:
  pass-secret-to-action:
    runs-on: ubuntu-latest
    steps:
      - name: Pass the received secret to an action
        uses: ./.github/actions/my-action
        with:
          token: ${{ secrets.access-token }}
  pass-secret-to-workflow:
    uses: ./.github/workflows/my-workflow
    secrets:
       token: ${{ secrets.access-token }}
```

- **`on.workflow_call.secrets.<secret_id>`** — string identifier for the secret.
- **`on.workflow_call.secrets.<secret_id>.required`** — boolean; whether the secret must be supplied.

### `on.workflow_dispatch` inputs

- **`on.workflow_dispatch.inputs.<input_id>.required`** — boolean; whether the input must be supplied.
- **`on.workflow_dispatch.inputs.<input_id>.type`** — one of `boolean`, `choice`, `number`, `environment`, or `string`.

## `permissions`

Controls the `GITHUB_TOKEN` scopes. Set at workflow level (applies to all jobs) or per job.

**How permissions are calculated:** the `GITHUB_TOKEN` starts at the default for the enterprise/org/repo. A restricted default at any level applies to its repositories. Permissions are then adjusted by the workflow-level `permissions`, then the job-level `permissions`. Finally, for a pull-request event other than `pull_request_target` from a **forked** repository, if **Send write tokens to workflows from pull requests** is not selected, any write permissions are reduced to read-only.

For forked repositories you can add/remove `read` permissions but typically cannot grant `write` — unless an admin selected **Send write tokens to workflows from pull requests**.

## `env`

A map of variables available to the steps of all jobs. Variables in `env` cannot be defined in terms of other variables in the same map.

```yaml
env:
  SERVER: production
```

## `jobs`

A workflow run is made of one or more jobs (`jobs.<job_id>`), which run in parallel by default. Each `job_id` is a unique string key.

### `jobs.<job_id>.env`

A map of variables available to all steps in the job.

```yaml
jobs:
  job1:
    env:
      FIRST_NAME: Mona
```

### `jobs.<job_id>.timeout-minutes`

Maximum minutes to let a job run before GitHub cancels it. **Default: 360.** If it exceeds the runner's execution time limit, the job is canceled at that limit instead.

> The `GITHUB_TOKEN` may be the limiting factor for self-hosted runners if the job timeout exceeds 24 hours.

### `jobs.<job_id>.continue-on-error`

Prevents a workflow run from failing when this job fails. Set `true` to let the run pass when the job fails; in a matrix, other jobs keep running.

```yaml
runs-on: ${{ matrix.os }}
continue-on-error: ${{ matrix.experimental }}
strategy:
  fail-fast: false
  matrix:
    node: [13, 14]
    os: [macos-latest, ubuntu-latest]
    experimental: [false]
    include:
      - node: 15
        os: ubuntu-latest
        experimental: true
```

## Steps — `jobs.<job_id>.steps`

A job runs a sequence of `steps`. Each step runs in its own process with access to the workspace and filesystem; **environment-variable changes are not preserved between steps**. GitHub displays only the first 1,000 checks, but you can run unlimited steps within usage limits.

```yaml
name: Greeting from Mona
on: push
jobs:
  my-job:
    name: My Job
    runs-on: ubuntu-latest
    steps:
      - name: Print a greeting
        env:
          MY_VAR: Hi there! My name is
          FIRST_NAME: Mona
          MIDDLE_NAME: The
          LAST_NAME: Octocat
        run: |
          echo $MY_VAR $FIRST_NAME $MIDDLE_NAME $LAST_NAME.
```

### `steps[*].id`

Unique identifier for the step, used to reference it in contexts.

### `steps[*].if`

Prevents a step from running unless a condition is met.

```yaml
steps:
  - name: My first step
    if: ${{ github.event_name == 'pull_request' && github.event.action == 'unassigned' }}
    run: echo This event is a pull request that had an assignee removed.
```

Using status-check functions (`failure()`, `success()`, `always()`, `cancelled()`):

```yaml
steps:
  - name: My first step
    uses: octo-org/action-name@main
  - name: My backup step
    if: ${{ failure() }}
    uses: actions/heroku@1.0.0
```

**Secrets** cannot be referenced directly in `if:`. Set them as job-level env vars and reference those instead; an unset secret evaluates to an empty string.

```yaml
name: Run a step if a secret has been set
on: push
jobs:
  my-jobname:
    runs-on: ubuntu-latest
    env:
      super_secret: ${{ secrets.SuperSecret }}
    steps:
      - if: ${{ env.super_secret != '' }}
        run: echo 'This step will only run if the secret has a value set.'
      - if: ${{ env.super_secret == '' }}
        run: echo 'This step will only run if the secret does not have a value set.'
```

### `steps[*].name`

A display name for the step.

### `steps[*].uses`

Selects an action to run. An action can live in the same repo, a public repo, or a published Docker image. **Pin the version** with a Git ref, SHA, or Docker tag:

- Commit SHA — safest for stability and security.
- Major-version tag — receives fixes/patches while retaining compatibility (at the author's discretion).
- Default branch — convenient but a new major version could break your workflow.

Some actions require inputs set via `with`. Docker-container actions must run in a Linux environment (see `runs-on`).

```yaml
steps:
  # Reference a specific commit
  - uses: actions/checkout@8f4b7f84864484a7bf31766abe9204da3cbe65b3
  # Reference the major version of a release
  - uses: actions/checkout@v5
  # Reference a specific version
  - uses: actions/checkout@v5.2.0
  # Reference a branch
  - uses: actions/checkout@main
```

Reference syntaxes:
- Public action: `{owner}/{repo}@{ref}` — e.g. `actions/heroku@main`, `actions/aws@v2.0.1`
- Public action subdirectory: `{owner}/{repo}/{path}@{ref}` — e.g. `actions/aws/ec2@main`
- Same-repo action: `./path/to/dir` — you must check out the repo first
- Docker Hub image: `docker://{image}:{tag}` — e.g. `docker://alpine:3.8`
- GitHub Container Registry: `docker://ghcr.io/OWNER/IMAGE_NAME`
- Public Docker registry: `docker://{host}/{image}:{tag}` — e.g. `docker://gcr.io/cloud-builders/gradle`

For an action in a **different private repo** not configured for access, check out that repo using a token secret, then reference the action locally:

```yaml
jobs:
  my_first_job:
    steps:
      - name: Check out repository
        uses: actions/checkout@v5
        with:
          repository: octocat/my-private-repo
          ref: v1.0
          token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
          path: ./.github/actions/my-private-repo
      - name: Run my action
        uses: ./.github/actions/my-private-repo/my-action
```

### `steps[*].run`

Runs command-line programs (max 21,000 characters) using the OS shell. If no `name` is given, the step name defaults to the `run` command. Commands use non-login shells by default. Each `run` is a new process/shell; multi-line commands share one shell.

```yaml
- name: Install Dependencies
  run: npm install

- name: Clean install dependencies and build
  run: |
    npm ci
    npm run build
```

### `steps[*].working-directory`

Sets the working directory for the command.

```yaml
- name: Clean temp directory
  run: rm -rf *
  working-directory: ./temp
```

### `steps[*].shell`

Overrides the default shell. Built-in keywords include `bash`, `sh`, `pwsh`, `powershell`, `cmd`, `python`. The shell runs a temporary file containing the `run` commands.

```yaml
# examples by shell
- { shell: bash,       run: echo $PATH }
- { shell: cmd,        run: echo %PATH% }
- { shell: pwsh,       run: echo ${env:PATH} }
- { shell: powershell, run: echo ${env:PATH} }
- shell: python
  run: |
    import os
    print(os.environ['PATH'])
```

**Custom shell** — set `shell` to a template string `command [options] {0} [more_options]`. GitHub treats the first word as the command and inserts the temp script's filename at `{0}`. The command (e.g. `perl`) must be installed on the runner.

```yaml
- name: Display the environment variables and their values
  shell: perl {0}
  run: |
    print %ENV
```

**Exit codes / error-action preference** (defaults on GitHub-hosted runners):
- `bash`/`sh` — fail-fast via `set -e`; `shell: bash` also adds `-o pipefail`. Override with a template like `bash {0}`. `sh`-like shells exit with the last command's code.
- `powershell`/`pwsh` — prepends `$ErrorActionPreference = 'stop'`; appends `if ((Test-Path -LiteralPath variable:\LASTEXITCODE)) { exit $LASTEXITCODE }`. Opt out with a custom shell like `pwsh -File {0}`.
- `cmd` — no full fail-fast; you must check error codes in your script. `cmd.exe` exits with the last program's error level.

### `steps[*].with`

A map of input parameters defined by the action. Inputs become environment variables prefixed with `INPUT_` and upper-cased (e.g. `first_name` → `INPUT_FIRST_NAME`). Docker-container inputs must use `args`.

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: actions/hello_world@main
        with:
          first_name: Mona
          middle_name: The
          last_name: Octocat
```

- **`with.args`** — a string of inputs for a Docker container, passed to the container's `ENTRYPOINT` at startup (replaces the `Dockerfile` `CMD`). Arrays are not supported; quote arguments containing spaces.

  ```yaml
  steps:
    - name: Explain why this job ran
      uses: octo-org/action-name@main
      with:
        entrypoint: /bin/echo
        args: The ${{ github.event_name }} event triggered this step.
  ```

- **`with.entrypoint`** — overrides (or sets) the Docker `ENTRYPOINT`; accepts a single string executable. Usable with Docker-container actions and with JavaScript actions that define no inputs.

### `steps[*].env`

Sets variables for the step in the runner environment. Set secrets/sensitive values via the `secrets` context, not directly.

```yaml
steps:
  - name: My first action
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      FIRST_NAME: Mona
      LAST_NAME: Octocat
```

### `steps[*].continue-on-error`

Set `true` to let the job pass even when this step fails.

### `steps[*].timeout-minutes`

Maximum minutes to run the step before killing the process. **Maximum: 360.** Must be a positive integer; fractional values unsupported.

### Background steps (`background`, `wait`, `wait-all`, `cancel`, `parallel`)

- **`steps[*].background`** — runs a step asynchronously; the job continues without waiting. Use for long-running processes (databases, servers, monitors). Works on `run` or `uses` steps; give it an `id` to reference later. **Max 10 background steps run concurrently** per job; extras queue. Outputs/env changes are available only after a `wait`/`wait-all` that includes the step. A failed background step fails the job at the next `wait`/`wait-all` (unless `continue-on-error`). An implicit `wait-all` runs before post-job cleanup. **Not allowed inside a composite action** (a composite action can itself be a background step).

  ```yaml
  steps:
    - name: Start server
      id: server
      run: npm start
      background: true
    - name: Run tests against the server
      run: npm test
    - name: Wait for the server step to finish
      wait: server
  ```

- **`steps[*].wait`** — pauses the job until named background steps complete (a single `id` string or an array). Does no work itself; fails if a referenced step failed. Always runs; does **not** support `if`.

  ```yaml
  steps:
    - { name: Build frontend, id: build-frontend, run: npm run build:frontend, background: true }
    - { name: Build backend,  id: build-backend,  run: npm run build:backend,  background: true }
    - name: Run linter while builds run
      run: npm run lint
    - name: Wait for both builds to finish
      wait: [build-frontend, build-backend]
    - name: Run tests
      run: npm test
  ```

- **`steps[*].wait-all`** — pauses until **all** active background steps complete. Takes no arguments. Fails if any waited-on step failed (unless `continue-on-error`). Always runs; does not support `if`.

- **`steps[*].cancel`** — gracefully terminates one background step by `id` (`SIGTERM`, then `SIGKILL` after a short grace period). Always runs; does not support `if`.

  ```yaml
  steps:
    - { name: Start long-running monitor, id: monitor, run: ./scripts/monitor.sh, background: true }
    - name: Run the main task
      run: npm test
    - name: Stop the monitor
      cancel: monitor
  ```

- **`steps[*].parallel`** — runs a group of steps concurrently, then waits for all to finish (shorthand: every step runs as a background step with an implicit `wait`). Use for a self-contained group that should all finish before the job continues; use `background` when you need finer control. Same 10-step concurrency limit. **Not allowed inside a composite action.**

  ```yaml
  steps:
    - uses: actions/checkout@v5
    - parallel:
        - { name: Build frontend, run: npm run build:frontend }
        - { name: Build backend,  run: npm run build:backend }
        - { name: Build docs,     run: npm run build:docs }
    - name: Run tests after all builds complete
      run: npm test
  ```

## Matrix strategy — `jobs.<job_id>.strategy.matrix`

Defines a matrix of job configurations. **A matrix generates at most 256 jobs per workflow run** (GitHub-hosted and self-hosted). Matrix variables become properties in the `matrix` context (e.g. `matrix.version`). By default GitHub maximizes parallel jobs by runner availability; the order of variables determines job creation order.

**Single dimension** — `version: [10, 12, 14]` runs three jobs:

```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        version: [10, 12, 14]
    steps:
      - uses: actions/setup-node@v5
        with:
          node-version: ${{ matrix.version }}
```

**Multi-dimensional** — a job runs for every combination. Two OSes × three versions = six jobs:

```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        os: [ubuntu-22.04, ubuntu-24.04]
        version: [10, 12, 14]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v5
        with:
          node-version: ${{ matrix.version }}
```

A variable can be an array of objects (produces 4 jobs here):

```yaml
matrix:
  os:
    - ubuntu-latest
    - macos-latest
  node:
    - version: 14
    - version: 20
      env: NODE_OPTIONS=--openssl-legacy-provider
```

- **`matrix.include`** — for each object in `include`, its key:value pairs are added to matrix combinations that don't overwrite original values; otherwise a new combination is created. Original values are never overwritten, but added values can be.
- **`matrix.exclude`** — a partial match excludes a combination. All `include` combinations are processed **after** `exclude`, so `include` can add back excluded combinations.
- **`strategy.max-parallel`** — by default GitHub maximizes parallel jobs by runner availability.

## Service containers — `jobs.<job_id>.services`

Hosts service containers (e.g. databases, Redis) for a job. The runner creates a Docker network and manages the containers' lifecycle. If the job runs **in a container** (or a step uses container actions), ports need not be mapped — reference the service by its label/hostname. If the job runs **directly on the runner**, map required ports to the host and access via `localhost` and the mapped port.

```yaml
services:
  nginx:
    image: nginx
    ports:
      - 8080:80        # host 8080 -> container 80
  redis:
    image: redis
    ports:
      - 6379/tcp       # random free host port -> container 6379
steps:
  - run: |
      echo "Redis available on 127.0.0.1:${{ job.services.redis.ports['6379'] }}"
      echo "Nginx available on 127.0.0.1:${{ job.services.nginx.ports['80'] }}"
```

Service sub-keys:
- **`services.<service_id>.image`** — Docker Hub image name or registry name. An empty string means the service does not start (useful for conditional services, e.g. `image: ${{ options.nginx == true && 'nginx' || '' }}`).
- **`services.<service_id>.credentials`** — registry `username`/`password` (typically `${{ github.actor }}` / `${{ secrets.github_token }}` or Docker Hub creds).
- **`services.<service_id>.env`** — map of env vars in the service container.
- **`services.<service_id>.ports`** — array of ports to expose.
- **`services.<service_id>.volumes`** — array of volumes, `<source>:<destinationPath>` (named volume or absolute host path : absolute container path).
- **`services.<service_id>.options`** — additional `docker create` options. **`--network` is not supported.**
- **`services.<service_id>.command`** — overrides the image's default `CMD`; passed as args after the image name in `docker create`.
- **`services.<service_id>.entrypoint`** — overrides the image's default `ENTRYPOINT` (single string); combine with `command` to pass args.

## Reusable-workflow job keys

- **`jobs.<job_id>.uses`** — location and version of a reusable workflow file to run as a job.

  ```yaml
  jobs:
    call-workflow:
      uses: octo-org/example-repo/.github/workflows/called-workflow.yml@main
      with:
        username: mona
  ```

- **`jobs.<job_id>.with`** — map of inputs passed to the called workflow; must match the called workflow's input specs. Unlike step-level `with`, these are **not** exposed as env vars — reference them via the `inputs` context.
- **`jobs.<job_id>.with.<input_id>`** — input identifier/value pair; must match an input defined by `on.workflow_call.inputs.<input_id>` and its declared `type`. Allowed expression contexts: `github`, `needs`.
- **`jobs.<job_id>.secrets`** — map of secrets passed to the called workflow; names must match.

  ```yaml
  jobs:
    call-workflow:
      uses: octo-org/example-repo/.github/workflows/called-workflow.yml@main
      secrets:
        access-token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
  ```

- **`jobs.<job_id>.secrets.inherit`** — pass all the caller's secrets (org, repo, environment) to the called workflow. Works across repos in the same org, or across orgs in the same enterprise.

  ```yaml
  jobs:
    pass-secrets-to-workflow:
      uses: ./.github/workflows/called-workflow.yml
      secrets: inherit
  ```

- **`jobs.<job_id>.secrets.<secret_id>`** — secret identifier/value pair; must match `on.workflow_call.secrets.<secret_id>`. Allowed expression contexts: `github`, `needs`, `secrets`.

## Filter pattern cheat sheet

Special characters usable in path, branch, and tag filters:

- `*` — zero or more characters, but **not** `/`. E.g. `Octo*` matches `Octocat`.
- `**` — zero or more of any character.
- `?` — zero or one of the preceding character.
- `+` — one or more of the preceding character.
- `[]` — one character listed or in a range (ranges limited to `a-z`, `A-Z`, `0-9`). E.g. `[CB]at` matches `Cat`/`Bat`; `[1-2]00` matches `100`/`200`.
- `!` — at the start of a pattern, negates previous positive patterns; no special meaning elsewhere.

`*`, `[`, and `!` are special in YAML — a pattern starting with them must be quoted; a flow sequence containing `[`/`]` must also be quoted.

```yaml
# Valid
paths:
  - '**/README.md'
# Invalid - parse error
paths:
  - **/README.md
# Valid
branches: [ main, 'release/v[0-9].[0-9]' ]
# Invalid - parse error
branches: [ main, release/v[0-9].[0-9] ]
```

### Patterns to match branches and tags

| Pattern | Description | Example matches |
| --- | --- | --- |
| `feature/*` | `*` matches any character but not `/`. | `feature/my-branch`, `feature/your-branch` |
| `feature/**` | `**` matches any character including `/`. | `feature/beta-a/my-branch`, `feature/mona/the/octocat` |
| `main`, `releases/mona-the-octocat` | Exact branch/tag name. | `main`, `releases/mona-the-octocat` |
| `'*'` | All names without a `/`. Must be quoted. | `main`, `releases` |
| `'**'` | All names. Default when no filter is used. | `all/the/branches`, `every/tag` |
| `'*feature'` | Must be quoted (starts with `*`). | `mona-feature`, `feature`, `ver-10-feature` |
| `v2*` | Names starting with `v2`. | `v2`, `v2.0`, `v2.9` |
| `v[12].[0-9]+.[0-9]+` | Semver branches/tags with major version 1 or 2. | `v1.10.1`, `v2.0.0` |

### Patterns to match file paths

Path patterns must match the whole path, starting from the repository root.

| Pattern | Description | Example matches |
| --- | --- | --- |
| `'*'` | Any character but not `/`. Must be quoted. | `README.md`, `server.rb` |
| `'*.jsx?'` | `?` matches zero or one preceding character. | `page.js`, `page.jsx` |
| `'**'` | Any character including `/`. Default when no filter. | `all/the/files.md` |
| `'*.js'` | All `.js` files at repo root. | `app.js`, `index.js` |
| `'**.js'` | All `.js` files in the repository. | `index.js`, `js/index.js`, `src/js/app.js` |
| `docs/*` | Files at the root of `docs/` only. | `docs/README.md`, `docs/file.txt` |
| `docs/**` | Any files under `docs/` and subdirs. | `docs/README.md`, `docs/mona/octocat.txt` |
| `docs/**/*.md` | `.md` files anywhere under `docs/`. | `docs/README.md`, `docs/a/markdown/file.md` |
| `'**/docs/**'` | Any files in a `docs` dir anywhere. | `docs/hello.md`, `dir/docs/my-file.txt` |
| `'**/README.md'` | A `README.md` anywhere. | `README.md`, `js/README.md` |
| `'**/*src/**'` | Any file in a folder with `src` suffix. | `a/src/app.js`, `my-src/code/js/app.js` |
| `'**/*-post.md'` | Files with suffix `-post.md` anywhere. | `my-post.md`, `path/their-post.md` |
| `'**/migrate-*.sql'` | Prefix `migrate-`, suffix `.sql`, anywhere. | `migrate-10909.sql`, `db/migrate-v1.0.sql` |
| `'*.md'`, `'!README.md'` | `!` negates; a later negative excludes a match. | `hello.md` (not `README.md`, not `docs/hello.md`) |
| `'*.md'`, `'!README.md'`, `README*` | Checked sequentially; a later pattern re-includes. | `hello.md`, `README.md`, `README.doc` |

## Keys defined by include

These keys exist in the source only as headings backed by server-rendered `reusables` includes (no body in the raw file). Identifiers are preserved for completeness; see the live page for full descriptions.

**Triggers:** `name` · `on` · `on.<event_name>.types` · `on.<pull_request|pull_request_target>.<branches|branches-ignore>` (+ include/exclude examples) · `on.push.<branches|tags|branches-ignore|tags-ignore>` (+ examples) · `on.<push|pull_request|pull_request_target>.<paths|paths-ignore>` (+ examples, git-diff comparisons) · `on.workflow_run.<branches|branches-ignore>` · `on.workflow_dispatch` · `on.workflow_dispatch.inputs`

**Workflow/job config:** `permissions` scopes and forked-repo behavior · `defaults` · `defaults.run` · `defaults.run.shell` · `defaults.run.working-directory` · `concurrency`

**Job keys:** `jobs.<job_id>.name` · `jobs.<job_id>.permissions` · `jobs.<job_id>.needs` · `jobs.<job_id>.if` · `jobs.<job_id>.runs-on` (GitHub-hosted / self-hosted / runner groups) · `jobs.<job_id>.snapshot` · `jobs.<job_id>.environment` · `jobs.<job_id>.concurrency` · `jobs.<job_id>.outputs` · `jobs.<job_id>.defaults[.run[.shell|.working-directory]]` · `jobs.<job_id>.strategy` (intro) · `jobs.<job_id>.strategy.fail-fast` · `jobs.<job_id>.container[.image|.credentials|.env|.ports|.volumes|.options]`
