# @rntpkgs/dep-guardian-action

GitHub Action for [dep-guardian](https://github.com/rnataoliveira/dep-guardian) — automated security fixer for npm projects. Reads Dependabot, CodeQL, Secret Scanning, and `npm audit` alerts and applies real dependency fixes, then opens a pull request.

## Usage

```yaml
name: dep-guardian

on:
  schedule:
    - cron: '0 9 * * 1'   # every Monday at 09:00 UTC
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write
  issues: write
  security-events: read

jobs:
  dep-guardian:
    runs-on: ubuntu-latest
    env:
      FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '24'
          cache: 'npm'

      - run: npm ci

      - name: Configure git
        run: |
          git config user.name "dep-guardian[bot]"
          git config user.email "dep-guardian[bot]@users.noreply.github.com"

      - uses: rnataoliveira/dep-guardian/packages/action@main
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          major-bump-mode: issue
```

Run `dg init` from the CLI to generate this workflow automatically.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `github-token` | yes | `${{ github.token }}` | GitHub token with repo, pull-requests, issues, and security-events permissions |
| `repo` | no | current repo | Repository in `owner/repo` format |
| `path` | no | `${{ github.workspace }}` | Path to the repository checkout |
| `dry-run` | no | `false` | Plan fixes without making changes or creating PRs |
| `major-bump-mode` | no | `issue` | What to do with major version bumps: `issue`, `pr`, or `skip` |
| `sources` | no | all four | Comma-separated alert sources: `dependabot,codeql,npm-audit,secret-scanning` |
| `validate` | no | all four | Comma-separated validation steps to run after fixing: `lint,typecheck,build,test` |
| `protected` | no | `` | Comma-separated packages to never auto-fix |
| `base-branch` | no | repo default | Base branch for pull requests |

## Outputs

| Output | Description |
|---|---|
| `fixes-applied` | Number of vulnerabilities fixed |
| `pr-url` | URL of the created pull request (if any) |
| `issues-created` | Comma-separated URLs of issues created for major bumps |

## What it does

| Alert type | Action |
|---|---|
| Minor / patch vulnerability | Updates `package.json`, runs install, validates, opens a PR |
| Major version bump required | Opens a GitHub Issue with changelog and migration notes |
| Transitive dependency | Finds the direct dep that owns it, updates that to a version shipping a safe transitive |
| CodeQL finding | Surfaces it in the PR body — requires manual code review |
| Exposed secret | Fails the action immediately — requires manual remediation |

## Required token permissions

- `security_events: read` — fetch Dependabot, CodeQL, and Secret Scanning alerts
- `contents: write` — push the fix branch
- `pull-requests: write` — open the PR
- `issues: write` — create issues for major bumps

The default `GITHUB_TOKEN` covers all of these when the workflow `permissions` block is set as shown above.

## License

MIT
