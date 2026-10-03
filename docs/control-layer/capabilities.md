# Stage 0: agent and tooling capabilities

Verified on 2026-09-24 against the installed CLIs, by reading `--help` and by running
`scripts/probe-agents.sh`. Rerun that script after upgrading either agent CLI. If a
check fails, the adapter that depends on it needs attention.

| Tool | Version |
|---|---|
| Claude Code | 2.1.281 |
| Codex CLI | 0.155.1 (0.156.1 passes the probe since Stage 5) |
| gh | 2.101.0 |
| git | 2.50.1 |
| Go | 1.27.1 |

## Claude Code

Headless mode is `claude -p <prompt>`. Every feature below was checked with a real run.

- **Working directory.** Claude uses the process cwd. There is no `--cd` flag, so the
  runner starts Claude with `cmd.Dir = worktree`. `CLAUDE.md`, skills, and project
  settings load from that directory.
- **Structured output.** `--output-format json --json-schema '<schema>'` returns one JSON
  object on stdout. The validated result is in `structured_output`, and `result` holds
  the same data as a string. Other useful fields are `session_id`, `is_error`,
  `subtype`, `permission_denials`, `num_turns`, and `total_cost_usd`.
- **Event stream.** `--output-format stream-json --verbose` prints one JSON event per
  line (`system`, `assistant`, `user`, `result`, and others). The `result` event carries
  the same fields as the json output, `structured_output` included, but it isn't always
  the last line. Resuming a missing session prints a `result` with subtype
  `error_during_execution` and `num_turns` 0, and "No conversation found" on stderr.
  The json output printed nothing on stdout in that case.
- **Transcripts.** Each persisted session's full transcript is at
  `~/.claude/projects/<cwd with every non-alphanumeric character as "-">/<session>.jsonl`,
  or under `$CLAUDE_CONFIG_DIR` when set. It records every tool call, its output, and
  per-turn token usage.
- **Session IDs.** `--session-id <uuid>` is honored, so foreman can assign the ID before
  launch and record it before the agent starts working.
- **Resume.** `--resume <uuid> -p <message>` continues the same session and keeps the
  same ID. It works with `--json-schema`.
- **Permissions.** Any tool call that isn't pre-approved is denied, not prompted, and
  shows up in `permission_denials`. Both `--permission-mode auto` and `manual` refused a
  plain `touch`. `--permission-mode acceptEdits --allowedTools Bash` worked.
  `bypassPermissions` exists, but foreman does not default to it.
- **Stdin.** When stdin is not a TTY, Claude waits 3s for piped input. The runner must
  connect stdin to `/dev/null`.
- **Hooks.** The user's global hooks still run in headless sessions. A broken SessionEnd
  hook only wrote to stderr, and stdout stayed valid JSON. Keep stdout and stderr in
  separate streams.
- **Nullable objects in the schema.** The outcome schema's `documents` is
  `"type": ["object", "null"]` with strict-mode properties. Both CLIs accept it and
  return the object for a "planned" outcome (probe-agents.sh).
- **Other options.** `--no-session-persistence` gives disposable reviewer sessions.
  `--max-budget-usd` caps spend.

## Codex CLI

Headless mode is `codex exec <prompt>`.

- **Working directory.** `-C <dir>` sets it, and cwd also works. `AGENTS.md` loads from
  there.
- **Structured output.** `--output-schema <file>` constrains the final message, and
  `-o <file>` writes that message to a file. The schema must follow OpenAI strict mode:
  every property listed in `required` and `additionalProperties: false`. A unit test
  enforces this.
- **Session IDs.** Codex assigns the ID at runtime. `--json` prints JSONL events, and the
  first one is `{"type":"thread.started","thread_id":...}`. The adapter reads the ID
  from there. It cannot be assigned in advance.
- **Resume.** `codex exec resume <thread_id> <message>` continues the thread and keeps
  the ID. It accepts `--output-schema`, `--json`, and `-o`. It has no `-s` flag, so the
  sandbox is set with `-c sandbox_mode=...` or comes from config.
- **Stage settings.** `codex exec` and `codex exec resume` accept `-m <model>` and
  `-c model_reasoning_effort="<effort>"`. The installed CLI help confirms both flags;
  stage-specific model calls have not been probed against the service.
- **Sandbox.** `-s workspace-write` blocks network access by default. With it, curl to
  api.github.com returned `000`. Adding `-c sandbox_workspace_write.network_access=true`
  made it return `200`.
- **GitHub credentials in the sandbox.** With network access on, `gh` and git over HTTPS
  work inside the Codex sandbox using the user's keyring login.
- **Worktrees.** A linked worktree keeps its index, refs, and objects in the main clone's
  `.git`, outside the worktree. `-s workspace-write` doesn't let Codex write there, so
  `git commit` fails. Adding that directory with
  `-c 'sandbox_workspace_write.writable_roots=["<clone>/.git"]'` fixes it, for both
  `exec` and `exec resume`. `exec resume` has no `--add-dir` flag. The earlier note said
  no flag was needed, but the probe's clone was under `$TMPDIR`, which workspace-write
  can write by default. The probe now excludes `/tmp` and `$TMPDIR` for these checks.
- **Review.** `codex exec review --base <branch>` is a built-in review command and accepts
  `--output-schema`. Foreman doesn't use it. It uses plain `codex exec` with its own
  review prompt, so both reviewers get the same inputs and return the same schema.
- **Other options.** `--ephemeral` gives disposable reviewer sessions. `codex exec` also
  has a `--worktree` flag, which foreman doesn't use because it manages worktrees itself.

## Progress text

Checked on 2026-10-03 with Claude Code 2.1.288 and Codex CLI 0.160.0.

- **It is billed output.** Claude's `assistant` text blocks and Codex's `agent_message`
  items are model output, counted in Claude's `usage.output_tokens` and Codex's
  `turn.completed` `output_tokens`.
- **No flag stops it.** Neither `claude --help` nor `codex exec --help` has an option
  for it. Claude's `--brief` enables a messaging tool and is unrelated. The prompt is
  the only lever.
- **It is a small share of a real turn.** Past foreman logs held about 1,900 and 3,900
  characters of Claude text (roughly 500 and 1,000 tokens) against 52,716 and 86,500
  output tokens. Most output is tool input and thinking.
- **The silent-work section cuts it.** `scripts/probe-agents.sh` runs a three-file read
  task five times with and without the section, and counts a run only if each file
  came back from its own successful tool call and the answer was 6. All 20 runs did.
  Codex wrote 52 characters of progress text without it and none with it, and its
  output tokens fell from 1,333 to 1,040. Claude (haiku) wrote none either way on that
  task, so its output tokens (3,410 and 3,721) only show run-to-run noise. Codex's
  built-in instructions ask for a preamble before tool calls; the section has to say it
  overrides them, or Codex still writes one.

## gh

The user is authenticated through the keyring.

- **Inbox search.** `gh search issues --assignee=@me --state=open --json number,title,repository,updatedAt,url`
  works. Other queries use `--mentions=@me`, and `gh search prs --review-requested=@me`.
  The results include `repository.nameWithOwner`.
- **Pull requests.** `gh pr view <n> --json` exposes `state`, `reviewDecision`,
  `latestReviews`, `statusCheckRollup`, `headRefName`, `mergedAt`, and
  `closingIssuesReferences`.
- **Mergeability.** `gh pr view <n> --json mergeable` gives MERGEABLE, CONFLICTING, or
  UNKNOWN. GitHub computes it lazily, so UNKNOWN is common right after a push to either
  branch. `gh pr view --json` also lists `baseRefName`.
  `gh api repos/<owner>/<repo>/git/ref/heads/<branch> --jq .object.sha` prints a
  branch's head commit. The field list was read from gh 2.101.0; a conflicting
  PR was not probed live.
- **CI.** `gh pr checks <n> --json name,state,bucket` gives each check a `bucket` of pass,
  fail, pending, skipping, or cancel. `--required` limits it to required checks.
  According to `gh pr checks --help`, gh exits 8 while checks are pending. When a PR has
  no checks, gh errors with "no checks reported on the '<branch>' branch". Foreman
  ignores the exit code whenever JSON came back. This was not probed live on a
  real PR.
- **Creating PRs.** `gh pr create --title --body --head` can run from inside the worktree.
- **Issue comments.** `gh api --method POST repos/<owner>/<repo>/issues/<n>/comments -f body=<text>`
  creates a comment, and `--method PATCH repos/<owner>/<repo>/issues/comments/<id>` edits
  one. Both return the REST comment (`id`, `html_url`, `body`). `-f` sends the value as a
  raw string; `-F` would read a leading `@` as a file. GitHub caps a comment at 65,536
  characters. This is from the REST API reference and unit tests against its response
  shape; it wasn't probed live, because a probe would post on a real issue.
- **Pull request comments.** A pull request's conversation is spread over three REST
  endpoints: `repos/<owner>/<repo>/issues/<n>/comments` for the conversation comments,
  `repos/<owner>/<repo>/pulls/<n>/comments` for the inline comments and their replies
  (`path`, `line`, `original_line`, `in_reply_to_id`), and
  `repos/<owner>/<repo>/pulls/<n>/reviews` for each submitted review, whose `body` is
  empty for a bare approval. `gh api <path> --paginate --slurp` returns one JSON array
  per page, so the parser unwraps `[][]comment`. A reply inside an inline thread is
  `--method POST repos/<owner>/<repo>/pulls/<n>/comments/<id>/replies`. Verified against
  the REST reference and unit tests over recorded responses, not probed live, because a
  probe would comment on a real pull request.

## git worktree

`git worktree add -b <branch> <path> <base>` creates an isolated checkout.
`git worktree list --porcelain` makes the operation idempotent: if the branch is already
checked out at the expected path, foreman reuses it. Git refuses to check out a branch in
two worktrees at once, which prevents two agents from writing to the same checkout.
`git merge-base --is-ancestor <a> <b>` exits 0 when `a` is in `b`'s history, 1 when it
is not, and another code on an error such as an unknown commit. Foreman uses it to check
a conflict repair after `git fetch origin`.

`git merge-tree --write-tree --name-only --no-messages <branch> <base>` (git 2.38 or
newer) merges in memory and touches neither the index nor the working tree. It exits 0
for a clean merge and prints the tree id. It exits 1 for a conflict and prints the tree
id, then one conflicting path per line. Any other exit is an error, such as a missing
ref. Verified by `internal/worktree/git_test.go` against git 2.50.

## Assumptions in the design doc that the tooling doesn't support as written

1. **"The agent asks me mid-run."** A headless turn can't stop and wait for input. What
   works instead is that the agent ends its turn with `status: blocked` and its
   `questions`. Foreman stores them, and `respond` resumes the same session with the
   answers. From the user's side this behaves the same, and both CLIs support it.
2. **`AgentAdapter.getStatus`.** Neither CLI can report on a turn that's still running.
   Foreman gets status from its own records instead: the task file, whether the runner
   process is alive, and the runner's log. `getStatus` was removed from the interface.
3. **"The agent runs `git push` and `gh pr create`."** Both agents can do this. Codex
   needs `-c sandbox_workspace_write.network_access=true`. With that set, `gh api user`
   and an authenticated `git ls-remote` both work inside the Codex sandbox, because gh
   and git reach the keyring normally. The probe checks this. The agent reports the PR
   in its outcome (`pr_opened`, `prNumber`, `prUrl`), and foreman confirms with `gh`
   that the PR exists.
4. **Unattended permissions.** Neither `auto` nor the default permission mode lets
   Claude run shell commands headless. The adapter has to pass an explicit
   `--allowedTools` list. The default list must cover the repo's test, build, and lint
   commands, and it can be configured.
5. **The recommendation field.** In the probe, Claude filled `recommendation` with a full
   sentence, not an option label. Since Stage 2, the implementation prompt says the
   field must be the label only. Stage 3 should still accept text that starts with a
   label.
