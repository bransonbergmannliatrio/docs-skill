# Workflow syntax for GitHub Actions

**Sources:**

- **Primary fixture (raw source):** `fixtures/github-actions-workflow-syntax.md` — GitHub Actions "Workflow syntax" reference in raw form, with unexpanded `{% data reusables.* %}` / `{% data variables.product.* %}` Liquid includes.
- **Rendered reference page:** <https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions> — used to resolve include-only prose and canonical examples (fetched in spans; the rendered page truncates past `jobs.<job_id>.permissions`).
- **Raw reusable/include source files** (github/docs, `main`), used to resolve the include-only sections verbatim where the rendered fetch truncated: `data/reusables/actions/*`, `data/reusables/actions/jobs/*`, and `data/reusables/repositories/*`. ~45 reusable files were pulled, e.g. `workflows/section-triggering-a-workflow`, `workflows/triggering-workflow-branches1-4`, `workflows/run-on-specific-branches-or-tags1-4`, `workflows/triggering-a-workflow-paths1-5`, `workflows/section-specifying-branches`, `workflow-dispatch`, `workflow-dispatch-inputs(-example)`, `jobs/section-assigning-permissions-to-jobs(-specific)`, `github-token-available-permissions`, `github-token-scope-descriptions`, `forked-write-permission`, `workflow-runs-dependabot-note`, `jobs/setting-default-values-*`, `jobs/section-using-concurrency(-jobs)`, `actions-group-concurrency`, `jobs/section-using-jobs-in-a-workflow(-id|-name|-needs)`, `jobs/section-using-conditions-to-control-job-execution`, `jobs/choosing-runner-*`, `jobs/section-using-environments-for-jobs`, `jobs/section-defining-outputs-for-jobs`, `supported-shells`, `jobs/about-matrix-strategy`, `jobs/matrix-expand-with-include`, `jobs/matrix-add-with-include`, `jobs/section-using-a-build-matrix-for-your-jobs-failfast`, `docker-container-os-support`, `jobs/section-running-jobs-in-a-container*`, `registry-credentials`, `reusable-workflow-calling-syntax`, `uses-keyword-example`, `repositories/actions-scheduled-workflow-example`.

> Product-variable substitutions applied throughout: `prodname_dotcom`/`github` → **GitHub**, `prodname_actions` → **GitHub Actions**, `prodname_dependabot` → **Dependabot**, `prodname_ghe_server` → **GitHub Enterprise Server**, `prodname_registry` → **GitHub Packages**, `prodname_container_registry` → **Container registry**, `pat_generic` → **personal access token**, `prodname_github_app` → **GitHub App**, `action-checkout` → `actions/checkout@v6`, `action-setup-node` → `actions/setup-node@v4`. YAML examples are reproduced verbatim from the fixture and/or the rendered/reusable sources.

**TL;DR** — A workflow is a YAML file (`.yml`/`.yaml`) stored in `.github/workflows/`, made up of one or more jobs that run in parallel by default. The top level sets a `name`/`run-name`, the trigger (`on`), token `permissions`, workflow `env`, `defaults`, `concurrency`, and the `jobs` map. Each job picks a runner (`runs-on`); optionally declares `needs`, `if`, `environment`, `concurrency`, `outputs`, `strategy`/`matrix`, `container`, and `services`; and runs a sequence of `steps`. A step either runs a shell command (`run`) or an action (`uses`). A job can instead call a reusable workflow via `uses`/`with`/`secrets`, and reusable workflows declare their interface under `on.workflow_call`.

**At a glance — top-level keys:** `name` · `run-name` · `on` · `permissions` · `env` · `defaults` · `concurrency` · `jobs`

**Job keys (`jobs.<job_id>.*`):** `name` · `permissions` · `needs` · `if` · `runs-on` · `snapshot` · `environment` · `concurrency` · `outputs` · `env` · `defaults` · `steps` · `timeout-minutes` · `strategy` · `continue-on-error` · `container` · `services` · `uses` · `with` · `secrets`

**Step keys (`jobs.<job_id>.steps[*].*`):** `id` · `if` · `name` · `uses` · `run` · `working-directory` · `shell` · `with` · `env` · `continue-on-error` · `timeout-minutes` · `background` · `wait` · `wait-all` · `cancel` · `parallel`

> **Runner note:** GitHub Enterprise Server users should use self-hosted runners; GitHub-hosted runners are **not** supported there.

---

## About YAML syntax for workflows

Workflow files use YAML syntax, and must have either a `.yml` or `.yaml` file extension. You must store workflow files in the `.github/workflows` directory of your repository. If you're new to YAML, see [Learn YAML in Y minutes](https://learnxinyminutes.com/docs/yaml/).

## `name`

The name of the workflow. GitHub displays the names of your workflows under your repository's "Actions" tab. If you omit `name`, GitHub displays the workflow file path relative to the root of the repository.

## `run-name`

The name for workflow runs generated from the workflow. GitHub displays the workflow run name in the list of workflow runs on your repository's "Actions" tab. If `run-name` is omitted or is only whitespace, then the run name is set to event-specific information for the workflow run. For example, for a workflow triggered by a `push` or `pull_request` event, it is set as the commit message or the title of the pull request.

This value can include expressions and can reference the `github` and `inputs` contexts.

### Example of `run-name`

```yaml
run-name: Deploy to ${{ inputs.deploy_target }} by @${{ github.actor }}
```

## `on`

To automatically trigger a workflow, use `on` to define which events can cause the workflow to run. For a list of available events, see the "Events that trigger workflows" reference.

You can define single or multiple events that can trigger a workflow, or set a time schedule. You can also restrict the execution of a workflow to only occur for specific files, tags, or branch changes.

```yaml
# Single event
on: push
```

```yaml
# Multiple events
on: [push, fork]
```

## `on.<event_name>.types`

Use `on.<event_name>.types` to define the type of activity that will trigger a workflow run. Most GitHub events are triggered by more than one type of activity. For example, `label` is triggered when a label is `created`, `edited`, or `deleted`. The `types` keyword enables you to narrow down activity that causes the workflow to run. When only one activity type triggers a webhook event, the `types` keyword is unnecessary. You can use an array of event `types`.

```yaml
on:
  label:
    types: [created, edited]
```

## `on.<pull_request|pull_request_target>.<branches|branches-ignore>`

When using the `pull_request` and `pull_request_target` events, you can configure a workflow to run only for pull requests that target specific branches.

Use the `branches` filter when you want to include branch name patterns or when you want to both include and exclude branch name patterns. Use the `branches-ignore` filter when you only want to exclude branch name patterns. You cannot use both the `branches` and `branches-ignore` filters for the same event in a workflow.

If you define both `branches`/`branches-ignore` and `paths`/`paths-ignore`, the workflow will only run when both filters are satisfied.

The `branches` and `branches-ignore` keywords accept glob patterns that use characters like `*`, `**`, `+`, `?`, `!` and others to match more than one branch name. If a name contains any of these characters and you want a literal match, you need to escape each of these special characters with `\`. See the [Filter pattern cheat sheet](#filter-pattern-cheat-sheet).

### Example: Including branches

The patterns defined in `branches` are evaluated against the Git ref's name. The following workflow would run whenever there is a `pull_request` event for a pull request targeting a branch named `main` (`refs/heads/main`), a branch named `mona/octocat` (`refs/heads/mona/octocat`), or a branch whose name starts with `releases/`, like `releases/10` (`refs/heads/releases/10`).

```yaml
on:
  pull_request:
    # Sequence of patterns matched against refs/heads
    branches:
      - main
      - 'mona/octocat'
      - 'releases/**'
```

> If a workflow is skipped due to branch filtering, path filtering, or a commit message, then checks associated with that workflow will remain in a "Pending" state. A pull request that requires those checks to be successful will be blocked from merging.

### Example: Excluding branches

When a pattern matches the `branches-ignore` pattern, the workflow will not run. The following workflow would run whenever there is a `pull_request` event unless the pull request targets a branch named `mona/octocat` or a branch whose name matches `releases/**-alpha`, like `releases/beta/3-alpha`.

```yaml
on:
  pull_request:
    # Sequence of patterns matched against refs/heads
    branches-ignore:
      - 'mona/octocat'
      - 'releases/**-alpha'
```

### Example: Including and excluding branches

You cannot use `branches` and `branches-ignore` to filter the same event in a single workflow. To both include and exclude branch patterns for a single event, use the `branches` filter along with the `!` character to indicate which branches should be excluded. If you define a branch with the `!` character, you must also define at least one branch without the `!` character.

The order that you define patterns matters:

- A matching negative pattern (prefixed with `!`) after a positive match will exclude the Git ref.
- A matching positive pattern after a negative match will include the Git ref again.

The following workflow will run on `pull_request` events targeting `releases/10` or `releases/beta/mona`, but not `releases/10-alpha` or `releases/beta/3-alpha`, because the negative pattern follows the positive pattern.

```yaml
on:
  pull_request:
    branches:
      - 'releases/**'
      - '!releases/**-alpha'
```

## `on.push.<branches|tags|branches-ignore|tags-ignore>`

When using the `push` event, you can configure a workflow to run on specific branches or tags.

Use the `branches`/`branches-ignore` filters for branches and the `tags`/`tags-ignore` filters for tags (you cannot use both a filter and its `-ignore` counterpart for the same ref type in one workflow). If you define only `tags`/`tags-ignore` or only `branches`/`branches-ignore`, the workflow won't run for events affecting the undefined Git ref. If you define neither, the workflow runs for events affecting either branches or tags. If you define both `branches`/`branches-ignore` and `paths`/`paths-ignore`, the workflow only runs when both filters are satisfied. Glob-pattern and escaping rules match those described for pull-request branch filters above.

### Example: Including branches and tags

The following workflow would run whenever there is a `push` event to a branch named `main`, `mona/octocat`, or one starting with `releases/`; or to a tag named `v2` or one starting with `v1.` like `v1.9.1`.

```yaml
on:
  push:
    # Sequence of patterns matched against refs/heads
    branches:
      - main
      - 'mona/octocat'
      - 'releases/**'
    # Sequence of patterns matched against refs/tags
    tags:
      - v2
      - v1.*
```

### Example: Excluding branches and tags

When a pattern matches the `branches-ignore` or `tags-ignore` pattern, the workflow will not run.

```yaml
on:
  push:
    # Sequence of patterns matched against refs/heads
    branches-ignore:
      - 'mona/octocat'
      - 'releases/**-alpha'
    # Sequence of patterns matched against refs/tags
    tags-ignore:
      - v2
      - v1.*
```

### Example: Including and excluding branches and tags

Use the `branches` or `tags` filter with the `!` character to both include and exclude patterns for a single event. The order of patterns matters (negative after positive excludes; positive after negative re-includes). The following runs on pushes to `releases/10` or `releases/beta/mona`, but not `releases/10-alpha` or `releases/beta/3-alpha`.

```yaml
on:
  push:
    branches:
      - 'releases/**'
      - '!releases/**-alpha'
```

## `on.<push|pull_request|pull_request_target>.<paths|paths-ignore>`

When using the `push` and `pull_request` events, you can configure a workflow to run based on what file paths are changed. Path filters are not evaluated for pushes of tags.

Use the `paths` filter to include (or both include and exclude) file path patterns; use `paths-ignore` to only exclude. You cannot use both `paths` and `paths-ignore` for the same event. To both include and exclude, use `paths` prefixed with `!`. The order of `paths` patterns matters (negative after positive excludes; positive after negative re-includes). If you define both `branches`/`branches-ignore` and `paths`/`paths-ignore`, the workflow runs only when both filters are satisfied. The `paths`/`paths-ignore` keywords accept glob patterns using `*` and `**`.

### Example: Including paths

If at least one path matches a pattern in the `paths` filter, the workflow runs. This runs anytime you push a JavaScript file (`.js`).

```yaml
on:
  push:
    paths:
      - '**.js'
```

### Example: Excluding paths

When all the path names match patterns in `paths-ignore`, the workflow will not run. If any path names do not match, the workflow will run. The following only runs on `push` events that include at least one file outside the `docs` directory at the repository root.

```yaml
on:
  push:
    paths-ignore:
      - 'docs/**'
```

### Example: Including and excluding paths

If you define a path with `!`, you must also define at least one path without `!`. This runs anytime the push includes a file in `sub-project` or its subdirectories, unless the file is in `sub-project/docs`.

```yaml
on:
  push:
    paths:
      - 'sub-project/**'
      - '!sub-project/docs/**'
```

### Git diff comparisons

The filter determines whether a workflow should run by evaluating the changed files against the `paths-ignore` or `paths` list. If there are no files changed, the workflow will not run. GitHub generates the list of changed files using **two-dot diffs for pushes** and **three-dot diffs for pull requests**:

- **Pull requests:** three-dot diffs compare the most recent version of the topic branch and the commit where the topic branch was last synced with the base branch.
- **Pushes to existing branches:** a two-dot diff compares the head and base SHAs directly.
- **Pushes to new branches:** a two-dot diff against the parent of the ancestor of the deepest commit pushed.

Limits that change how filtered workflows run:

- If a push contains more than **1,000 commits**, the workflow will **always** run.
- If generating the diff **times out**, the workflow will **always** run.
- If the generated diff contains more than **3,000 files** and the filter-matched files are not in the first 3,000 returned, the workflow will **not** run.

## `on.schedule`

You can use `on.schedule` to define a time schedule for your workflows. Use [POSIX cron syntax](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/crontab.html#tag_20_25_07). By default, scheduled workflows run in UTC; you can optionally specify a timezone using an IANA timezone string. Scheduled workflows run on the latest commit on the default branch. The shortest interval is once every **5 minutes**.

> For schedules whose `timezone` observes daylight saving time (DST), during spring-forward transitions scheduled workflows in skipped hours advance to the next valid time (e.g. a 2:30 AM schedule advances to 3:00 AM).

Cron syntax has five fields separated by a space:

```text
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of the month (1 - 31)
│ │ │ ┌───────────── month (1 - 12 or JAN-DEC)
│ │ │ │ ┌───────────── day of the week (0 - 6 or SUN-SAT)
│ │ │ │ │
* * * * *
```

| Operator | Description | Example |
| --- | --- | --- |
| `*` | Any value | `15 * * * *` runs at every minute 15 of every hour of every day. |
| `,` | Value list separator | `2,10 4,5 * * *` runs at minute 2 and 10 of the 4th and 5th hour of every day. |
| `-` | Range of values | `30 4-6 * * *` runs at minute 30 of the 4th, 5th, and 6th hour. |
| `/` | Step values | `20/15 * * * *` runs every 15 minutes starting from minute 20 through 59 (minutes 20, 35, and 50). |

```yaml
# Runs at 5:30 AM America/New_York, Monday through Friday
on:
  schedule:
    - cron: '30 5 * * 1-5'
      timezone: "America/New_York"
```

A single workflow can be triggered by multiple `schedule` events. Access the schedule that triggered the workflow through the `github.event.schedule` context.

```yaml
on:
  schedule:
    - cron: '30 5 * * 1,3'
    - cron: '30 5,17 * * 2,4'

jobs:
  test_schedule:
    runs-on: ubuntu-latest
    steps:
      - name: Not on Monday or Wednesday
        if: github.event.schedule != '30 5 * * 1,3'
        run: echo "This step will be skipped on Monday and Wednesday"
      - name: Every time
        run: echo "This step will always run"
```

## `on.workflow_call`

Use `on.workflow_call` to define the inputs and outputs for a reusable workflow. You can also map the secrets that are available to the called workflow.

## `on.workflow_call.inputs`

When using the `workflow_call` keyword, you can optionally specify inputs that are passed to the called workflow from the caller workflow.

In addition to the standard input parameters that are available, `on.workflow_call.inputs` requires a `type` parameter. If a `default` parameter is not set, the default value of the input is `false` for a boolean, `0` for a number, and `""` for a string.

Within the called workflow, you can use the `inputs` context to refer to an input. If a caller workflow passes an input that is not specified in the called workflow, this results in an error.

### Example of `on.workflow_call.inputs`

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

## `on.workflow_call.inputs.<input_id>.type`

Required if input is defined for the `on.workflow_call` keyword. The value of this parameter is a string specifying the data type of the input. This must be one of: `boolean`, `number`, or `string`.

## `on.workflow_call.outputs`

A map of outputs for a called workflow. Called workflow outputs are available to all downstream jobs in the caller workflow. Each output has an identifier, an optional `description`, and a `value`. The `value` must be set to the value of an output from a job within the called workflow.

In the example below, two outputs (`workflow_output1` and `workflow_output2`) are mapped to outputs called `job_output1` and `job_output2`, both from a job called `my_job`.

### Example of `on.workflow_call.outputs`

```yaml
on:
  workflow_call:
    # Map the workflow outputs to job outputs
    outputs:
      workflow_output1:
        description: "The first job output"
        value: ${{ jobs.my_job.outputs.job_output1 }}
      workflow_output2:
        description: "The second job output"
        value: ${{ jobs.my_job.outputs.job_output2 }}
```

## `on.workflow_call.secrets`

A map of the secrets that can be used in the called workflow. Within the called workflow, you can use the `secrets` context to refer to a secret.

> If you are passing the secret to a nested reusable workflow, then you must use [`jobs.<job_id>.secrets`](#jobsjob_idsecrets) again to pass the secret.

If a caller workflow passes a secret that is not specified in the called workflow, this results in an error.

### Example of `on.workflow_call.secrets`

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
    # passing the secret to an action
      - name: Pass the received secret to an action
        uses: ./.github/actions/my-action
        with:
          token: ${{ secrets.access-token }}

  # passing the secret to a nested reusable workflow
  pass-secret-to-workflow:
    uses: ./.github/workflows/my-workflow
    secrets:
       token: ${{ secrets.access-token }}
```

## `on.workflow_call.secrets.<secret_id>`

A string identifier to associate with the secret.

## `on.workflow_call.secrets.<secret_id>.required`

A boolean specifying whether the secret must be supplied.

## `on.workflow_run.<branches|branches-ignore>`

When using the `workflow_run` event, you can specify what branches the triggering workflow must run on in order to trigger your workflow. The `branches`/`branches-ignore` filters accept glob patterns (`*`, `**`, `+`, `?`, `!`, escaped with `\` for literals). You cannot use both filters for the same event; to both include and exclude, use `branches` with `!` (negative after positive excludes; positive after negative re-includes).

```yaml
# Runs only when the workflow named "Build" runs on a branch starting with releases/
on:
  workflow_run:
    workflows: ["Build"]
    types: [requested]
    branches:
      - 'releases/**'
```

```yaml
# Runs only when "Build" runs on a branch not named canary
on:
  workflow_run:
    workflows: ["Build"]
    types: [requested]
    branches-ignore:
      - "canary"
```

```yaml
# Runs when "Build" runs on releases/10 or releases/beta/mona, but not
# releases/10-alpha, releases/beta/3-alpha, or main
on:
  workflow_run:
    workflows: ["Build"]
    types: [requested]
    branches:
      - 'releases/**'
      - '!releases/**-alpha'
```

## `on.workflow_dispatch`

When using the `workflow_dispatch` event, you can optionally specify inputs that are passed to the workflow. This trigger only receives events when the workflow file is on the default branch.

## `on.workflow_dispatch.inputs`

The triggered workflow receives the inputs in the `inputs` context.

> - The workflow also receives the inputs in the `github.event.inputs` context. The information is identical except that the `inputs` context preserves Boolean values as Booleans instead of converting them to strings. The `choice` type resolves to a string and is a single selectable option.
> - The maximum number of top-level properties for `inputs` is **25**.
> - The maximum payload for `inputs` is **65,535 characters**.

### Example of `on.workflow_dispatch.inputs`

```yaml
on:
  workflow_dispatch:
    inputs:
      logLevel:
        description: 'Log level'
        required: true
        default: 'warning'
        type: choice
        options:
          - info
          - warning
          - debug
      print_tags:
        description: 'True to print to STDOUT'
        required: true
        type: boolean
      tags:
        description: 'Test scenario tags'
        required: true
        type: string
      environment:
        description: 'Environment to run tests against'
        type: environment
        required: true

jobs:
  print-tag:
    runs-on: ubuntu-latest
    if: ${{ inputs.print_tags }}
    steps:
      - name: Print the input tag to STDOUT
        run: echo The tags are ${{ inputs.tags }}
```

## `on.workflow_dispatch.inputs.<input_id>.required`

A boolean specifying whether the input must be supplied.

## `on.workflow_dispatch.inputs.<input_id>.type`

The value of this parameter is a string specifying the data type of the input. This must be one of: `boolean`, `choice`, `number`, `environment` or `string`.

## `permissions`

You can use `permissions` to modify the default permissions granted to the `GITHUB_TOKEN`, adding or removing access as required, so that you only allow the minimum required access.

You can use `permissions` either as a top-level key (to apply to all jobs in the workflow) or within specific jobs. When you add the `permissions` key within a specific job, all actions and run commands within that job that use the `GITHUB_TOKEN` gain the access rights you specify.

Owners of an organization or enterprise can restrict write access for the `GITHUB_TOKEN` at the repository level. When a workflow is triggered by the `pull_request_target` event, the `GITHUB_TOKEN` is granted read/write repository permission, even when triggered from a public fork.

### Defining access for the `GITHUB_TOKEN` scopes

For each of the available permissions, you can assign one of the access levels: `read` (if applicable), `write`, or `none`. `write` includes `read`. **If you specify the access for any of these permissions, all of those that are not specified are set to `none`.**

| Permission | Allows an action using `GITHUB_TOKEN` to |
| --- | --- |
| `actions` | Work with GitHub Actions. E.g. `actions: write` permits cancelling a workflow run. |
| `artifact-metadata` | Work with artifact metadata. E.g. `artifact-metadata: write` permits creating storage records on behalf of a build artifact. |
| `attestations` | Work with artifact attestations. E.g. `attestations: write` permits generating an artifact attestation for a build. |
| `checks` | Work with check runs and check suites. E.g. `checks: write` permits creating a check run. |
| `code-quality` | Work with code quality. E.g. `code-quality: write` permits uploading code coverage reports. |
| `contents` | Work with repository contents. E.g. `contents: read` permits listing commits; `contents: write` allows creating a release. |
| `deployments` | Work with deployments. E.g. `deployments: write` permits creating a new deployment. |
| `discussions` | Work with GitHub Discussions. E.g. `discussions: write` permits closing or deleting a discussion. |
| `id-token` | Fetch an OpenID Connect (OIDC) token. Requires `id-token: write`. |
| `issues` | Work with issues. E.g. `issues: write` permits adding a comment to an issue. |
| `models` | Generate AI inference responses with GitHub Models. E.g. `models: read` permits using the GitHub Models inference API. |
| `packages` | Work with GitHub Packages. E.g. `packages: write` permits uploading and publishing packages. |
| `pages` | Work with GitHub Pages. E.g. `pages: write` permits requesting a GitHub Pages build. |
| `pull-requests` | Work with pull requests. E.g. `pull-requests: write` permits adding a label to a pull request. |
| `security-events` | Work with code scanning alerts. E.g. `security-events: read` lists code scanning alerts; `security-events: write` updates an alert's status. For Dependabot alerts, use `vulnerability-alerts`. |
| `statuses` | Work with commit statuses. E.g. `statuses: read` permits listing commit statuses for a reference. |
| `vulnerability-alerts` | Read Dependabot alerts. E.g. `vulnerability-alerts: read` lists Dependabot alerts. Only `read` and `none` are supported (`write` is not valid). `write-all`/`read-all` automatically include it as `read`. |

Full scope template, plus the shorthand forms:

```yaml
permissions:
  actions: read|write|none
  artifact-metadata: read|write|none
  attestations: read|write|none
  checks: read|write|none
  code-quality: read|write|none
  contents: read|write|none
  deployments: read|write|none
  id-token: write|none
  issues: read|write|none
  models: read|none
  discussions: read|write|none
  packages: read|write|none
  pages: read|write|none
  pull-requests: read|write|none

  security-events: read|write|none
  statuses: read|write|none
  vulnerability-alerts: read|none
```

```yaml
permissions: read-all
```

```yaml
permissions: write-all
```

```yaml
# Disable all permissions
permissions: {}
```

#### Changing the permissions in a forked repository

You can use the `permissions` key to add and remove read permissions for forked repositories, but typically you can't grant write access. The exception is where an admin user has selected the **Send write tokens to workflows from pull requests** option in the GitHub Actions settings.

## How permissions are calculated for a workflow job

The permissions for the `GITHUB_TOKEN` are initially set to the default setting for the enterprise, organization, or repository. If the default is set to the restricted permissions at any of these levels then this will apply to the relevant repositories. For example, if you choose the restricted default at the organization level then all repositories in that organization will use the restricted permissions as the default. The permissions are then adjusted based on any configuration within the workflow file, first at the workflow level and then at the job level. Finally, if the workflow was triggered by a pull request event other than `pull_request_target` from a forked repository, and the **Send write tokens to workflows from pull requests** setting is not selected, the permissions are adjusted to change any write permissions to read only.

### Setting the `GITHUB_TOKEN` permissions for all jobs in a workflow

You can specify `permissions` at the top level of a workflow, so that the setting applies to all jobs in the workflow.

#### Example: Setting the `GITHUB_TOKEN` permissions for an entire workflow

This example grants read access for all permissions to every job in the workflow.

```yaml
name: "My workflow"

on: [ push ]

permissions: read-all

jobs:
  ...
```

### Using the `permissions` key for forked repositories

You can use the `permissions` key to add and remove `read` permissions for forked repositories, but typically you can't grant `write` access. The exception to this behavior is where an admin user has selected the **Send write tokens to workflows from pull requests** option in the GitHub Actions settings.

### Permissions for workflow runs triggered by Dependabot

Workflow runs triggered by Dependabot pull requests run as if they are from a forked repository, and therefore use a read-only `GITHUB_TOKEN`. These workflow runs cannot access any secrets.

## `env`

A `map` of variables that are available to the steps of all jobs in the workflow. You can also set variables that are only available to the steps of a single job or to a single step (see [`jobs.<job_id>.env`](#jobsjob_idenv) and [`jobs.<job_id>.steps[*].env`](#jobsjob_idstepsenv)).

Variables in the `env` map cannot be defined in terms of other variables in the map.

> When more than one environment variable is defined with the same name, GitHub uses the most specific variable. An environment variable defined in a step overrides job and workflow variables with the same name while the step executes; a job-level variable overrides a workflow variable while the job executes.

### Example of `env`

```yaml
env:
  SERVER: production
```

## `defaults`

Use `defaults` to create a `map` of default settings that will apply to all jobs in the workflow. You can also set default settings that are only available to a job (see [`jobs.<job_id>.defaults`](#jobsjob_iddefaults)).

> When more than one default setting is defined with the same name, GitHub uses the most specific default setting. For example, a default setting defined in a job overrides a default setting with the same name defined in a workflow.

## `defaults.run`

You can use `defaults.run` to provide default `shell` and `working-directory` options for all `run` steps in a workflow. You can also set defaults for `run` that are only available to a job (see [`jobs.<job_id>.defaults.run`](#jobsjob_iddefaultsrun)). You cannot use contexts or expressions in this keyword.

```yaml
defaults:
  run:
    shell: bash
    working-directory: ./scripts
```

## `defaults.run.shell`

Use `shell` to define the `shell` for a step. See the supported-shells table under [`jobs.<job_id>.steps[*].shell`](#jobsjob_idstepsshell). The most specific setting wins.

## `defaults.run.working-directory`

Use `working-directory` to define the working directory for the `shell` for a step.

> Ensure the `working-directory` you assign exists on the runner before you run your shell in it.

## `concurrency`

Use `concurrency` to ensure that only a single job or workflow using the same concurrency group will run at a time. A concurrency group can be any string or expression; at the workflow level the expression can only use the `github`, `inputs`, and `vars` contexts. You can also specify `concurrency` at the job level (see [`jobs.<job_id>.concurrency`](#jobsjob_idconcurrency)).

There can be at most one running job or workflow in a concurrency group at any time. When a concurrent job or workflow is queued while another in the same group is in progress, the queued one becomes `pending`. By default, any existing `pending` job or workflow in the same group is canceled and the new queued one takes its place.

- To also cancel any currently running job or workflow in the same group, specify `cancel-in-progress: true` (or a conditional expression).
- To allow more than one `pending` run to wait, use the optional `queue` property: `single` (default — at most one pending run) or `max` (up to 100 pending runs; extras are canceled when the queue is full). The combination of `queue: max` and `cancel-in-progress: true` is not allowed and causes a validation error.

> - Concurrency group names are case insensitive (`prod` and `Prod` are the same group).
> - Runs in the same group are processed FIFO by the time each started waiting on the group, not the dispatch time; ordering is not guaranteed.

```yaml
# Limit concurrency of entire workflow runs for a branch
on:
  push:
    branches:
      - main

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

```yaml
# Job-level concurrency
jobs:
  job-1:
    runs-on: ubuntu-latest
    concurrency:
      group: staging_environment
      cancel-in-progress: true
```

```yaml
# Queue multiple pending runs instead of canceling
on:
  push:
    branches:
      - main

concurrency:
  group: production-deploy
  queue: max
```

```yaml
# Fallback value (github.head_ref only defined on pull_request events)
concurrency:
  group: ${{ github.head_ref || github.run_id }}
  cancel-in-progress: true
```

```yaml
# Conditional cancellation (do not cancel on release/ branches)
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ !contains(github.ref, 'release/')}}
```

## `jobs`

A workflow run is made up of one or more `jobs`, which run in parallel by default. To run jobs sequentially, define dependencies on other jobs using the `jobs.<job_id>.needs` keyword. Each job runs in a runner environment specified by `runs-on`. You can run an unlimited number of jobs as long as you are within the workflow usage limits. To find the unique identifier of a job running in a workflow run, use the GitHub API (workflow-jobs REST endpoint).

## `jobs.<job_id>`

Use `jobs.<job_id>` to give your job a unique identifier. The key `job_id` is a string and its value is a map of the job's configuration data. Replace `<job_id>` with a string unique to the `jobs` object. The `<job_id>` must start with a letter or `_` and contain only alphanumeric characters, `-`, or `_`.

```yaml
jobs:
  my_first_job:
    name: My first job
  my_second_job:
    name: My second job
```

## `jobs.<job_id>.name`

Use `jobs.<job_id>.name` to set a name for the job, which is displayed in the GitHub UI.

## `jobs.<job_id>.permissions`

For a specific job, you can use `jobs.<job_id>.permissions` to modify the default permissions granted to the `GITHUB_TOKEN`, adding or removing access as required, so that you only allow the minimum required access. By specifying the permission within a job definition, you can configure a different set of permissions for the `GITHUB_TOKEN` for each job. Alternatively, specify permissions for all jobs at the workflow level (see [`permissions`](#permissions)).

The same scope descriptions and access levels apply as at the workflow level (see [Defining access for the `GITHUB_TOKEN` scopes](#defining-access-for-the-github_token-scopes)); `write` includes `read`, and any permission you do not specify is set to `none`.

### Defining access for the `GITHUB_TOKEN` scopes

Identical to the workflow-level scope table and `read-all`/`write-all`/`{}` shorthands documented under [`permissions`](#defining-access-for-the-github_token-scopes) above.

#### Changing the permissions in a forked repository

Same as at the workflow level: you can add/remove read permissions for forked repositories but typically cannot grant write access unless an admin has selected **Send write tokens to workflows from pull requests**.

#### Example: Setting the `GITHUB_TOKEN` permissions for one job in a workflow

This example sets permissions that apply only to the job named `stale`. Write access is granted for `issues` and `pull-requests`; all other permissions have no access.

```yaml
jobs:
  stale:
    runs-on: ubuntu-latest

    permissions:
      issues: write
      pull-requests: write

    steps:
      - uses: actions/stale@v9
```

## `jobs.<job_id>.needs`

Use `jobs.<job_id>.needs` to identify any jobs that must complete successfully before this job will run. It can be a string or array of strings. If a job fails or is skipped, all jobs that need it are skipped unless they use a conditional expression that causes them to continue. A failure or skip applies to all jobs in the dependency chain from the point of failure onwards. To run a job even if a job it depends on did not succeed, use the `always()` conditional expression in `jobs.<job_id>.if`.

```yaml
# Requiring successful dependent jobs — runs sequentially job1, job2, job3
jobs:
  job1:
  job2:
    needs: job1
  job3:
    needs: [job1, job2]
```

```yaml
# Not requiring success — job3 always runs after job1 and job2 complete
jobs:
  job1:
  job2:
    needs: job1
  job3:
    if: ${{ always() }}
    needs: [job1, job2]
```

## `jobs.<job_id>.if`

You can use the `jobs.<job_id>.if` conditional to prevent a job from running unless a condition is met. You can use any supported context and expression.

> The `jobs.<job_id>.if` condition is evaluated **before** [`jobs.<job_id>.strategy.matrix`](#jobsjob_idstrategymatrix) is applied.

When you use expressions in an `if` conditional, you can optionally omit the `${{ }}` syntax because GitHub Actions automatically evaluates the `if` as an expression. However, you must always use `${{ }}` (or escape with `''`, `""`, or `()`) when the expression starts with `!`, since `!` is reserved in YAML.

```yaml
# Only run for a specific repository; otherwise the job is skipped
name: example-workflow
on: [push]
jobs:
  production-deploy:
    if: github.repository == 'octo-org/octo-repo-prod'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: '14'
      - run: npm install -g bats
```

## `jobs.<job_id>.runs-on`

Use `jobs.<job_id>.runs-on` to define the type of machine to run the job on. The destination machine can be a GitHub-hosted runner, a larger/hosted runner in a group, or a self-hosted runner. You can target runners by labels, group membership, or a combination.

You can provide `runs-on` as: a single string; a single variable containing a string; an array of strings/variables (or a mix); or a `key: value` pair using the `group` or `labels` keys. If you specify an array, the workflow executes on any runner that matches **all** of the specified values.

```yaml
# Runs only on a self-hosted runner with all three labels
runs-on: [self-hosted, linux, x64, gpu]
```

```yaml
# Mixing strings and variables (expressions must be quoted)
on:
  workflow_dispatch:
    inputs:
      chosen-os:
        required: true
        type: choice
        options:
        - Ubuntu
        - macOS

jobs:
  test:
    runs-on: [self-hosted, "${{ inputs.chosen-os }}"]
    steps:
    - run: echo Hello world!
```

> Quotation marks are not required around simple strings like `self-hosted`, but they are required for expressions like `"${{ inputs.chosen-os }}"`. To run on multiple machines, use [`jobs.<job_id>.strategy`](#jobsjob_idstrategy).

### Choosing GitHub-hosted runners

If you use a GitHub-hosted runner, each job runs in a fresh instance of a runner image specified by `runs-on`. The value is a workflow label or a runner group name. Beyond the standard runners, GitHub offers Team and Enterprise Cloud customers larger runners (more cores/disk, GPU, ARM).

> `-latest` images are the latest stable images GitHub provides and might not be the most recent OS version from the vendor. Beta and deprecated images are provided "as-is" and excluded from the SLA and warranty.

```yaml
runs-on: ubuntu-latest
```

### Choosing self-hosted runners

Self-hosted runners are selected via labels. Use the `self-hosted` label plus any additional labels you assigned.

```yaml
runs-on: [self-hosted, linux]
```

### Choosing runners in a group

You can use `runs-on` to target runner groups, so the job executes on any runner that is a member of that group. For more granular control, combine runner groups with labels. Runner groups can only have larger/hosted runners or self-hosted runners as members. (Rendered examples use `group:` alone, and `group:` combined with `labels:`; on GHEC/GHES, group-name prefixes can differentiate groups.)

## `jobs.<job_id>.snapshot`

You can use `jobs.<job_id>.snapshot` to generate a custom image. Add the `snapshot` keyword to the job, using either string or mapping syntax. Each job that includes the `snapshot` keyword creates a separate image; to generate only one image/version, include all workflow steps in a single job. Each successful run of a job with `snapshot` creates a new version of that image. (Available on non-GHES versions.)

## `jobs.<job_id>.environment`

Use `jobs.<job_id>.environment` to define the environment that the job references. You can provide the environment as only the environment `name`, or as an object with `name` and `url`. The URL maps to `environment_url` in the deployments API.

> All deployment protection rules must pass before a job referencing the environment is sent to a runner.

```yaml
# Single environment name
environment: staging_environment
```

```yaml
# Name and URL
environment:
  name: production_environment
  url: https://github.com
```

```yaml
# URL from a step output
environment:
  name: production_environment
  url: ${{ steps.step_id.outputs.url_output }}
```

```yaml
# Environment name from an expression
environment:
  name: ${{ github.ref_name }}
```

```yaml
# Use an environment's secrets/variables without creating a deployment
environment:
  name: testing
  deployment: false
```

The value of `url` can be an expression (allowed contexts: `github`, `inputs`, `vars`, `needs`, `strategy`, `matrix`, `job`, `runner`, `env`, `steps`). The value of `name` can be an expression (allowed contexts: `github`, `inputs`, `vars`, `needs`, `strategy`, `matrix`). Setting `deployment: false` is not compatible with custom deployment protection rules.

## `jobs.<job_id>.concurrency`

Use `jobs.<job_id>.concurrency` to ensure that only a single job or workflow using the same concurrency group will run at a time. A concurrency group can be any string or expression (allowed contexts: `github`, `inputs`, `vars`, `needs`, `strategy`, `matrix`). You can also specify `concurrency` at the workflow level (see [`concurrency`](#concurrency)). The queueing/cancellation semantics (`cancel-in-progress`, `queue: single|max`) are identical to workflow-level concurrency.

## `jobs.<job_id>.outputs`

Use `jobs.<job_id>.outputs` to create a `map` of outputs for a job. Job outputs are available to all downstream jobs that depend on this job (via `needs`). Outputs can be a maximum of **1 MB per job**; the total of all outputs in a workflow run can be a maximum of **50 MB** (approximated using UTF-16 encoding). Job outputs containing expressions are evaluated on the runner at the end of each job. Outputs containing secrets are redacted on the runner and not sent to GitHub Actions (you'll see: "Skip output `{output.Key}` since it may contain secret").

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    # Map a step output to a job output
    outputs:
      output1: ${{ steps.step1.outputs.test }}
      output2: ${{ steps.step2.outputs.test }}
    steps:
      - id: step1
        run: echo "test=hello" >> "$GITHUB_OUTPUT"
      - id: step2
        run: echo "test=world" >> "$GITHUB_OUTPUT"
  job2:
    runs-on: ubuntu-latest
    needs: job1
    steps:
      - env:
          OUTPUT1: ${{needs.job1.outputs.output1}}
          OUTPUT2: ${{needs.job1.outputs.output2}}
        run: echo "$OUTPUT1 $OUTPUT2"
```

When using a matrix, job outputs are combined from all jobs inside the matrix. Actions does not guarantee the order matrix jobs run in — ensure each output name is unique, otherwise the last matrix job to run overrides the value.

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      output_1: ${{ steps.gen_output.outputs.output_1 }}
      output_2: ${{ steps.gen_output.outputs.output_2 }}
      output_3: ${{ steps.gen_output.outputs.output_3 }}
    strategy:
      matrix:
        version: [1, 2, 3]
    steps:
      - name: Generate output
        id: gen_output
        run: |
          version="${{ matrix.version }}"
          echo "output_${version}=${version}" >> "$GITHUB_OUTPUT"
  job2:
    runs-on: ubuntu-latest
    needs: [job1]
    steps:
      - run: echo '${{ toJSON(needs.job1.outputs) }}'
```

## `jobs.<job_id>.env`

A `map` of variables that are available to all steps in the job. You can set variables for the entire workflow or an individual step (see [`env`](#env) and [`jobs.<job_id>.steps[*].env`](#jobsjob_idstepsenv)). The most specific same-named variable wins.

### Example of `jobs.<job_id>.env`

```yaml
jobs:
  job1:
    env:
      FIRST_NAME: Mona
```

## `jobs.<job_id>.defaults`

Use `jobs.<job_id>.defaults` to create a `map` of default settings that will apply to all steps in the job. You can also set default settings for the entire workflow (see [`defaults`](#defaults)). The most specific default setting wins.

## `jobs.<job_id>.defaults.run`

Use `jobs.<job_id>.defaults.run` to provide default `shell` and `working-directory` to all `run` steps in the job. You can also set defaults for `run` for the entire workflow (see [`defaults.run`](#defaultsrun)). These can be overridden at the `jobs.<job_id>.defaults.run` and `jobs.<job_id>.steps[*].run` levels.

## `jobs.<job_id>.defaults.run.shell`

Use `shell` to define the default shell for `run` steps in the job. See the supported-shells table under [`jobs.<job_id>.steps[*].shell`](#jobsjob_idstepsshell).

## `jobs.<job_id>.defaults.run.working-directory`

Use `working-directory` to define the default working directory for `run` steps in the job. Ensure the directory exists on the runner.

### Example: Setting default `run` step options for a job

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash
        working-directory: ./scripts
```

## `jobs.<job_id>.steps`

A job contains a sequence of tasks called `steps`. Steps can run commands, run setup tasks, or run an action in your repository, a public repository, or an action published in a Docker registry. Not all steps run actions, but all actions run as a step. Each step runs in its own process in the runner environment and has access to the workspace and filesystem. Because steps run in their own process, changes to environment variables are not preserved between steps. GitHub provides built-in steps to set up and complete a job.

GitHub only displays the first 1,000 checks, however, you can run an unlimited number of steps as long as you are within the workflow usage limits.

### Example of `jobs.<job_id>.steps`

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

## `jobs.<job_id>.steps[*].id`

A unique identifier for the step. You can use the `id` to reference the step in contexts.

## `jobs.<job_id>.steps[*].if`

You can use the `if` conditional to prevent a step from running unless a condition is met. You can use any supported context and expression. You can optionally omit the `${{ }}` syntax (GitHub Actions evaluates `if` as an expression), except when the expression starts with `!`.

### Example: Using contexts

This step only runs when the event type is a `pull_request` and the event action is `unassigned`.

```yaml
steps:
  - name: My first step
    if: ${{ github.event_name == 'pull_request' && github.event.action == 'unassigned' }}
    run: echo This event is a pull request that had an assignee removed.
```

### Example: Using status check functions

The `my backup step` only runs when the previous step of a job fails.

```yaml
steps:
  - name: My first step
    uses: octo-org/action-name@main
  - name: My backup step
    if: ${{ failure() }}
    uses: actions/heroku@1.0.0
```

### Example: Using secrets

Secrets cannot be directly referenced in `if:` conditionals. Instead, set secrets as job-level environment variables, then reference the environment variables to conditionally run steps. If a secret has not been set, an expression referencing it returns an empty string.

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

## `jobs.<job_id>.steps[*].name`

A name for your step to display on GitHub.

## `jobs.<job_id>.steps[*].uses`

Selects an action to run as part of a step in your job. An action is a reusable unit of code. You can use an action defined in the same repository as the workflow, a public repository, or in a published Docker container image.

We strongly recommend that you include the version of the action you are using by specifying a Git ref, SHA, or Docker tag. If you don't specify a version, it could break your workflows or cause unexpected behavior when the action owner publishes an update.

- Using the commit SHA of a released action version is the safest for stability and security.
- If the action publishes major version tags, you should expect to receive critical fixes and security patches while still retaining compatibility (at the author's discretion).
- Using the default branch of an action may be convenient, but a new major version with a breaking change could break your workflow.

Some actions require inputs that you must set using the [`with`](#jobsjob_idstepswith) keyword. Actions are either JavaScript files or Docker containers. If the action is a Docker container you must run the job in a Linux environment (see [`runs-on`](#jobsjob_idruns-on)).

### Example: Using versioned actions

```yaml
steps:
  # Reference a specific commit
  - uses: actions/checkout@8f4b7f84864484a7bf31766abe9204da3cbe65b3
  # Reference the major version of a release
  - uses: actions/checkout@v6
  # Reference a specific version
  - uses: actions/checkout@v6.2.0
  # Reference a branch
  - uses: actions/checkout@main
```

### Example: Using a public action

`{owner}/{repo}@{ref}` — you can specify a branch, ref, or SHA in a public GitHub repository.

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        # Uses the default branch of a public repository
        uses: actions/heroku@main
      - name: My second step
        # Uses a specific version tag of a public repository
        uses: actions/aws@v2.0.1
```

### Example: Using a public action in a subdirectory

`{owner}/{repo}/{path}@{ref}` — a subdirectory in a public GitHub repository at a specific branch, ref, or SHA.

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: actions/aws/ec2@main
```

### Example: Using an action in the same repository as the workflow

`./path/to/dir` — the path to the directory that contains the action in your workflow's repository. You must check out your repository before using the action. The path is relative (`./`) to the default working directory (`github.workspace`, `$GITHUB_WORKSPACE`).

```yaml
jobs:
  my_first_job:
    runs-on: ubuntu-latest
    steps:
      # This step checks out a copy of your repository.
      - name: My first step - check out repository
        uses: actions/checkout@v6
      # This step references the directory that contains the action.
      - name: Use local hello-world-action
        uses: ./.github/actions/hello-world-action
```

### Example: Using a Docker Hub action

`docker://{image}:{tag}` — a Docker image published on [Docker Hub](https://hub.docker.com/).

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: docker://alpine:3.8
```

### Example: Using the GitHub Packages Container registry

`docker://{host}/{image}:{tag}` — a public Docker image in the GitHub Packages Container registry.

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: docker://ghcr.io/OWNER/IMAGE_NAME
```

### Example: Using a Docker public registry action

`docker://{host}/{image}:{tag}` — a Docker image in a public registry. This example uses the Google Container Registry at `gcr.io`.

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: docker://gcr.io/cloud-builders/gradle
```

### Example: Using an action inside a different private repository than the workflow

If the action is in an internal repository, or in a private repository configured to allow access from your workflow's repository, you can reference it directly. If not, check out the repository and reference the action locally: generate a personal access token, add it as a secret, then check out and reference the action. Replace `PERSONAL_ACCESS_TOKEN` with the name of your secret.

```yaml
jobs:
  my_first_job:
    steps:
      - name: Check out repository
        uses: actions/checkout@v6
        with:
          repository: octocat/my-private-repo
          ref: v1.0
          token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
          path: ./.github/actions/my-private-repo
      - name: Run my action
        uses: ./.github/actions/my-private-repo/my-action
```

Alternatively, use a GitHub App instead of a personal access token so your workflow continues to run even if the token owner leaves.

## `jobs.<job_id>.steps[*].run`

Runs command-line programs that do not exceed 21,000 characters using the operating system's shell. If you do not provide a `name`, the step name will default to the text specified in the `run` command. Commands run using non-login shells by default. Each `run` keyword represents a new process and shell in the runner environment. When you provide multi-line commands, each line runs in the same shell.

```yaml
# A single-line command
- name: Install Dependencies
  run: npm install
```

```yaml
# A multi-line command
- name: Clean install dependencies and build
  run: |
    npm ci
    npm run build
```

## `jobs.<job_id>.steps[*].working-directory`

Using the `working-directory` keyword, you can specify the working directory of where to run the command. Alternatively, specify a default working directory for all `run` steps in a job or the whole workflow (see [`defaults.run.working-directory`](#defaultsrunworking-directory) and [`jobs.<job_id>.defaults.run.working-directory`](#jobsjob_iddefaultsrunworking-directory)).

```yaml
- name: Clean temp directory
  run: rm -rf *
  working-directory: ./temp
```

## `jobs.<job_id>.steps[*].shell`

You can override the default shell settings using the `shell` keyword, using built-in keywords or a custom set of options. The shell command run internally executes a temporary file containing the commands specified in `run`. Alternatively, specify a default shell for all `run` steps in a job or the whole workflow (see [`defaults.run.shell`](#defaultsrunshell) and [`jobs.<job_id>.defaults.run.shell`](#jobsjob_iddefaultsrunshell)).

| Supported platform | `shell` parameter | Description | Command run internally |
| --- | --- | --- | --- |
| Linux / macOS | unspecified | The default shell on non-Windows platforms. Runs a different command than when `bash` is specified explicitly. If `bash` is not found in the path, treated as `sh`. | `bash -e {0}` |
| All | `bash` | The default shell on non-Windows with a fallback to `sh`. On Windows uses the bash shell included with Git for Windows. | `bash --noprofile --norc -eo pipefail {0}` |
| All | `pwsh` | PowerShell Core. GitHub appends `.ps1` to your script name. | `pwsh -command ". '{0}'"` |
| All | `python` | Executes the python command. | `python {0}` |
| Linux / macOS | `sh` | Fallback for non-Windows if no shell is provided and `bash` is not found. | `sh -e {0}` |
| Windows | `cmd` | GitHub appends `.cmd` to your script name and substitutes for `{0}`. | `%ComSpec% /D /E:ON /V:OFF /S /C "CALL "{0}""` |
| Windows | `pwsh` | Default shell on Windows. PowerShell Core (`.ps1`). If a self-hosted Windows runner lacks PowerShell Core, PowerShell Desktop is used. | `pwsh -command ". '{0}'"` |
| Windows | `powershell` | PowerShell Desktop. GitHub appends `.ps1` to your script name. | `powershell -command ". '{0}'"` |

### Example: Running a command using Bash

```yaml
steps:
  - name: Display the path
    shell: bash
    run: echo $PATH
```

### Example: Running a command using Windows `cmd`

```yaml
steps:
  - name: Display the path
    shell: cmd
    run: echo %PATH%
```

### Example: Running a command using PowerShell Core

```yaml
steps:
  - name: Display the path
    shell: pwsh
    run: echo ${env:PATH}
```

### Example: Using PowerShell Desktop to run a command

```yaml
steps:
  - name: Display the path
    shell: powershell
    run: echo ${env:PATH}
```

### Example: Running an inline Python script

```yaml
steps:
  - name: Display the path
    shell: python
    run: |
      import os
      print(os.environ['PATH'])
```

### Custom shell

You can set the `shell` value to a template string using `command [options] {0} [more_options]`. GitHub interprets the first whitespace-delimited word of the string as the command, and inserts the file name for the temporary script at `{0}`. The command used (e.g. `perl`) must be installed on the runner.

```yaml
steps:
  - name: Display the environment variables and their values
    shell: perl {0}
    run: |
      print %ENV
```

### Exit codes and error action preference

For built-in shell keywords, GitHub-hosted runners provide the following defaults:

- `bash`/`sh`:
  - By default, fail-fast behavior is enforced using `set -e` for both `sh` and `bash`. When `shell: bash` is specified, `-o pipefail` is also applied.
  - You can take full control by providing a template string, e.g. `bash {0}`.
  - `sh`-like shells exit with the exit code of the last command executed, which is also the default for actions.
- `powershell`/`pwsh`:
  - Fail-fast when possible. `$ErrorActionPreference = 'stop'` is prepended.
  - `if ((Test-Path -LiteralPath variable:\LASTEXITCODE)) { exit $LASTEXITCODE }` is appended so action statuses reflect the script's last exit code.
  - Opt out with a custom shell like `pwsh -File {0}` or `powershell -Command "& '{0}'"`.
- `cmd`:
  - There is no full opt-in to fail-fast other than checking each error code in your script.
  - `cmd.exe` exits with the error level of the last program it executed and returns the error code to the runner.

## `jobs.<job_id>.steps[*].with`

A `map` of the input parameters defined by the action. Each input parameter is a key/value pair. Input parameters are set as environment variables, prefixed with `INPUT_` and converted to upper case. Input parameters defined for a Docker container must use `args`.

### Example of `jobs.<job_id>.steps[*].with`

Defines three input parameters (`first_name`, `middle_name`, `last_name`) for the `hello_world` action, accessible as `INPUT_FIRST_NAME`, `INPUT_MIDDLE_NAME`, and `INPUT_LAST_NAME`.

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

## `jobs.<job_id>.steps[*].with.args`

A `string` that defines the inputs for a Docker container. GitHub passes the `args` to the container's `ENTRYPOINT` when the container starts up. An array of strings is not supported; a single argument that includes spaces should be surrounded by double quotes `""`.

### Example of `jobs.<job_id>.steps[*].with.args`

```yaml
steps:
  - name: Explain why this job ran
    uses: octo-org/action-name@main
    with:
      entrypoint: /bin/echo
      args: The ${{ github.event_name }} event triggered this step.
```

The `args` are used in place of the `CMD` instruction in a `Dockerfile`. If you use `CMD`, ordered by preference: (1) document required arguments in the README and omit them from `CMD`; (2) use defaults that allow using the action without specifying any `args`; (3) if the action exposes a `--help` flag or similar, use that as the default.

## `jobs.<job_id>.steps[*].with.entrypoint`

Overrides the Docker `ENTRYPOINT` in the `Dockerfile`, or sets it if one wasn't already specified. Unlike the Docker `ENTRYPOINT` instruction (shell and exec forms), the `entrypoint` keyword accepts only a single string defining the executable to be run.

### Example of `jobs.<job_id>.steps[*].with.entrypoint`

```yaml
steps:
  - name: Run a custom command
    uses: octo-org/action-name@main
    with:
      entrypoint: /a/different/executable
```

The `entrypoint` keyword is meant to be used with Docker container actions, but you can also use it with JavaScript actions that don't define any inputs.

## `jobs.<job_id>.steps[*].env`

Sets variables for steps to use in the runner environment. You can also set variables for the entire workflow or a job (see [`env`](#env) and [`jobs.<job_id>.env`](#jobsjob_idenv)). Public actions may specify expected variables in the README. For secrets/sensitive values (passwords, tokens), you must set them using the `secrets` context.

### Example of `jobs.<job_id>.steps[*].env`

```yaml
steps:
  - name: My first action
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      FIRST_NAME: Mona
      LAST_NAME: Octocat
```

## `jobs.<job_id>.steps[*].continue-on-error`

Prevents a job from failing when a step fails. Set to `true` to allow a job to pass when this step fails.

## `jobs.<job_id>.steps[*].timeout-minutes`

The maximum number of minutes to run the step before killing the process. **Maximum: 360** for both GitHub-hosted and self-hosted runners. Fractional values are not supported; `timeout-minutes` must be a positive integer.

## `jobs.<job_id>.steps[*].background`

Runs a step asynchronously so the job continues to the next step without waiting for it to finish. Use `background: true` for long-running processes (databases, servers, monitoring tasks). Synchronize with background steps later using [`wait`](#jobsjob_idstepswait)/[`wait-all`](#jobsjob_idstepswait-all), or stop them with [`cancel`](#jobsjob_idstepscancel).

You can use `background` on steps that use `run` or `uses`. To reference a background step from `wait`/`cancel`, give it an [`id`](#jobsjob_idstepsid). A maximum of **10 background steps** can run concurrently in a single job; additional ones are queued. Outputs and environment changes are only available after a `wait`/`wait-all` step that includes it. If a background step fails, the job fails at the next `wait`/`wait-all` that includes it (unless `continue-on-error` is set on that step). An implicit `wait-all` runs before any post-job cleanup.

> You cannot use `background` on steps inside a composite action. A composite action can itself run as a background step, but it cannot declare background steps internally.

### Example: Running a step in the background

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

## `jobs.<job_id>.steps[*].wait`

Pauses the job until one or more background steps complete. A `wait` step performs no work itself; it only blocks until the referenced background steps finish. Provide a single step `id` as a string, or multiple `id`s as an array. After a `wait` step completes, the outputs of the referenced background steps become available to subsequent steps. If a referenced background step failed, the `wait` step fails too.

> A `wait` step always runs and does not support the [`if`](#jobsjob_idstepsif) conditional.

### Example: Waiting for specific background steps

```yaml
steps:
  - name: Build frontend
    id: build-frontend
    run: npm run build:frontend
    background: true

  - name: Build backend
    id: build-backend
    run: npm run build:backend
    background: true

  - name: Run linter while builds run
    run: npm run lint

  - name: Wait for both builds to finish
    wait: [build-frontend, build-backend]

  - name: Run tests
    run: npm test
```

## `jobs.<job_id>.steps[*].wait-all`

Pauses the job until all active background steps complete. Like `wait`, the `wait-all` step fails if any of the background steps it waits on failed, unless you set [`continue-on-error`](#jobsjob_idstepscontinue-on-error) to `true`. The `wait-all` keyword takes no arguments.

> A `wait-all` step always runs and does not support the `if` conditional.

### Example: Waiting for all background steps

```yaml
steps:
  - name: Start database
    id: db
    run: docker run -d postgres:15
    background: true

  - name: Start cache
    id: cache
    run: docker run -d redis:7
    background: true

  - name: Run integration tests
    run: npm run test:integration

  - name: Wait for all services to stop
    wait-all:
```

## `jobs.<job_id>.steps[*].cancel`

Gracefully terminates a running background step. The runner sends the step's process a termination signal (`SIGTERM`) so it can clean up, and forcibly stops it (`SIGKILL`) if it does not exit within a short grace period. The `cancel` keyword targets a single background step by its `id`.

> A `cancel` step always runs and does not support the `if` conditional.

### Example: Canceling a background step

```yaml
steps:
  - name: Start long-running monitor
    id: monitor
    run: ./scripts/monitor.sh
    background: true

  - name: Run the main task
    run: npm test

  - name: Stop the monitor
    cancel: monitor
```

## `jobs.<job_id>.steps[*].parallel`

Runs a group of steps concurrently, then waits for all of them to finish before continuing. The `parallel` keyword is shorthand: every step in the group runs as a background step, with an implicit `wait` at the end. Use it for an independent group of steps that can run at the same time and that you don't need to reference individually; use [`background`](#jobsjob_idstepsbackground) when you need finer control. Each step in the group is subject to the same 10-step concurrency limit.

> You cannot use `parallel` inside a composite action.

### Example: Running steps in parallel

```yaml
steps:
  - uses: actions/checkout@v6

  - parallel:
      - name: Build frontend
        run: npm run build:frontend

      - name: Build backend
        run: npm run build:backend

      - name: Build docs
        run: npm run build:docs

  - name: Run tests after all builds complete
    run: npm test
```

The group above is equivalent to declaring each step with `background: true` followed by a `wait` step.

## `jobs.<job_id>.timeout-minutes`

The maximum number of minutes to let a job run before GitHub automatically cancels it. **Default: 360.** If the timeout exceeds the job execution time limit for the runner, the job will be canceled when the execution time limit is met instead.

> The `GITHUB_TOKEN` expires when a job finishes or after a maximum of 24 hours. For self-hosted runners, the token may be the limiting factor if the job timeout is greater than 24 hours.

## `jobs.<job_id>.strategy`

Use `jobs.<job_id>.strategy` to use a matrix strategy for your jobs. A matrix strategy lets you use variables in a single job definition to automatically create multiple job runs based on the combinations of the variables (for example, to test your code in multiple versions of a language or on multiple operating systems).

## `jobs.<job_id>.strategy.matrix`

Use `jobs.<job_id>.strategy.matrix` to define a matrix of different job configurations. A matrix will generate a maximum of **256 jobs per workflow run** (GitHub-hosted and self-hosted). The variables you define become properties in the `matrix` context and can be referenced elsewhere in the workflow. By default GitHub maximizes the number of parallel jobs by runner availability; the order of variables determines the order in which jobs are created (the first variable is the first job created).

### Using a single-dimension matrix

Defines `version` with `[10, 12, 14]`, running three jobs — one per value — accessed through `matrix.version`.

```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        version: [10, 12, 14]
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.version }}
```

### Using a multi-dimensional matrix

Specify multiple variables to create a multi-dimensional matrix; a job runs for each possible combination. Two OSes × three versions = six jobs.

```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        os: [ubuntu-22.04, ubuntu-24.04]
        version: [10, 12, 14]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.version }}
```

A variable configuration can be an `array` of `object`s. The following matrix produces 4 jobs.

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

Each job has its own combination of `os` and `node`:

```yaml
- matrix.os: ubuntu-latest
  matrix.node.version: 14
- matrix.os: ubuntu-latest
  matrix.node.version: 20
  matrix.node.env: NODE_OPTIONS=--openssl-legacy-provider
- matrix.os: macos-latest
  matrix.node.version: 14
- matrix.os: macos-latest
  matrix.node.version: 20
  matrix.node.env: NODE_OPTIONS=--openssl-legacy-provider
```

## `jobs.<job_id>.strategy.matrix.include`

For each object in the `include` list, the key:value pairs are added to each matrix combination if none of them overwrite the original matrix values. If an object cannot be added to any combination, a new combination is created instead. Original matrix values are never overwritten, but added values can be overwritten.

### Example: Expanding configurations

Runs four jobs (one per `os`/`node` combination); when `os` is `windows-latest` and `node` is `16`, an additional variable `npm` with value `6` is included.

```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        os: [windows-latest, ubuntu-latest]
        node: [14, 16]
        include:
          - os: windows-latest
            node: 16
            npm: 6
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - if: ${{ matrix.npm }}
        run: npm install -g npm@${{ matrix.npm }}
      - run: npm --version
```

### Example: Adding configurations

Runs 10 jobs (one per `os`/`version` combination) plus a job for `os: windows-latest`, `version: 17`.

```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        os: [macos-latest, windows-latest, ubuntu-latest]
        version: [12, 14, 16]
        include:
          - os: windows-latest
            version: 17
```

If you don't specify any matrix variables, all configurations under `include` run. The following runs two jobs, one per `include` entry.

```yaml
jobs:
  includes_only:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - site: "production"
            datacenter: "site-a"
          - site: "staging"
            datacenter: "site-b"
```

## `jobs.<job_id>.strategy.matrix.exclude`

An excluded configuration only has to be a partial match for it to be excluded. All `include` combinations are processed after `exclude`, which allows you to use `include` to add back combinations that were previously excluded.

## `jobs.<job_id>.strategy.fail-fast`

You can control how job failures are handled with `jobs.<job_id>.strategy.fail-fast` and `jobs.<job_id>.continue-on-error`.

`jobs.<job_id>.strategy.fail-fast` applies to the entire matrix. If it is set to `true` or evaluates to `true`, GitHub cancels all in-progress and queued jobs in the matrix if any job in the matrix fails. This property **defaults to `true`**.

`jobs.<job_id>.continue-on-error` applies to a single job. If `true`, other jobs in the matrix continue running even if that job fails.

```yaml
# Four jobs; continue-on-error per matrix.experimental. If a non-experimental job
# fails, in-progress/queued jobs are cancelled; the experimental failure is isolated.
jobs:
  test:
    runs-on: ubuntu-latest
    continue-on-error: ${{ matrix.experimental }}
    strategy:
      fail-fast: true
      matrix:
        version: [6, 7, 8]
        experimental: [false]
        include:
          - version: 9
            experimental: true
```

## `jobs.<job_id>.strategy.max-parallel`

By default, GitHub will maximize the number of jobs run in parallel depending on runner availability. Use `max-parallel` to set the maximum number of jobs that can run simultaneously when using a matrix.

## `jobs.<job_id>.continue-on-error`

`jobs.<job_id>.continue-on-error` applies to a single job. If it is `true`, other jobs in the matrix will continue running even if the job with `continue-on-error: true` fails. Prevents a workflow run from failing when a job fails — set to `true` to allow a workflow run to pass when this job fails.

### Example: Preventing a specific failing matrix job from failing a workflow run

You can allow specific jobs in a job matrix to fail without failing the workflow run — for example, an experimental job with `node` set to `15`.

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

## `jobs.<job_id>.container`

> If your workflows use Docker container actions, job containers, or service containers, you must use a Linux runner: with GitHub-hosted runners you must use an Ubuntu runner; with self-hosted runners you must use a Linux machine with Docker installed.

Use `jobs.<job_id>.container` to create a container to run any steps in a job that don't already specify a container. If you have steps that use both script and container actions, the container actions run as sibling containers on the same network with the same volume mounts. If you do not set a `container`, all steps run directly on the host specified by `runs-on` unless a step refers to an action configured to run in a container.

> The default shell for `run` steps inside a container is `sh` instead of `bash`. Override with [`jobs.<job_id>.defaults.run`](#jobsjob_iddefaultsrun) or [`jobs.<job_id>.steps[*].shell`](#jobsjob_idstepsshell).

```yaml
name: CI
on:
  push:
    branches: [ main ]
jobs:
  container-test-job:
    runs-on: ubuntu-latest
    container:
      image: node:18
      env:
        NODE_ENV: development
      ports:
        - 80
      volumes:
        - my_docker_volume:/volume_mount
      options: --cpus 1
    steps:
      - name: Check for dockerenv file
        run: (ls /.dockerenv && echo Found dockerenv) || (echo No dockerenv)
```

When you only specify a container image, you can omit the `image` keyword:

```yaml
jobs:
  container-test-job:
    runs-on: ubuntu-latest
    container: node:18
```

## `jobs.<job_id>.container.image`

Use `jobs.<job_id>.container.image` to define the Docker image to use as the container to run the action. The value can be the Docker Hub image name or a registry name.

## `jobs.<job_id>.container.credentials`

If the image's container registry requires authentication to pull the image, use `jobs.<job_id>.container.credentials` to set a `map` of the `username` and `password`. The credentials are the same values you would provide to the `docker login` command.

```yaml
container:
  image: ghcr.io/owner/image
  credentials:
     username: ${{ github.actor }}
     password: ${{ secrets.github_token }}
```

## `jobs.<job_id>.container.env`

Use `jobs.<job_id>.container.env` to set a `map` of environment variables in the container.

## `jobs.<job_id>.container.ports`

Use `jobs.<job_id>.container.ports` to set an `array` of ports to expose on the container.

## `jobs.<job_id>.container.volumes`

Use `jobs.<job_id>.container.volumes` to set an `array` of volumes for the container to use. You can use volumes to share data between services or other steps in a job. You can specify named Docker volumes, anonymous Docker volumes, or bind mounts on the host. Specify a volume as `<source>:<destinationPath>`, where `<source>` is a volume name or absolute host path and `<destinationPath>` is an absolute path in the container.

```yaml
volumes:
  - my_docker_volume:/volume_mount
  - /data/my_data
  - /source/directory:/destination/directory
```

## `jobs.<job_id>.container.options`

Use `jobs.<job_id>.container.options` to configure additional Docker container resource options. For a list of options, see [`docker create` options](https://docs.docker.com/engine/reference/commandline/create/#options).

> The `--network` and `--entrypoint` options are not supported.

## `jobs.<job_id>.services`

> Container/service OS support: you must use a Linux runner (Ubuntu on GitHub-hosted runners; a Linux machine with Docker on self-hosted runners).

Used to host service containers for a job in a workflow. Service containers are useful for creating databases or cache services like Redis. The runner automatically creates a Docker network and manages the life cycle of the service containers.

If you configure your job to run in a container, or your step uses container actions, you don't need to map ports to access the service or action — Docker exposes all ports between containers on the same user-defined bridge network, and you reference the service container by its hostname (mapped to the label name you configure). If the job runs directly on the runner machine and your step doesn't use a container action, you must map any required Docker service container ports to the Docker host and access the service using localhost and the mapped port.

### Example: Using localhost

Creates two services: nginx and redis. When you specify the container port but not the host port, the container port is randomly assigned to a free host port, which GitHub sets in the `${{job.services.<service_name>.ports}}` context.

```yaml
services:
  nginx:
    image: nginx
    # Map port 8080 on the Docker host to port 80 on the nginx container
    ports:
      - 8080:80
  redis:
    image: redis
    # Map random free TCP port on Docker host to port 6379 on redis container
    ports:
      - 6379/tcp
steps:
  - run: |
      echo "Redis available on 127.0.0.1:${{ job.services.redis.ports['6379'] }}"
      echo "Nginx available on 127.0.0.1:${{ job.services.nginx.ports['80'] }}"
```

## `jobs.<job_id>.services.<service_id>.image`

The Docker image to use as the service container to run the action. The value can be the Docker Hub image name or a registry name. If assigned an empty string, the service will not start — useful for conditional services.

```yaml
services:
  nginx:
    image: ${{ options.nginx == true && 'nginx' || '' }}
```

## `jobs.<job_id>.services.<service_id>.credentials`

If the image's container registry requires authentication to pull the image, use `credentials` to set a `map` of the `username` and `password` (the same values you would provide to `docker login`).

### Example of `jobs.<job_id>.services.<service_id>.credentials`

```yaml
services:
  myservice1:
    image: ghcr.io/owner/myservice1
    credentials:
      username: ${{ github.actor }}
      password: ${{ secrets.github_token }}
  myservice2:
    image: dockerhub_org/myservice2
    credentials:
      username: ${{ secrets.DOCKER_USER }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

## `jobs.<job_id>.services.<service_id>.env`

Sets a `map` of environment variables in the service container.

## `jobs.<job_id>.services.<service_id>.ports`

Sets an `array` of ports to expose on the service container.

## `jobs.<job_id>.services.<service_id>.volumes`

Sets an `array` of volumes for the service container to use. You can use volumes to share data between services or other steps in a job. Specify a volume as `<source>:<destinationPath>`, where `<source>` is a volume name or absolute host path and `<destinationPath>` is an absolute container path.

### Example of `jobs.<job_id>.services.<service_id>.volumes`

```yaml
volumes:
  - my_docker_volume:/volume_mount
  - /data/my_data
  - /source/directory:/destination/directory
```

## `jobs.<job_id>.services.<service_id>.options`

Additional Docker container resource options. For a list of options, see [`docker create` options](https://docs.docker.com/engine/reference/commandline/create/#options).

> The `--network` option is not supported.

## `jobs.<job_id>.services.<service_id>.command`

Overrides the Docker image's default command (`CMD`). The value is passed as arguments after the image name in the `docker create` command. If you also specify `entrypoint`, `command` provides the arguments to that entrypoint.

### Example of `jobs.<job_id>.services.<service_id>.command`

```yaml
services:
  mysql:
    image: mysql:8
    command: --sql_mode=STRICT_TRANS_TABLES --max_allowed_packet=512M
    env:
      MYSQL_ROOT_PASSWORD: test
    ports:
      - 3306:3306
```

## `jobs.<job_id>.services.<service_id>.entrypoint`

Overrides the Docker image's default `ENTRYPOINT`. The value is a single string defining the executable to run. Use this when you need to replace the image's entrypoint entirely. You can combine `entrypoint` with `command` to pass arguments to the custom entrypoint.

### Example of `jobs.<job_id>.services.<service_id>.entrypoint`

```yaml
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.17
    entrypoint: etcd
    command: >-
      --listen-client-urls http://0.0.0.0:2379
      --advertise-client-urls http://0.0.0.0:2379
    ports:
      - 2379:2379
```

## `jobs.<job_id>.uses`

The location and version of a reusable workflow file to run as a job. Use one of the following syntaxes:

- `{owner}/{repo}/.github/workflows/{filename}@{ref}` for reusable workflows in public and private repositories.
- `./.github/workflows/{filename}` for reusable workflows in the same repository.

In the first option, `{ref}` can be a SHA, a release tag, or a branch name. If a release tag and a branch have the same name, the release tag takes precedence. Using the commit SHA is the safest option for stability and security. In the second option (without `{owner}/{repo}` and `@{ref}`) the called workflow is from the same commit as the caller workflow; ref prefixes such as `refs/heads` and `refs/tags` are not allowed, and you cannot use contexts or expressions in this keyword.

### Example of `jobs.<job_id>.uses`

```yaml
jobs:
  call-workflow-1-in-local-repo:
    uses: octo-org/this-repo/.github/workflows/workflow-1.yml@172239021f7ba04fe7327647b213799853a9eb89
  call-workflow-2-in-local-repo:
    uses: ./.github/workflows/workflow-2.yml
  call-workflow-in-another-repo:
    uses: octo-org/another-repo/.github/workflows/workflow.yml@v1
```

## `jobs.<job_id>.with`

When a job is used to call a reusable workflow, you can use `with` to provide a map of inputs that are passed to the called workflow. Any inputs you pass must match the input specifications defined in the called workflow. Unlike [`jobs.<job_id>.steps[*].with`](#jobsjob_idstepswith), the inputs you pass with `jobs.<job_id>.with` are not available as environment variables in the called workflow — reference them using the `inputs` context.

### Example of `jobs.<job_id>.with`

```yaml
jobs:
  call-workflow:
    uses: octo-org/example-repo/.github/workflows/called-workflow.yml@main
    with:
      username: mona
```

## `jobs.<job_id>.with.<input_id>`

A pair consisting of a string identifier for the input and the value of the input. The identifier must match the name of an input defined by `on.workflow_call.inputs.<input_id>` in the called workflow. The data type of the value must match the type defined by [`on.workflow_call.inputs.<input_id>.type`](#onworkflow_callinputsinput_idtype). Allowed expression contexts: `github`, and `needs`.

## `jobs.<job_id>.secrets`

When a job is used to call a reusable workflow, you can use `secrets` to provide a map of secrets that are passed to the called workflow. Any secrets you pass must match the names defined in the called workflow.

### Example of `jobs.<job_id>.secrets`

```yaml
jobs:
  call-workflow:
    uses: octo-org/example-repo/.github/workflows/called-workflow.yml@main
    secrets:
      access-token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
```

## `jobs.<job_id>.secrets.inherit`

Use the `inherit` keyword to pass all the calling workflow's secrets to the called workflow. This includes all secrets the calling workflow has access to, namely organization, repository, and environment secrets. The `inherit` keyword can be used to pass secrets across repositories within the same organization, or across organizations within the same enterprise.

### Example of `jobs.<job_id>.secrets.inherit`

```yaml
on:
  workflow_dispatch:

jobs:
  pass-secrets-to-workflow:
    uses: ./.github/workflows/called-workflow.yml
    secrets: inherit
```

```yaml
on:
  workflow_call:

jobs:
  pass-secret-to-action:
    runs-on: ubuntu-latest
    steps:
      - name: Use a repo or org secret from the calling workflow.
        run: echo ${{ secrets.CALLING_WORKFLOW_SECRET }}
```

## `jobs.<job_id>.secrets.<secret_id>`

A pair consisting of a string identifier for the secret and the value of the secret. The identifier must match the name of a secret defined by [`on.workflow_call.secrets.<secret_id>`](#onworkflow_callsecretssecret_id) in the called workflow. Allowed expression contexts: `github`, `needs`, and `secrets`.

## Filter pattern cheat sheet

You can use special characters in path, branch, and tag filters.

- `*`: Matches zero or more characters, but does not match the `/` character. For example, `Octo*` matches `Octocat`.
- `**`: Matches zero or more of any character.
- `?`: Matches zero or one of the preceding character.
- `+`: Matches one or more of the preceding character.
- `[]`: Matches one alphanumeric character listed in the brackets or included in ranges. Ranges can only include `a-z`, `A-Z`, and `0-9`. For example, the range `[0-9a-z]` matches any digit or lowercase letter. `[CB]at` matches `Cat` or `Bat`, and `[1-2]00` matches `100` and `200`.
- `!`: At the start of a pattern makes it negate previous positive patterns. It has no special meaning if not the first character.

The characters `*`, `[`, and `!` are special characters in YAML. If you start a pattern with `*`, `[`, or `!`, you must enclose the pattern in quotes. Also, if you use a [flow sequence](https://yaml.org/spec/1.2.2/#flow-sequences) with a pattern containing `[` and/or `]`, the pattern must be enclosed in quotes.

```yaml
# Valid
paths:
  - '**/README.md'

# Invalid - creates a parse error that
# prevents your workflow from running.
paths:
  - **/README.md

# Valid
branches: [ main, 'release/v[0-9].[0-9]' ]

# Invalid - creates a parse error
branches: [ main, release/v[0-9].[0-9] ]
```

### Patterns to match branches and tags

| Pattern | Description | Example matches |
| --- | --- | --- |
| `feature/*` | The `*` wildcard matches any character, but does not match slash (`/`). | `feature/my-branch`, `feature/your-branch` |
| `feature/**` | The `**` wildcard matches any character including slash (`/`) in branch and tag names. | `feature/beta-a/my-branch`, `feature/your-branch`, `feature/mona/the/octocat` |
| `main`, `releases/mona-the-octocat` | Matches the exact name of a branch or tag name. | `main`, `releases/mona-the-octocat` |
| `'*'` | Matches all branch and tag names that don't contain a slash (`/`). The `*` character is special in YAML; when you start a pattern with `*`, you must use quotes. | `main`, `releases` |
| `'**'` | Matches all branch and tag names. This is the default behavior when you don't use a `branches` or `tags` filter. | `all/the/branches`, `every/tag` |
| `'*feature'` | The `*` character is special in YAML; when you start a pattern with `*`, you must use quotes. | `mona-feature`, `feature`, `ver-10-feature` |
| `v2*` | Matches branch and tag names that start with `v2`. | `v2`, `v2.0`, `v2.9` |
| `v[12].[0-9]+.[0-9]+` | Matches all semantic versioning branches and tags with major version 1 or 2. | `v1.10.1`, `v2.0.0` |

### Patterns to match file paths

Path patterns must match the whole path, and start from the repository's root.

| Pattern | Description of matches | Example matches |
| --- | --- | --- |
| `'*'` | The `*` wildcard matches any character, but does not match slash (`/`). Must be quoted when a pattern starts with `*`. | `README.md`, `server.rb` |
| `'*.jsx?'` | The `?` character matches zero or one of the preceding character. | `page.js`, `page.jsx` |
| `'**'` | The `**` wildcard matches any character including slash (`/`). Default behavior when you don't use a `path` filter. | `all/the/files.md` |
| `'*.js'` | The `*` wildcard matches any character, but does not match slash (`/`). Matches all `.js` files at the root of the repository. | `app.js`, `index.js` |
| `'**.js'` | Matches all `.js` files in the repository. | `index.js`, `js/index.js`, `src/js/app.js` |
| `docs/*` | All files within the root of the `docs` directory only, at the root of the repository. | `docs/README.md`, `docs/file.txt` |
| `docs/**` | Any files in the `docs` directory and its subdirectories at the root of the repository. | `docs/README.md`, `docs/mona/octocat.txt` |
| `docs/**/*.md` | A file with a `.md` suffix anywhere in the `docs` directory. | `docs/README.md`, `docs/mona/hello-world.md`, `docs/a/markdown/file.md` |
| `'**/docs/**'` | Any files in a `docs` directory anywhere in the repository. | `docs/hello.md`, `dir/docs/my-file.txt`, `space/docs/plan/space.doc` |
| `'**/README.md'` | A README.md file anywhere in the repository. | `README.md`, `js/README.md` |
| `'**/*src/**'` | Any file in a folder with a `src` suffix anywhere in the repository. | `a/src/app.js`, `my-src/code/js/app.js` |
| `'**/*-post.md'` | A file with the suffix `-post.md` anywhere in the repository. | `my-post.md`, `path/their-post.md` |
| `'**/migrate-*.sql'` | A file with the prefix `migrate-` and suffix `.sql` anywhere in the repository. | `migrate-10909.sql`, `db/migrate-v1.0.sql`, `db/sept/migrate-v1.sql` |
| `'*.md'`, `'!README.md'` | Using `!` in front of a pattern negates it. When a file matches a pattern and also matches a negative pattern defined later, the file will not be included. | `hello.md` (not `README.md`, not `docs/hello.md`) |
| `'*.md'`, `'!README.md'`, `README*` | Patterns are checked sequentially. A pattern that negates a previous pattern will re-include file paths. | `hello.md`, `README.md`, `README.doc` |
