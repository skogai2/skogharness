# harness mode: driving vv-harness features through herdr workers

applies when the caller workspace is a harness project (`.harness/features.json`
exists). the controller is the harness **lead**; every rule in
`${CLAUDE_PLUGIN_ROOT}/rules/parallel-work.md` ("Lead-owned state", "Task mirroring
and integration order", "Worktree hygiene") binds it exactly as it binds workflow
mode. this file only says how those rules map onto herdr workers.

## 1. select the batch

read `.harness/features.json` and keep only features that are:

- `status` `pending` or `failed`
- **verified**: `spec.verdict == "PASS"` **and** no spec drift — if
  `spec.hash` is set it must equal `sha256(description)` (the same check
  `hooks/session-start.sh` runs). a drifted or unverified feature is excluded and
  reported; run `/vv-harness:harness-issue-prep` on it first.
- **unblocked**: every id in `depends_on` is already `passing`.

two features may run in parallel only if their `scope` lists do not overlap
(compare path prefixes: `src/auth/` overlaps `src/auth/token.ts`). overlapping or
dependent features go to the same worker, in `depends_on` order, or wait for a
later round. one feature = one worker = one worktree; never split a feature
across workers.

## 2. prepare state (lead, main checkout, before any launch)

1. `git fetch` and confirm a clean tree; record `merge_base=$(git rev-parse HEAD)`.
2. for each feature: `TaskCreate` with `metadata.feature_id` (and `addBlockedBy`
   from `depends_on`). this is what arms the lead's TaskCompleted gate later.
   if `TaskCreate` is unavailable in this session (seen live, F125), say so,
   record it in the registry and the handoff, and run the gate's checks by hand
   at integration step 6: `./.harness/init.sh focused_test <test_file>` and the
   coverage check the gate would have run.
3. set `assigned_to: "herdr:<agent-name>"` and `status: "in-progress"` in
   `features.json`, then commit that metadata change on its own.
4. create one herdr worktree per feature, pinned to the recorded base so every
   worker and every later scope-diff share it:

   ```bash
   herdr worktree create --workspace "$HERDR_WORKSPACE_ID" \
     --branch "feat/<id>-<slug>" --base "$merge_base" --label "<id>" --no-focus
   ```

   then rename the returned workspace's root tab to the worker's label and start
   the worker in its root pane (see worktree isolation in `SKILL.md`). record
   the worktree workspace id in the registry; removal needs it. if herdr asks for repository trust, stop and ask
   the user; `--trust-repository` is never a routine retry.

## 3. build the packet

each packet is assembled mechanically from the feature record — the controller
adds nothing it has not read from `features.json` or the repo. fill the
template below into a file, then launch with `"$(tr '\n' ' ' < packet.txt)"`:
herdr rejects newlines in launch argv, and feature descriptions contain both
newlines and quotes. keep the file as evidence. template, with fields filled
from the feature:

```text
you are a vv-harness WORKER, not the lead. feature {id} in worktree {worktree_path} on branch {branch} (base {merge_base}).
SPEC (verified, verbatim — this is the whole contract): {description}
SCOPE (you may edit only these paths): {scope joined by ", "}
OUT OF SCOPE: everything else, and always .harness/features.json, .harness/context_summary.md, .harness/claude-progress.txt, .harness/harness.json, .harness/mld/.
DEPENDS ON (already merged, rely on as-is): {depends_on with each one's description, or "none"}
TESTS: write the failing test first; put it in {test_file or "a new test file under the project's test layout"}; run ./.harness/init.sh focused_test <file> and ./.harness/init.sh smoke_test.
DO NOT run /harness-continue, claim other features, merge, push, or change feature status — ignore the orientation's "next claimable" line.
COMMIT your work on {branch} (git add and git commit as separate calls).
REPORT when done, as a markdown file at {report_path}: files_changed, test_file, commands run with exit codes, coverage if measured, approaches_tried, any needed scope expansion (as a request, not an edit), open uncertainty. STOP and report instead if you need an out-of-scope edit, a test fails 3 times for the same reason, or the spec is ambiguous.
```

the project's `.claude/` hooks (scope, git identity, task gates) run only in
claude workers. a codex worker is held to SCOPE by the packet alone, so the
integration scope-diff below is its only enforcement: never skip it.

risk routing (from `parallel-work.md` "Model policy" / "Dynamic overrides"):
`risk: "elevated"` or `require_plan_approval: true` → claude worker launched
with `--model opus`, plan-first (`references/plan-mode.md`) even if the user did not pass
`--plan-mode`; the plan is checked against SCOPE and every numbered AC before
acceptance. everything else → the runtime's configured model, direct mode.

## 4. integrate (lead, main checkout, one feature at a time)

for each worker whose report checks out, in `depends_on` order:

1. read its report file and the diff. **scope-diff**:
   `git diff --name-only <merge_base>...<branch>` — every file outside `scope`
   is accounted for (accepted as a recorded `scope_expansions`, or excluded)
   before merging.
2. elevated features: run the author-blind conformance check
   (`harness-continue` Step 5b, 3.5) before going further.
3. run that feature's focused test + smoke **on the worktree branch**, then merge
   it into the integration branch.
4. once no follow-up for that feature is pending: copy anything the registry
   still needs from the worker's screen, then retire it by removing the worktree
   before any repo-wide run:
   `herdr worktree remove --workspace <worktree_workspace_id>`. this deletes
   the worktree workspace with its tab and worker tui, so no separate `tab close`
   is needed; compare `herdr workspace list` before and after and confirm
   only that workspace went away.
5. in `features.json`: `status: "passing"`, `test_file`, `coverage`,
   `approaches_tried` (from the report), `scope_expansions`, clear `assigned_to`.
6. mark the mirrored task complete — the TaskCompleted gate runs here.
7. commit — the commit gate runs `full_test` because the commit flips a feature
   to passing.
8. confirm with `herdr agent list` that the worker is gone.

a rejected or blocked worker: leave `status` `in-progress` with a
`failure_reason`, or set it to `failed`, report it to the user with the report
path, and never merge its branch.

## 5. close the session

the normal harness session end applies unchanged: retrospective in
`context_summary.md`, MLD file, `claude-progress.txt` handoff. in the handoff,
include the herdr registry (agent name, tab label, model, branch, disposition)
per feature, so the next session can recover any unfinished worker branch.
