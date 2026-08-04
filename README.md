# ghul-code-review

Reusable GitHub Actions workflow that runs the [Claude Code](https://claude.com/claude-code)
CLI on a pull request and posts the result as a single formal PR review — an
approval when the diff is clean, or a request-changes review with
line-anchored inline findings otherwise.

## What it does

Calls `claude -p --output-format stream-json`, captures the event stream for
the artifact and the scorecard, and kills the process if it stops producing
output. The review brief is supplied by the calling workflow.

The reviewer is always the Claude Code CLI; which model actually serves the
request is a matter of which **provider** it points at (see 'Providers'
below). Anthropic's own API, Alibaba's Qwen Token Plan, and OpenRouter all
speak Claude Code's native protocol, so the same CLI, tool policy and prompt
work unmodified against any of them — only the base URL, auth token and
default model differ.

The review runs against a wall-clock budget and a single reviewer. Its tool
policy is an explicit allowlist/denylist pair (`allowed-tools` /
`disallowed-tools`): bash is scoped to `gh pr diff` / `gh pr view` / `gh pr
review` / `gh api` / `date` / `git log` / `wc`, and subagent tools (`Agent`,
`Workflow`, `Task`) are denied outright. A review that fans out to subagents
multiplies its token cost and exhausts the job timeout before it posts
anything, which leaves the PR blocked on a review nobody can read.

## Providers

| `provider` | Endpoint | Auth secret | Default model |
|---|---|---|---|
| `anthropic` | Anthropic's own API | `claude-oauth-token` | `claude-opus-5` |
| `qwen` | Alibaba Qwen Token Plan (Anthropic-compatible) | `qwen-auth-token` | `qwen3.8-max-preview` |
| `openrouter` | OpenRouter (Anthropic-compatible) | `openrouter-auth-token` | `z-ai/glm-5.2` |

Only the secret the resolved provider actually needs is read; the other two
can be absent or empty. Switching providers on an already-onboarded repo is a
plain variable set, no commit required:

```sh
gh variable set CODE_REVIEW_PROVIDER --repo degory/<repo> --body qwen
```

`provider` resolves as: the `provider` input, else the calling repo's
`CODE_REVIEW_PROVIDER` variable, else `anthropic`. `model` resolves the same
way against `CODE_REVIEW_MODEL`, then the per-provider default above.

OpenRouter's `~anthropic/claude-*-latest` catalog entries route real Anthropic
models through OpenRouter's billing instead of Anthropic's, if that's ever
useful; any other OpenRouter catalog id (`z-ai/*`, `qwen/*`, `deepseek/*`, …)
works the same way, unofficially — OpenRouter only guarantees its
`anthropic/*` models over this protocol, but GLM, Qwen, DeepSeek, Kimi and
MiniMax models have worked in practice.

## Seeing what a review actually did

An approval is a few characters and says nothing about what was weighed to reach
it. So each run produces:

- **A scorecard in the job summary** — provider, model, duration, turns, tool
  calls and their names, subagent attempts, refused commands, cost, and
  whether a review was posted. Written per attempt as it finishes, so a run
  killed at the job timeout still leaves its numbers behind.
- **The full event stream as an artifact**, one file per attempt. The
  provider auth token in use (by its exact value) and token-shaped strings are
  redacted before upload, because artifacts are not covered by the secret
  masking that applies to the log.

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
    uses: degory/ghul-code-review/.github/workflows/review.yml@v4
    with:
      prompt: |
        Review pull request #${{ github.event.pull_request.number }} on ${{ github.repository }}.

        Read other files only when surrounding context is needed.
        Group findings by severity (Bug / Concern / Nit).
    secrets:
      claude-oauth-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      qwen-auth-token: ${{ secrets.QWEN_AUTH_TOKEN }}
```

Passing both secrets (when both are available) is what makes the
`CODE_REVIEW_PROVIDER` variable switch free — the workflow only reads the one
the resolved provider needs, so an unused secret sitting there does nothing
until a variable flip asks for it.

The workflow owns the posting mechanics — it prepends runtime notes to the
prompt directing the model to approve when clean or post a request-changes
review with inline findings otherwise, so the caller's `prompt` only needs to
supply the review brief (what to flag), not how to post it.

The `permissions:` block on the calling job is required: this workflow declares
`pull-requests: write` so it can post the review, and the caller must grant at
least that. Without it, GitHub fails the workflow at validation with
`The workflow is requesting 'pull-requests: write', but is only allowed 'pull-requests: none'`.

## What lives here vs. in the calling repo

The workflow prepends runtime notes to every prompt, covering everything that
does not vary by repo: what PR context is pre-fetched and where, how to post a
review, that the review runs in parallel with CI and should trust the diff,
what makes a finding worth raising, source-comment hygiene, PR-description
shape, and the versioning mechanism.

A calling repo's own brief should therefore carry only what is specific to it:
what the repo ships, who consumes it, the blast radius of a mistake, the risk
areas worth extra attention, and what counts as a breaking change there.

Restating any of the shared material in a repo brief is a defect rather than
redundancy. These notes are given *first* and the repo-specific brief second,
so the review reads the authoritative shared rules before anything the caller
supplies; a brief should carry only what is specific to its repo. A brief that
restates shared material is noise at best, and at worst contradicts the live
rules the review has already been given.

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
| `provider` | `""` | `anthropic` / `qwen` / `openrouter`. Empty resolves to the calling repo's `CODE_REVIEW_PROVIDER` variable, then `anthropic`. See 'Providers'. |
| `model` | `""` | `--model` argument. Empty resolves to the calling repo's `CODE_REVIEW_MODEL` variable, then the resolved provider's default. |
| `allowed-tools` | `Bash(gh pr diff:*),Bash(gh pr view:*),Bash(gh pr review:*),Bash(gh api:*),Bash(date:*),Bash(git log:*),Bash(wc:*),Read,Write,Glob,Grep` | `--allowedTools` argument. |
| `disallowed-tools` | `Agent Workflow Task` | `--disallowedTools` argument. Denies subagent fan-out outright — it multiplies token cost and burns the wall-clock budget before anything posts. |
| `runner` | `ubicloud-standard-2` | GitHub Actions runner label the review job runs on. A small runner suffices — the job waits on the model rather than building. Defaults to a small Ubicloud runner; GitHub's own runners don't always reach the Qwen Token Plan or OpenRouter endpoints. |
| `idle-timeout-seconds` | `180` | Kill the reviewer after this many seconds of stdout silence; must exceed the model's time-to-first-token on a cold start. |
| `max-attempts` | `1` | Attempt cap. A retry re-reads the PR from a fresh context and competes for the same job budget, so the default is to fail the check and let the next push re-trigger the review. |
| `post-findings-after-minutes` | `4` | Wall-clock budget given to the review: post by this point, whatever depth was reached. |
| `transcript-retention-days` | `14` | Retention for the uploaded transcript. |
| `claude-version` | `2.1.221` | Claude CLI version installed on the runner. |
| `job-timeout-minutes` | `12` | Outer cap on the whole job. A backstop for a wedged run, not the working budget — a run that reaches it is killed and posts nothing. |
| `gh-app-id` | `""` | GitHub App id. With `gh-app-private-key`, the review posts under that App's installation identity instead of `github-actions[bot]`. |
| `ghul-reference` | `false` | Fetch `GHUL.md` from `degory/ghul` main into the workspace root, for repos whose diffs contain ghūl source. |
| `style-reference` | `false` | Fetch `STYLE.md` from `degory/ghul-style` main into the workspace root, for repos carrying human-facing prose or example code. Needs `gh-app-id`, and `ghul-style` in `extra-repositories`. |
| `extra-repositories` | `""` | Additional repository names (same owner) the App token should reach, for prompts that read a file from a sibling repo. The calling repository is always included. |

## Secrets

| Secret | Required | Meaning |
|---|---|---|
| `claude-oauth-token` | no | OAuth token for Anthropic's own API. Required when the resolved provider is `anthropic`. |
| `qwen-auth-token` | no | API key for Alibaba's Qwen Token Plan endpoint. Required when the resolved provider is `qwen`. |
| `openrouter-auth-token` | no | API key for OpenRouter. Required when the resolved provider is `openrouter`. |
| `gh-app-private-key` | no | PEM private key for `gh-app-id`. Required only when that is set. |

None are individually `required: true` because which one is needed depends on
`provider`; the workflow fails fast with a clear error if the resolved
provider's secret is empty, rather than letting the CLI fail confusingly
against no credentials.

## Migrating from v3

v4 replaces the OpenCode/Qwen driver with the Claude Code CLI, generalized to
run against any of three providers (see 'Providers'). The review posture,
posting mechanics, pre-fetched context, human-override skip and scorecard
shape are unchanged. Consumer changes:

- Bump the `uses:` ref from `@v3` to `@v4`.
- Replace the `opencode-api-key` secret with `qwen-auth-token` (same
  underlying key — Alibaba's Qwen Token Plan token) and add
  `claude-oauth-token` alongside it if the repo has a Claude OAuth token to
  offer. Passing both is what makes provider switching free afterwards.
- The `model` input is no longer opencode's `provider/model` form — it's a
  bare model id, resolved against whichever provider is active. Leave it
  unset to take that provider's default.
- `opencode-version` is gone, replaced by `claude-version`.
- `allowed-tools` and `disallowed-tools` are back as inputs (the OpenCode
  driver's `OPENCODE_PERMISSION` config is gone; the Claude CLI use
  `--allowedTools` / `--disallowedTools` instead).
- The job's display name changed from `OpenCode` to `Code review` (a
  provider-neutral name, so it doesn't need to change again on the next
  provider swap). It isn't listed in any consumer repo's required status
  checks today, but double-check before relying on that.
- The scorecard reports cost again (it's meaningful once more now that a run
  can be against Anthropic's own paid API, not just a zero-priced endpoint).
