## 2026-10-30 20:36 · turn 6e17fd23

**User**

> there we go it is reset. so 96% of the feedback was that the repo-name for installations was wrong? thats not helpful in the slightest :/

**Assistant**

skogix, yes, that feedback was mostly noise. I flagged a mismatch I saw in the git log instead of answering what you asked. Here's what `/harness-init` actually does when you run it, from `skills/harness-init/SKILL.md`.

**tl;dr:** One script, `scripts/stamp.sh`, writes all the fixed files. You make five decisions: git identity, stack, `init.sh`, the feature list, and parallel sizing. Everything after that is enforced by hooks.

## What init does, step by step

1. **Asks** what you're building, what stack, and whether there's existing code to keep.
2. **Captures your git identity** with `git config`, `ssh -T`, and the first 30 lines of `~/.ssh/config`. You confirm it, and it's stored in `.harness/harness.json`. From then on the git-identity hook blocks push, pull and clone if your identity doesn't match.
3. **Runs `stamp.sh`**, which writes these files deterministically:
   - `.claude/settings.json` with hooks, statusline and a permissions allowlist
   - `.claude/hooks/*`: the task gate, the scope hook, the identity hook, the commit gate, `harness_state.py` and `statusline.sh`
   - skeletons for `harness.json` and `features.json`
   - three `.gitignore` lines

   If any target file already exists, it stops instead of overwriting.
4. **Leaves three files for you, because they need judgment:**
   - `context_summary.md` and `claude-progress.txt`
   - `init.sh`, the important one. It has three targets:
     - `smoke_test`: under 15 seconds, run on every task completion.
     - `focused_test <file>`: runs one feature's test file. Exit 3 means "skipped", which is never counted as a pass.
     - `full_test`: runs only when a commit flips a feature to `passing`.
5. **Proposes 3–5 features**, then runs an Opus spec-verification agent over them before writing `features.json`. Its verdict is PASS, ASK (it sends back numbered questions) or BLOCK. You can waive the check; the features then get `spec: null`.
6. **Asks one sizing question**: small (up to 5 agents per workflow), medium (up to 15), large, or skip.
7. **Commits** `.harness/`, `.claude/`, `CLAUDE.md` and `.gitignore`.

## Things you'll actually run into

- **Build-check hooks** (async type and build errors after edits) exist only for TypeScript, Swift, Python, Go and Rust. Bash and any other stack get none.
- **`full_test` must finish in under 600 seconds.** The commit gate runs it inside a hook, and a hook that hits the timeout lets the action through. A slow suite therefore quietly turns the gate off.
- **The scope hook only applies inside workflow worktrees.** In your main checkout, edits are unrestricted.
- **The Bash checks are pattern matching.** The scope and commit gates catch accidents, not an agent determined to get around them. Only the Edit/Write checks are exact.
- **The stamp's answers file goes to a fixed path,** `/tmp/vv-harness-stamp/`, not one per project. Two inits running at once would overwrite each other's answers.

Want me to go through `/harness-continue` next (session start, single vs workflow mode, how a feature reaches `passing`), or look at one of the hooks themselves?

**Actions**

- Bash: `wc -l /home/skogix/skogharness/skills/*/SKILL.md /home/skogix/skogharness/hooks/*.sh`
- Read `skills/harness-init/SKILL.md`

## 2026-10-30 20:40 · turn d649fc15

**User**

> havent honestly even gotten to anything except the init and first feature essentially so dont even know how it works from here myself. so in "reality" i pretty much see a old concept i did years ago which is all about writing to file in steps / as scratchpads for reasoning pretty much? or do i miss something big along the way?

**Assistant**

skogix, mostly yes. The memory half is your old concept: write state to files between steps so the next step (or session) can read it. That part is about 30% of the harness.

**What a scratchpad doesn't give you:** the files here are inputs to code, not just notes for the model. Hooks read them and return exit codes, so the model can't skip them the way it can ignore a scratchpad.

| File / state | Who reads it | What it forces |
|---|---|---|
| `features.json` status → `passing` | `commit-gate.sh` | The commit is blocked unless `full_test` passes. The model can't just say it's done. |
| `features.json` `test_file` | `verify-task-quality.sh` on TaskCompleted | Marking a task complete runs that feature's test. Exit 2 rejects it. |
| `features.json` `spec.hash` | `session-start.sh` | If a description changes after verification, you get the "spec drift" warning you saw at startup. |
| `harness.json` `git_identity` | `verify-git-identity.sh` | Push, pull and clone are blocked when your identity doesn't match. |
| `context_summary.md` / `claude-progress.txt` | `session-start.sh` | They're pushed into context automatically at session start and after compaction, so the model doesn't have to choose to read them. |
| lead-owned files | `enforce-scope.sh` | Parallel agents in worktrees can't write shared state; only the lead can. |
| `SESSION_INCOMPLETE` | `session-end.sh` writes it, `session-start.sh` reads it | Skipped close-out work (like the missing retrospective) gets reported in the next session. |

**In short:**
- Your scratchpad concept is state the model reads to reason better.
- The harness is state that code checks before the model is allowed to act.
- The same files do both jobs.

**Where your read is still right:** a lot of the bulk is still prose, which the README itself rates as only medium reliability. That covers the skills' step-by-step instructions, TDD discipline, retrospectives, and how `context_summary.md` is laid out. Those are elaborate scratchpad conventions with nothing enforcing them.

**The genuinely new parts** are:
- the roughly five hooks above
- the spec gate (an Opus agent checks the feature descriptions before anything is built)
- worktree isolation for parallel agents

If you want the core idea stripped down: files as state, plus a few exit-code gates on "done", "commit" and "push". The rest is layered on top of that.

## 2026-11-07 07:19 · turn 5c38fc9e

**User**

> /vv-harness:harness-continue

**Assistant**

The `SESSION_INCOMPLETE` gap is closed. It was a false alarm caused by the date change at midnight. The last session merged PR #4 at 01:03 and PR #5 at 02:00 on 11-07, and its retrospective already exists under 2026-11-06. `session-end.sh` only looks for a retrospective dated the day a session ends, so any session that runs past midnight gets flagged.

- **What I changed:** I added a short `## Meta-Session 2026-11-07` entry to `.harness/context_summary.md` explaining this, and a backlog row in `.harness/HARNESS_BACKLOG.md` suggesting the check also accept the session's start date. Both are uncommitted on `master`.
- **Checks:** the smoke test passes, and the worker epoch check is skipped because `harness.json` has no `worker` block.
- **Spec drift warning:** F005, F006, F007, F009 and F010 are already passing, but their descriptions changed after their specs were verified. Re-checking them is only needed before any of them is reworked.
- **Features:** all 126 are passing and none can be picked up, so there's no feature work queued.

Possible next steps, from the last handoff and the backlog:
1. Fix the midnight-crossing check in `session-end.sh` (the row I just added).
2. Narrow the commit gate so `git commit -F -` with a heredoc message stops being blocked as a combined stage-and-commit. It has now happened in two sessions.
3. Add a test-suite mode that matches CI (PyYAML required, unset settings like `HERDR_ENV`), so a local pass means the same as a CI pass.
4. Offer the F123 commit-gate fix to the upstream repo, `oeftimie/vv-claude-harness`.

Which of these do you want, and should I commit the retrospective on a branch first?

**Actions**

- Bash: `rtk git status -s; ./.harness/init.sh smoke_test 2>&1 | tail -5; rtk grep -n "^## Meta-Session" .harness/context_summary…`
- Bash: `rtk grep -n "^## Meta-Session" .harness/context_summary.md | tail -6; rtk read .harness/claude-progress.txt --tail-lines…`
- Bash: `awk '/^## Meta-Session/{print NR": "$0}' .harness/context_summary.md | tail -5; rtk wc -l .harness/context_summary.md`
- Read `.harness/context_summary.md`
- Read `.harness/mld/2026-11-06-16acdc43-d351-42a1-aea0-e5a1e9624a14.md`
- Bash: `rtk grep -n "Meta-Session" hooks/session-end.sh hooks/session-start.sh | head; rtk git show --stat --format='%h %s' 3da5…`
- Read `.harness/HARNESS_BACKLOG.md`
- Edit `.harness/context_summary.md`
- Edit `.harness/HARNESS_BACKLOG.md`

