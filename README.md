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

## What lives here vs. in the calling repo

The workflow appends runtime notes to every prompt, covering everything that
does not vary by repo: what PR context is pre-fetched and where, how to post a
review, that the review runs in parallel with CI and should trust the diff,
what makes a finding worth raising, source-comment hygiene, PR-description
shape, and the versioning mechanism.

A calling repo's own brief should therefore carry only what is specific to it:
what the repo ships, who consumes it, the blast radius of a mistake, the risk
areas worth extra attention, and what counts as a breaking change there.

Restating any of the shared material in a repo brief is a defect rather than
redundancy. The brief is read *before* these notes, so a stale copy silently
overrides the current one.

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
| `gh-app-id` | `""` | GitHub App id. With `gh-app-private-key`, the review posts under that App's installation identity instead of `github-actions[bot]`. |
| `ghul-reference` | `false` | Fetch `GHUL.md` from `degory/ghul` main into the workspace root, for repos whose diffs contain ghūl source. |
| `style-reference` | `false` | Fetch `STYLE.md` from `degory/ghul-style` main into the workspace root, for repos carrying human-facing prose or example code. Needs `gh-app-id`, and `ghul-style` in `extra-repositories`. |
| `extra-repositories` | `""` | Additional repository names (same owner) the App token should reach, for prompts that read a file from a sibling repo. The calling repository is always included. |

## Secrets

| Secret | Required | Meaning |
|---|---|---|
| `claude-oauth-token` | yes | `CLAUDE_CODE_OAUTH_TOKEN` for API auth. |
| `gh-app-private-key` | no | PEM private key for `gh-app-id`. Required only when that is set. |

## Live progress

Each stream-json event becomes one summary line in the job log — `[init]`,
`[text]`, `[tool]`, `[result]`, `[thinking]` (throttled to per-1000-token
milestones), `[done]`. The GitHub Actions UI shows these as they happen.
