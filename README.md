# Hardware-Actions

A collection of (somewhat) reusable actions and workflows for my hardware related projects

## Actions

### comment_remover

### committer_check

## Workflows

### ci_label_remover
Remove `run_tests` label from the PR.

Example usage:
```yaml
on:
  pull_request_target:
    types: [ labeled ]

jobs:
  # Remove label that triggers the CI
  ci_label_remover:
    permissions:
      contents: read
      pull-requests: write

    uses: Tai-Min/Hardware-Actions/.github/workflows/ci_label_remover.yml@master
    with:
      label: ${{ github.event.label.name }}
```

### create_hw_release

### handle_skipped
Handle skipped CI job by setting outputs.skipped flag. Useful when  

Example usage:
```yaml
# Some job that can be skipped
on:
  workflow_call:
    outputs:
      skipped:
        description: true if skipped, empty otherwise
        value: ${{ jobs.handle_skipped.outputs.skipped }}

jobs:
    some_job:
        if: false
    handle_skipped:
        if: true
        uses: Tai-Min/Hardware-Actions/.github/workflows/handle_skipped.yml@master
```

```yaml
# Some job that depends on job above
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

### pr_alerts

### pr_external_commenter

### pr_labeler

### pr_rule_enforcer

### release_publisher

### stale_branch_remower

