# Shared community files and workflows

This public repository provides default community files for repositories owned by `sharafutdinovdi`.
The [profile README](profile/README.md) lists the open-source projects and roadmap.

## Community files

The code of conduct, security policy, support guide, issue forms and PR template apply when a repository has no local equivalent.
Project-specific files take precedence, including each project's existing security policy.
A local `.github/ISSUE_TEMPLATE` directory with a valid template or `config.yml` replaces the entire inherited template set.
The defaults are not copied into project repositories.

GitHub also supports inherited `CONTRIBUTING.md` and `FUNDING.yml`; this repository does not provide those files.
Contribution instructions remain project-specific.
Workflows, labeler configuration, Dependabot configuration and CODEOWNERS are not inherited.
See [GitHub's default community file rules](https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file).

Issue forms require the `bug` and `enhancement` labels in each receiving repository.
The bug form includes Revit 2022 through 2027 and a not-applicable option; the dropdown does not declare project support.
Issue contact URLs are static and cannot interpolate the receiving repository.
The shared config links both current projects' Discussions pages.
A future repository with different support links needs its own config and forms.

## Caller workflows

Reusable workflows run in the caller's repository context.
The caller owns triggers and grants the listed token permissions; a called workflow cannot elevate them.
`secrets: inherit` belongs on the calling job.
These workflows use only the automatic `github.token` and require no custom secrets.
See [GitHub's reusable workflow documentation](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows).

The examples reference `main` and receive shared updates immediately.
A caller can pin a commit SHA for controlled upgrades.
No caller is installed in either project by this repository.

### Community

Create `.github/workflows/community.yml` in the calling repository:

```yaml
name: Community
on:
  issues:
    types: [opened]
  pull_request_target:
    types: [opened, synchronize, reopened]
  schedule:
    - cron: '30 1 * * *'
permissions:
  contents: read
  issues: write
  pull-requests: write
jobs:
  community:
    uses: sharafutdinovdi/.github/.github/workflows/community.yml@main
    secrets: inherit
```

Keep the project's `.github/labeler.yml`.
The labeler reads the caller's configuration at `github.sha` (the PR base commit for `pull_request_target`) through the GitHub API when no local file exists; checkout is unnecessary.
The workflow does not execute PR code.
See [labeler configuration behavior](https://github.com/actions/labeler#inputs).
Build artifact comments remain project-specific.

All community inputs are optional:

| Input | Default |
| --- | --- |
| `configuration-path` | `.github/labeler.yml` in the calling repository |
| `issue-message` | Welcome text requesting complete reproduction details and screenshots |
| `pr-message` | Welcome text requesting the contribution guide and PR checklist |
| `days-before-stale` | `60` for issues and PRs |
| `days-before-close` | `14` for issues only |
| `stale-issue-message` | Empty; generates a message using the configured day counts |
| `stale-pr-message` | Inactivity notice stating that the PR remains open |
| `close-issue-message` | Closure notice with instructions to reopen |

The `pinned` and `bug` labels exempt issues and PRs from stale handling.
PRs are never closed automatically.
Custom day counts that disable stale handling or closure should include matching custom messages.

### Dependabot auto-merge

Create `.github/workflows/dependabot-auto-merge.yml` in the calling repository:

```yaml
name: Dependabot auto-merge
on:
  pull_request_target:
    types: [opened, synchronize, reopened]
permissions:
  contents: write
  issues: write
  pull-requests: write
jobs:
  auto-merge:
    uses: sharafutdinovdi/.github/.github/workflows/dependabot-auto-merge.yml@main
    secrets: inherit
```

This workflow has no inputs.
It requires a Dependabot-authored PR and a Dependabot actor.
Patch and minor updates enable squash auto-merge against the event's head commit.
Major updates receive `needs-review` and one maintained review comment.
The caller must enable auto-merge and squash merging in repository settings and configure its required checks and reviews.
The target event supplies write permissions for Dependabot automation; the workflow never checks out or executes PR code.

### Check failure hint

Add the `failure-hint` job below to the project's existing `.github/workflows/pr-checks.yml`.
This minimal complete example shows a check and its matching command:

```yaml
name: PR checks
on: pull_request
permissions:
  contents: read
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: '3.12'
      - run: pip install -r docs/requirements.txt
      - name: Build documentation
        run: mkdocs build --strict
  failure-hint:
    needs: [quality]
    if: >-
      failure() && github.event_name == 'pull_request' &&
      github.event.pull_request.head.repo.full_name == github.repository &&
      github.actor != 'dependabot[bot]'
    permissions:
      actions: read
      pull-requests: write
    uses: sharafutdinovdi/.github/.github/workflows/check-failure-comment.yml@main
    with:
      step-commands: '{"Build documentation":"mkdocs build --strict"}'
    secrets: inherit
```

The required string input `step-commands` is a JSON object mapping exact step display names to local commands.
Commands run from the repository root; include `cd` and required environment settings for checks that use another directory or configuration.
The workflow displays commands and never executes them.
Set `needs` to all check jobs and include a command for every check step that can fail.
Step names should be unique when their reproduction commands differ.

The workflow reads failed steps from the current run attempt, then creates or updates one marked PR comment.
Unmapped failed steps link to the run log and state that no local command is configured.
An obsolete PR head or a run without failed steps produces no comment.
Fork and Dependabot PR checks cannot post with their read-only tokens and are skipped.
A privileged `workflow_run` caller for those cases is separate migration work; do not run untrusted checks under `pull_request_target`.

## Maintenance

The initial content is committed directly to `main`.
Later changes use PRs.
Branch protection requires code-owner review, has zero general approvals and no required status checks, blocks force pushes, and permits administrator bypass.
`.github/CODEOWNERS` assigns the repository owner to all files.
Dependabot checks GitHub Actions dependencies monthly.

Validate workflow changes locally:

```sh
actionlint .github/workflows/*.yml
```
