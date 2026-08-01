# ghul-code-review

Reusable GitHub Actions workflow that runs [OpenCode](https://opencode.ai)
(driving a Qwen model) on a pull request and posts the result as a single formal
PR review — an approval when the diff is clean, or a request-changes review with
line-anchored inline findings otherwise.

## What it does

Calls `opencode run --format json`, captures the event stream for the artifact
and the scorecard, and kills the process if it stops producing output. The
review brief is supplied by the calling workflow.

The review runs against a wall-clock budget and a single reviewer. Its tool
policy is an explicit permission config: bash is an allowlist (every command it
does not name is denied), and subagent (`task`) and web access are denied
outright. A review that fans out to subagents multiplies its token cost and
exhausts the job timeout before it posts anything, which leaves the PR blocked
on a review nobody can read.

## Seeing what a review actually did

An approval is a few characters and says nothing about what was weighed to reach
it. So each run produces:

- **A scorecard in the job summary** — duration, turns, tool calls and their
  names, subagent attempts, refused commands, and whether a review was posted.
  Written per attempt as it finishes, so a run killed at the job timeout still
  leaves its numbers behind.
- **The full event stream as an artifact**, one file per attempt. The model API
  key (by its exact value) and token-shaped strings are redacted before upload,
  because artifacts are not covered by the secret masking that applies to the
  log.

The job log itself stays quiet during the run — the per-event live trace that
earlier versions rendered was only ever there to surface a since-fixed CLI hang,
and is gone. The scorecard and the artifact are the record of a run.

A healthy run is tens of tool calls over a few minutes. Hundreds of tool calls,
any subagent attempt, or a run approaching the job timeout is the review
treating the PR as a programme of work rather than something to read.

## Usage

In a consumer repo, add a job that calls into this workflow:

```yaml
jobs:
  code_review:
    if: ${{ github.event_name == 'pull_request' && github.actor != 'dependabot[bot]' }}
    permissions:
      contents: read
      pull-requests: write
    uses: degory/ghul-code-review/.github/workflows/review.yml@v3
    with:
      prompt: |
        Review pull request #${{ github.event.pull_request.number }} on ${{ github.repository }}.

        Read other files only when surrounding context is needed.
        Group findings by severity (Bug / Concern / Nit).
    secrets:
      opencode-api-key: ${{ secrets.OPENCODE_API_KEY }}
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

## Human override

Once a non-bot reviewer has approved a PR — at any point in its history, not
just its current state — every later run of this workflow skips the automated
review outright instead of re-running the reviewer on top of it. This holds even
across further pushes: nothing re-arms it for that PR.

The mechanism is skipping, not re-approving: the workflow simply doesn't post
a new review, so the bot's last review stays whatever it already was and can't
override the human reviewer's still-active approval in the aggregate review
decision. This relies on the calling repo's branch protection *not* dismissing
stale approvals on push — if it does, a human approval stops being active as
soon as the next commit lands, and this check will find it in `reviews.json`'s
history regardless but the PR will still need a fresh approval to merge.

## Inputs

| Input | Default | Meaning |
|---|---|---|
| `prompt` | (required) | The full review brief. |
| `model` | `alibaba-token-plan/qwen3.8-max-preview` | `--model` argument, in opencode's `provider/model` form. The provider prefix names the API the `opencode-api-key` secret authenticates against; point it at any openai-compatible provider opencode knows and supply that provider's key. |
| `idle-timeout-seconds` | `90` | Kill the reviewer after this many seconds of stdout silence. |
| `max-attempts` | `1` | Attempt cap. A retry re-reads the PR from a fresh context and competes for the same job budget, so the default is to fail the check and let the next push re-trigger the review. |
| `post-findings-after-minutes` | `4` | Wall-clock budget given to the review: post by this point, whatever depth was reached. |
| `transcript-retention-days` | `14` | Retention for the uploaded transcript. |
| `opencode-version` | `1.18.11` | `opencode-ai` npm package version installed on the runner. |
| `job-timeout-minutes` | `12` | Outer cap on the whole job. A backstop for a wedged run, not the working budget — a run that reaches it is killed and posts nothing. |
| `gh-app-id` | `""` | GitHub App id. With `gh-app-private-key`, the review posts under that App's installation identity instead of `github-actions[bot]`. |
| `ghul-reference` | `false` | Fetch `GHUL.md` from `degory/ghul` main into the workspace root, for repos whose diffs contain ghūl source. |
| `style-reference` | `false` | Fetch `STYLE.md` from `degory/ghul-style` main into the workspace root, for repos carrying human-facing prose or example code. Needs `gh-app-id`, and `ghul-style` in `extra-repositories`. |
| `extra-repositories` | `""` | Additional repository names (same owner) the App token should reach, for prompts that read a file from a sibling repo. The calling repository is always included. |

The tool policy is fixed in the workflow rather than caller-configurable: a bash
allowlist of `gh pr diff` / `gh pr view` / `gh pr review` / `gh api` / `date` /
`git log` / `wc`, plus the read, glob, grep and edit tools, with subagent
(`task`) and web access denied.

## Secrets

| Secret | Required | Meaning |
|---|---|---|
| `opencode-api-key` | yes | API key for the model provider named by the `model` input's provider prefix (default `alibaba-token-plan`). |
| `gh-app-private-key` | no | PEM private key for `gh-app-id`. Required only when that is set. |

## Migrating from v2

v3 replaces the Claude Code CLI with OpenCode driving a Qwen model. The review
posture, posting mechanics, pre-fetched context, human-override skip and
scorecard are unchanged; the driver and its plumbing are not. Consumer changes:

- Bump the `uses:` ref from `@v2` to `@v3`.
- Replace the `claude-oauth-token` secret with `opencode-api-key` — an API key
  for the model provider (default `alibaba-token-plan`), not a Claude OAuth
  token.
- The `model` input is now opencode's `provider/model` form and defaults to a
  Qwen model; leave it unset to take the default.
- The `allowed-tools`, `disallowed-tools` and `claude-version` inputs are gone
  (`claude-version` is replaced by `opencode-version`); the tool policy is now
  fixed in the workflow.
- The job's display name changed from `Claude` to `OpenCode`. If a repo's branch
  protection lists the review job by name as a required check, update it.
- The scorecard no longer reports cost (the default endpoint is priced at zero);
  it still reports duration, turns, tool calls, fan-out, refused and posted.
