# Hardware-Actions

A collection of (somewhat) reusable actions and workflows for my hardware related projects.

## Actions

### comment_remover

Delete a PR comment starting with some magic header.

Example usage:

```yaml
    - name: Delete previous comment
      if: steps.committer_check.outputs.association == 'true'
      uses: Tai-Min/Hardware-Actions/.github/actions/comment_remover@master
      with:
        pr_number: ${{ github.event.pull_request.number }}
        repo_token: ${{ secrets.GITHUB_TOKEN }}
        magic_header: "<!-- TEST_RUNNER -->"
```

### committer_check

Check whether PR HEAD committer is a collaborator. \
Will set the `outputs.association` to true if so.

Example usage:

```yaml
    - name: Committer check
      id: committer_check
      uses: Tai-Min/Hardware-Actions/.github/actions/committer_check@master
      with:
        repo: ${{ github.repository }}
        pr_number: ${{ github.event.pull_request.number }}
        repo_token: ${{ secrets.GITHUB_TOKEN }}
```

## Workflows

### run_tests_label_remover

Remove `run_tests` label from the PR.

Example usage:

```yaml
on:
  pull_request_target:
    types: [ labeled ]

jobs:
  run_tests_label_remover:
    permissions:
      contents: read
      pull-requests: write
    uses: Tai-Min/Hardware-Actions/.github/workflows/ci_label_remover.yml@master
    with:
      label: ${{ github.event.label.name }}
```

### create_hw_package

Create hardware package. This job expects repo structure to be:

* root
  * hw
    * kicad-project1
    * kicad-project2

This job will pack all KiCad files from a project folder, as well as fabrication files, schematics and PCB PDFs then create an issue with result, link to zip archive and further instructions to create release draft.

Example usage:

```yaml
name: Create hardware package

on:
  workflow_dispatch:
    inputs:
      name:
        description: Name of the hardware release
        required: true
        type: string
      sha:
        description: Commit sha
        required: true
        type: string
      projects:
        description: Comma separated project names # i.e: kicad-project1 from above
        required: true
        type: string

jobs:
  release_creator:
    permissions:
      contents: read
      issues: write 
    uses: Tai-Min/Hardware-Actions/.github/workflows/create_hw_package.yml@master
    with:
      name: ${{ inputs.name }}
      sha: ${{ inputs.sha }}
      projects: ${{ inputs.projects }}
```

### handle_skipped

Handle skipped CI job by setting `outputs.skipped` flag. Useful on jobs where some output is expected. \
Instead, add a conditional to this job and inverse of it to handle_skipped to set `outputs.skipped` for further usage.

Example usage:

```yaml

on:
  workflow_call:
    outputs:
      skipped:
        description: true if skipped, empty otherwise
        value: ${{ jobs.handle_skipped.outputs.skipped }}

jobs:
    skippable_job: # Some job that can be skipped
        if: false
    handle_skipped: # Mark skipped if so
        if: true
        uses: Tai-Min/Hardware-Actions/.github/workflows/handle_skipped.yml@master
```

```yaml
# Some job that depends on some_job
jobs:
    some_job:
        uses: The-Job-Above
    post_comment:
        needs: [some_job]
        steps:
        -   name: Post comment
            run: |
                if [[ "${{ needs.some_job.outputs.skipped }}" = 'true' ]]; then
                    COMMENT="Skipped."
                fi
```

### hardware_checks

Perform hardware checks on PR files if needed. Generate DRC, ERC reports, PDF schematics and PCBs then:

* push as artifacts
* create remote branch <pr_branch>-PR<pr_number>-AUTO
* fill outputs.comment with markdown if *committer_check* succeeds
* sets outputs.skipped to true if *committer_check* fails

Use with *stale_branch_remover*

Example usage:

```yaml
name: CI

on:
  pull_request_target:
    types: [ labeled ]

  hardware_checks:
    if: ${{ github.event.label.name == 'run_tests' }}
    permissions:
      pull-requests: write
      contents: write
    uses: Tai-Min/Hardware-Actions/.github/workflows/hardware_checks.yml@master
    with:
      head_ref: ${{ github.head_ref }}
      pr_number: ${{ github.event.pull_request.number }}
      repo_name: ${{ github.event.pull_request.head.repo.full_name }}
```

### pr_alerts

Post a comment to a PR if `.github` folder was changed and fails the check.

Example usage:

```yaml
name: On pull request

on:
  pull_request_target:

jobs:
  pr_alerts:
    permissions:
      contents: read
      pull-requests: write
    uses: Tai-Min/Hardware-Actions/.github/workflows/pr_alerts.yml@master
```

### pr_labeler

Label PR using this repo's labeler.yml logic.

Example usage:

```yaml
name: On pull request

on:
  pull_request_target:

jobs:
  pr_labeler:
    permissions:
      contents: read
      pull-requests: write
      issues: write
    uses: Tai-Min/Hardware-Actions/.github/workflows/pr_labeler.yml@master
```

### pr_rule_enforcer

Enforce:

* branch name starting with:
  * repo/
  * docs/
  * bug/
  * devel/
  * feature/
* signoff on commits

Post comment and fail if there are problems.

Example usage:

```yaml
name: On pull request

on:
  pull_request_target:

jobs:
  pr_rule_enforcer:
    permissions:
      contents: read
      pull-requests: write
    uses: Tai-Min/Hardware-Actions/.github/workflows/pr_rule_enforcer.yml@master
```

### release_publisher

Continuation of *create_hw_package*. This is the workflow that is triggered on `/release` command that is described by the bot that created the HW package's issue. \
This will create release draft.

Example usage:

```yaml
name: On issue commented

on:
  issue_comment:
    types: [created]

jobs:
  release_publisher:
    permissions:
      issues: write
      contents: write
      actions: read
    uses: Tai-Min/Hardware-Actions/.github/workflows/release_publisher.yml@master
    with:
      comment_id: ${{ github.event.comment.id }}
      comment_body: ${{ github.event.comment.body }}
      issue_number: ${{ github.event.issue.number }}
      is_pull_request: ${{ github.event.issue.pull_request != null }} # This shall be unchanged
    
```

### stale_branch_remower

This will remove stale branches created by *hardware_checks*.

Example usage:

```yaml
name: Cron jobs

on:
  schedule:
    - cron: "0 0 * * *" # Everday at midnight

jobs:
  # Remove stale branches created by actions
  stale_branch_remover:
      uses: Tai-Min/Hardware-Actions/.github/workflows/stale_branch_remover.yml@master
      permissions:
        actions: read
        pull-requests: read
        contents: write
```
