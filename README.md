# ghul-code-review

Reusable GitHub Actions workflow that runs Claude Code on a pull request and
posts the result as a single formal PR review — an approval when the diff is
clean, or a request-changes review with line-anchored inline findings otherwise.

## What it does

Calls the `claude` CLI directly with `--output-format stream-json`, parses the
event stream into one-line summaries that appear live in the GitHub Actions
log, and retries inline if the process stops producing output. The review
brief is supplied by the calling workflow.

## Usage

In a consumer repo, add a job that calls into this workflow:

```yaml
jobs:
  code_review:
    if: ${{ github.event_name == 'pull_request' && github.actor != 'dependabot[bot]' }}
    permissions:
      contents: read
      pull-requests: write
    uses: degory/ghul-code-review/.github/workflows/review.yml@v2
    with:
      prompt: |
        Review pull request #${{ github.event.pull_request.number }} on ${{ github.repository }}.

        Read other files only when surrounding context is needed.
        Group findings by severity (Bug / Concern / Nit).
    secrets:
      claude-oauth-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

The workflow owns the posting mechanics — it appends runtime notes to the
prompt directing the model to approve when clean or post a request-changes
review with inline findings otherwise, so the caller's `prompt` only needs to
supply the review brief (what to flag), not how to post it.

The `permissions:` block on the calling job is required: this workflow declares
`pull-requests: write` so it can post the review, and the caller must grant at
least that. Without it, GitHub fails the workflow at validation with
`The workflow is requesting 'pull-requests: write', but is only allowed 'pull-requests: none'`.

## Inputs

| Input | Default | Meaning |
|---|---|---|
| `prompt` | (required) | The full review brief sent via `-p`. |
| `model` | `claude-opus-4-8` | `--model` argument. |
| `allowed-tools` | `Bash(gh pr diff:*),Bash(gh pr view:*),Bash(gh pr review:*),Bash(gh api:*),Read,Write,Glob,Grep` | Tool allowlist. |
| `idle-timeout-seconds` | `90` | Kill `claude` after this many seconds of stdout silence and retry. |
| `max-attempts` | `3` | Inline retry cap. |
| `claude-version` | `2.1.159` | Claude CLI version installed on the runner. |
| `job-timeout-minutes` | `12` | Outer wall-clock cap on the whole review job. |

## Secrets

| Secret | Required | Meaning |
|---|---|---|
| `claude-oauth-token` | yes | `CLAUDE_CODE_OAUTH_TOKEN` for API auth. |

## Live progress

Each stream-json event becomes one summary line in the job log — `[init]`,
`[text]`, `[tool]`, `[result]`, `[thinking]` (throttled to per-1000-token
milestones), `[done]`. The GitHub Actions UI shows these as they happen.
