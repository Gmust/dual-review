# Dual Review for Claude Code

[Русская версия](README.ru.md)

`/dual-review` is a code review command for [Claude Code](https://code.claude.com).
Two independent reviewers look at your change in parallel, each in a fresh
session. Their findings become one numbered fix list, and fresh reviewers
re-check that list round after round until both agree on every fix. With
`--fix`, the agreed fixes are applied and two more fresh reviewers verify
them.

A finding only makes the list when two separate sessions agree it is real,
so you get fewer false alarms than from a single-pass review.

```
            ┌─ Reviewer A (correctness & safety) ─┐
round 1     ├─ Reviewer B (design & robustness) ──┼─► merge into one list F1..Fn
            └─ Reviewer S (security, risky diffs)─┘
                                │ anything not agreed yet?
            ┌─ fresh A: AGREE / AMEND / DISAGREE / CHOOSE ─┐
round 2..N  │                                              ├─► resolve votes
            └─ fresh B: AGREE / AMEND / DISAGREE / CHOOSE ─┘
                                │ all agreed, rejected or handed to you
                              report
                                │ --fix (or "yes")
            apply agreed fixes ─► fresh A + B verify ─► until both approve
```

## Install

Works the same on macOS, Linux and Windows. You need Claude Code and `git`.

In a Claude Code session, run:

```
/plugin marketplace add Gmust/dual-review
/plugin install dual-review@dual-review
```

When asked for a scope, pick **user** so it works in all your projects.
Then run `/reload-plugins` (or restart Claude Code).

From a terminal instead:

```bash
claude plugin marketplace add Gmust/dual-review
claude plugin install dual-review@dual-review
```

In the Claude desktop app or the VS Code extension, add the marketplace
`Gmust/dual-review` from the plugin browser and install **Dual Review**.

### Update

```
/plugin marketplace update dual-review
```

Or turn on auto-update once: `/plugin` → **Marketplaces** → `dual-review` →
**Enable auto-update**.

### Uninstall

```
/plugin uninstall dual-review@dual-review
```

## Use

Open Claude Code in a git repository, on the branch you want reviewed:

```
/dual-review                         # your branch vs the default branch, plus uncommitted changes
/dual-review main                    # against a specific base
/dual-review --staged                # staged changes only
/dual-review #482                    # a GitHub PR (gh pr checkout 482 first for full context)
/dual-review --fix                   # review, apply the agreed fixes, verify until both approve
/dual-review --depth deep            # override the risk-based depth (quick | standard | deep)
/dual-review --security              # add the security reviewer (on by default for deep)
/dual-review --rounds 5              # override the round cap
/dual-review --models opus,sonnet    # a different model for each reviewer
/dual-review main check the token refresh logic   # any other text is a focus note for the reviewers
```

If another command already uses the name `/dual-review`, use the full name
`/dual-review:dual-review`.

The report opens with a count, for example
`To fix: 2 (1 blocker, 1 major) · deadlocked 1 · pre-existing 3`, or says
`Nothing to fix.` Sections:

- **Agreed fixes**: both reviewers signed off on the problem and the fix.
- **Deadlocked**: they couldn't agree. You get both arguments and decide.
- **Suggestions**: minor items without consensus.
- **Pre-existing**: real bugs this change didn't introduce. Reported, never
  fixed unless you ask.
- **Rejected**, **Nits**, and **Prevent recurrence** (a lint rule, test or
  CLAUDE.md line that would have caught the bug).

Without `--fix` it asks before changing anything. Fixes are never committed;
run `git diff` to see them.

## How it works

**What the reviewers read**
- **Intent**: the PR title and body, the linked issue (from `Fixes #N` or a
  branch name like `fix/123-…`) and the commit messages. They check that the
  change does what it claims, and that a bug fix removes the root cause
  instead of hiding the symptom.
- **Your rules**: `CLAUDE.md` always, and `REVIEW.md` at the repo root if you
  have one (severity rules, paths to skip, checks to always run).
- **Depth**, chosen from what the change touches:

  | Depth | When | Rounds |
  |---|---|---|
  | quick | only docs, comments, copy or tests, under 200 lines (CLAUDE.md, AGENTS.md and REVIEW.md count as standard) | 2 |
  | standard | everything else | 3 |
  | deep | auth, payments, keys or crypto, smart contracts, migrations, CI or release scripts, concurrency, security config, or 1,000+ lines | 5 |

  Quick reports blockers and majors only. Deep asks reviewers to back each
  blocker or major with a command they ran, and turns on the security
  reviewer.
- **Security reviewer S** looks only for vulnerabilities, from an attacker's
  point of view: authorization, injection, secrets, crypto, SSRF, trust
  boundaries, dependencies and CI workflows, and on-chain access control and
  reentrancy. S only finds; A and B verify its findings by vote.

**How agreement works**
- In round 1, a finding that two reviewers found independently with
  compatible fixes is accepted straight away. Each finding lists every place
  the same bug pattern appears, and the fix covers all of them.
- Every later round starts two new reviewers. They re-read the code, see each
  open item with both sides' arguments, and vote: `AGREE`, `AMEND` (a better
  fix), `DISAGREE` (not real, wrong fix, wrong severity or pre-existing, with
  `file:line` evidence) or `CHOOSE` between two fixes.
- After round 1, new findings must be blockers or majors introduced by the
  change. A minor item gets one vote, then becomes a suggestion. Nits are
  never voted on.
- An item whose votes repeat unchanged, or that is still open at the round
  cap, is handed to you as deadlocked.
- Verification checks only the lines the fixes changed. An item that fails
  verification twice stops the loop with a suggestion to rethink that part.
- Your main session only merges and tracks votes. It never adds findings or
  breaks ties.

## Cost

Every round is two or three full reviews. A standard run is at most 3 review
rounds plus 2 verify rounds; most need fewer. A small change typically takes
2 to 4 minutes. Lower `--rounds` or use `--depth quick` to spend less.

## Customize

- **Per repo**: add a `REVIEW.md` at the repo root, for example:

  ```markdown
  # Review rules
  - Blocker: anything that can lose or leak user data.
  - Skip generated code under src/gen/ and all lockfiles.
  - Always check that new API routes have an integration test.
  - Report at most five nits.
  ```

- **Prompts**: fork this repo and edit `agents/dual-reviewer.md` (what the
  reviewers look for, severity, output format) or
  `skills/dual-review/SKILL.md` (scope, depth rules, round logic), then
  install from your fork.

## Install without the plugin system

If you can't use plugins, copy the files with the installer. Download the
repo (**Code** → **Download ZIP**) and unzip it, then run:

macOS / Linux:

```bash
cd dual-review-main
bash install.sh                  # into ~/.claude, all projects
bash install.sh --project        # into ./.claude of the repo you run it from
bash install.sh --uninstall
```

Windows (PowerShell):

```powershell
cd .\dual-review-main
powershell -ExecutionPolicy Bypass -File .\install.ps1               # into %USERPROFILE%\.claude
powershell -ExecutionPolicy Bypass -File .\install.ps1 -Project
powershell -ExecutionPolicy Bypass -File .\install.ps1 -Uninstall
```

`-ExecutionPolicy Bypass` applies to that one run only. Restart Claude Code
after the first install. Don't use both install methods at once: the copied
`/dual-review` would take the short name from the plugin's.

## Running in CI

In headless runs (`claude -p`, GitHub Actions), set
`CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`. Without it, reviewers in later
rounds can lose tool access once the main turn ends.

## Troubleshooting

- **`/dual-review` isn't in the menu**: run `/reload-plugins`, or restart
  Claude Code.
- **A different `/dual-review` runs**: you also have a copied install. Remove
  it with `bash install.sh --uninstall` (or `install.ps1 -Uninstall`), or call
  `/dual-review:dual-review`.
- **The marketplace add fails**: check that `git` is installed and on your
  `PATH`.

## License

[MIT](LICENSE)
