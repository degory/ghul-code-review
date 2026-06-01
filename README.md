# ghul-code-review

Reusable GitHub Actions workflow that runs `claude -p` directly on a PR diff,
posts findings as a single rollup PR comment, and bails out fast on the hangs
that plague `anthropics/claude-code-action@v1`.

## Why

`anthropics/claude-code-action@v1` regularly hangs after `Claude Code
initialized` with no further output, eating the full step timeout. This
workflow invokes the `claude` CLI directly with `--output-format stream-json`,
runs a per-step heartbeat watchdog that kills the process on 90 s of stdout
silence, and retries up to 3 × inline within the same job.

## Usage

In a consumer repo, add a job that calls into this workflow:

```yaml
jobs:
  code_review:
    if: ${{ github.event_name == 'pull_request' && github.actor != 'dependabot[bot]' }}
    uses: degory/ghul-code-review/.github/workflows/review.yml@v1
    with:
      prompt: |
        Review pull request #${{ github.event.pull_request.number }} on ${{ github.repository }}.

        Use `gh pr diff ${{ github.event.pull_request.number }}` for the diff.
        Read other files only when surrounding context is needed.
        Post a single rollup comment via `gh pr comment ${{ github.event.pull_request.number }} --body "..."` organised by severity (Bug / Concern / Nit).
        If the diff is clean, post a one-line "LGTM".
    secrets:
      claude-oauth-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

## Inputs

| Input | Default | Meaning |
|---|---|---|
| `prompt` | (required) | The full review brief sent via `-p`. |
| `model` | `claude-opus-4-8` | `--model` argument. |
| `allowed-tools` | `Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*),Read,Glob,Grep` | Tool allowlist. |
| `idle-timeout-seconds` | `90` | Kill claude after this many seconds of stdout silence. |
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
milestones), `[done]`. Tail the running job in the GitHub Actions UI to see
what claude is doing in real time.

## Private cross-repo calls

If this repo is private and you're calling it from another private repo
under the same owner, the access policy on this repo must allow it:

> Settings → Actions → General → Access → "Accessible from repositories
> owned by the user 'degory'"

The default for new private repos is "Not accessible", which blocks
cross-repo `uses:`.
