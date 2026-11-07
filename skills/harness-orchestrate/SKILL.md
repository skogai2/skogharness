---
name: harness-orchestrate
description: orchestrate claude and codex worker tuis from a controller agent — one worker per named herdr tab, driven over the herdr socket cli.
disable-model-invocation: true
metadata:
  author: skogix
  github: https://github.com/skogai
  repository: https://github.com/skogai2/skogai
---

# herdr orchestration

the top-level agent is the **controller** (codex, claude or another);
it delegates to **worker** tuis that gather evidence, draft bounded patches,
run tests, and report. the controller owns assignment, state, conflict
control, verification, integration, and the final user-facing answer. herdr is
the substrate: each worker tui owns a named tab, and the socket cli (`herdr …`)
gives the controller placement, input, screen reads, and native agent-status
waits. the installed binary is the authority on syntax: `herdr --skill` prints
herdr's own reference for that version, and `herdr <group> <verb> --help`
the exact flags.

## one worker, one named tab

every worker gets its own tab, labeled at creation with `tab create --label`.
the tab bar is the controller's dashboard: a labeled tab says which model is
running which slice, and stays readable no matter how many workers are live.

tabs also cost no screen area. a split pane divides the caller's screen with
every worker, and on a laptop the third worker leaves every pane too narrow to
read. never put a worker in a split pane unless the user explicitly asks to
watch it beside the caller.

this skill's layout deliberately differs from the herdr skill's default (sibling
pane, no new tabs): invoking this skill is the user's explicit request for it.

## one manager pane

besides the per-worker tabs, open **one** sibling pane next to the caller — the
manager pane — and route every ordinary command of the orchestration there:
integration test runs, `pane wait-output` targets, and a live worker board. the
caller pane stays the conversation; the manager pane is where the controller's
own non-agent work is visible to the user. create it once per orchestration,
with herdr's geometry rule (split a wide caller right, a narrow or tall one down):

```bash
herdr pane layout --pane "$HERDR_PANE_ID"
herdr pane split "$HERDR_PANE_ID" --direction right --cwd "$PWD" --no-focus
herdr pane run <manager_pane_id> "watch -n 10 herdr agent list"
```

record its pane id in the registry. stop the board (`pane send-keys
<manager_pane_id> ctrl+c`) before reusing the pane for a command. when the
caller pane is already too small to split usefully, skip the manager pane and
run those commands from a tab labeled `manager` instead. close the manager pane
when the orchestration is complete.

the manager pane runs the user's login shell, which may not be bash (fish
rejects `$?`): wrap every command as `bash -c '…'`. when waiting on its output,
match text the command prints, never text that also appears in the echoed command line — `pane wait-output --match
DONE` returns at once on `…; echo DONE`. prefer a `--regex` on the result shape
(`--regex "Summary: [0-9]"`).

two names per worker, both required at launch:

- **tab label** — `<model>: <slice>`, e.g. `opus: fix loops`,
  `codex: audit auth`. free text; keep it short, tab bars truncate.
- **agent name** — the slice as a slug, e.g. `fix-loops`. must match
  `[a-z][a-z0-9_-]{0,31}` and be unique among live agents. every `agent …` verb
  accepts it in place of a pane id.

## workers are tuis — no headless runners

a worker is an interactive tui — `claude` or `codex` — started with
`herdr agent start`, which validates the agent's identity and returns only once
herdr sees it ready for input. herdr's agent-state integrations hook those
tuis, so `agent wait`, `agent list`, and blocked/done detection exist only
while a real tui is on screen.

headless runners — `claude -p`, `codex exec`, anything that
streams json events into a pane — never report state, so waits never fire and
the delegation dies silently. a worker tab filling with raw json event lines is
a broken delegation: interrupt it (`herdr pane send-keys <pane_id> ctrl+c`)
and relaunch through `agent start`.

## invariants

- scope every action to the caller workspace unless the user names another
  target.
- resolve caller context from `$HERDR_WORKSPACE_ID`, `$HERDR_TAB_ID`, and
  `$HERDR_PANE_ID` (injected into every herdr pane) before any focused-window
  fallback.
- address workers by ids parsed from command json — agent name for `agent …`
  verbs, `w2:t3` / `w2:p4` for tab and pane verbs — never by guessed position.
- pass `--no-focus` on every creating verb (`tab create`, `workspace create`,
  `worktree create`, `pane split`); use focus verbs only when the user asks.
- retire every worker you launch when its assignment and planned integration
  follow-ups are finished (see retire workers).

## preflight

first confirm this controller runs inside a herdr-managed pane. if it does not,
say so and stop — never drive a focused herdr session from outside it:

```bash
test "${HERDR_ENV:-}" = 1        # herdr injects this into every pane it manages
```

in a vv-harness project the session-start orientation already reports this
(`herdr: inside herdr workspace …`); rerun the check anyway, since the pane can
differ from the one the session started in.

then inspect the daemon, integrations, and caller context:

```bash
herdr status                 # server running + socket path
herdr integration status     # selected worker runtime must show `current`
herdr pane current --current # caller pane / tab / workspace ids
herdr agent list             # agents already running
```

check/install only the integration for each selected runtime. reuse this
preflight in the same workspace while the daemon and integrations are unchanged;
a new packet alone does not require repeating it.

## launch workers

use each worker runtime's configured model unless the user (or harness risk
routing) requests an override; pass it as a native flag after `--` (claude
`--model opus`, codex `--model <model>`). launch is two commands: create the named tab, then start the tui in that tab's root pane.

```bash
# 1. named tab — returns .result.tab.tab_id and .result.root_pane.pane_id
herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" \
  --label "opus: fix loops" --no-focus

# 2. worker tui in that tab's root pane; native flags follow `--`
herdr agent start fix-loops --kind claude --pane <root_pane_id> -- \
  --permission-mode auto "<packet>"

herdr agent start audit-auth --kind codex --pane <root_pane_id> -- \
  --approve-for-me "<packet>"
```

capture `tab_id`, `pane_id`, and the agent name in the registry — retiring and
screen reads need all three.

workers launch in auto mode: claude with `--permission-mode auto`, codex with
`--approve-for-me` (approval requests go to codex's automatic reviewer inside
the workspace-write sandbox). workers run unattended without stalling on
routine prompts, while each runtime's own safety review stays on. never launch
with `--dangerously-skip-permissions`, `--permission-mode bypassPermissions`,
`--yolo` or `--dangerously-bypass-approvals-and-sandbox`: a claude controller in
auto mode refuses to start such a worker ("create unsafe agents"), and the
worker would have no guard at all. a worker that blocks on a prompt auto mode
would not clear is a stop condition: read its screen and report, do not
relaunch with bypass flags. plan-first claude runs use `--permission-mode plan`
instead (one mode per launch; see plan-first delegation).

`agent start` returns success only after herdr detects the agent ready. a
non-zero exit is not proof the launch failed — always read the tab's root pane
(`herdr pane read <pane_id> --source visible`) and `agent list` before
retrying. seen live (herdr 0.9.5, 2026-11-06):

- `agent_pane_busy` right after `tab create`: the tab's shell is not up yet.
  read the pane until a shell prompt shows, then retry the same `agent start`.
- `timeout` while the tui did start and is working: pass `--timeout 120000` to
  `agent start`. after a timeout herdr does not register the agent name
  (`agent_not_found`), so address that worker by pane id for every `agent …`
  verb and record that in the registry.
- `agent_not_ready` ("blocked during startup"): read the screen. a plan-first
  worker may already be at its plan acceptance menu (see
  references/plan-mode.md); go straight to plan review. otherwise the tui is on
  a first-run trust prompt. codex asks "trust this folder?" once per directory; claude
  asks about external CLAUDE.md imports once per project (after one answer it
  does not ask again; declined imports then silently stay off). these are
  trust decisions for the user: never answer them. notify
  (`herdr notification show "<worker> needs trust" --body "<tab label>"`),
  then re-enter `agent wait <pane_id> --until idle --until done` — a plain
  wait returns at once on the stale blocked state.
- `invalid_agent_argument` ("cannot be encoded safely"): the packet has
  newlines. launch packets must be a single line. build the packet in a file
  and pass `"$(tr '\n' ' ' < packet.txt)"`, which also keeps quotes and
  backticks in the packet away from the shell.

confirm the label stuck with `herdr tab list --workspace "$HERDR_WORKSPACE_ID"`; if the
tab shows a default label, restore it:

```bash
herdr tab rename <tab_id> "opus: fix loops"
```

## sending prompts to running tuis

prefer passing the initial packet as launch argv (shown above) — the session
starts working immediately and skips the tui-ready race. exception: codex
plan-first launches bare (see plan-first delegation).

for any follow-up, use `agent prompt` — it submits text plus enter atomically,
honoring the pane's live bracketed-paste mode, and `--wait` blocks until the
worker settles:

```bash
herdr agent prompt fix-loops "<follow-up prompt>" --wait --timeout 300000
```

a trailing `\n` in `pane send-text` does not submit — a tui renders it as a
soft newline and the prompt sits unsubmitted. an unconfirmed send is not
delivered: if `--wait` returns `agent_prompt_stalled`, the worker never changed
state — read the screen and resend.

## plan-first delegation (opt-in)

plan-first runs only when the user activates it — an explicit ask ("plan
first", "plan mode") or a `--plan-mode` flag on the invocation. the default
delegation is direct: launch and run hands-off. investigation complexity alone
does not add a planning approval step.

once activated, the worker plans, the controller reviews and accepts, then it
runs hands-off. the launch flags, shift+tab sequences, status checks, and
acceptance menus are exact and differ per tui. **read
[references/plan-mode.md](references/plan-mode.md) in full before launching a
plan-first worker.**

## delegation packets

send each worker prompt as a standalone contract. include:

- repo path and exact objective
- worker role and model
- execution mode: direct (default) or plan-first (user-activated)
- files, packages, or surfaces in scope
- files and behaviors explicitly out of scope
- claimed files or work slice, to avoid conflicts
- dependencies and the current interface/contract the worker can rely on
- expected evidence needed for this claim: changed files, commands with results
  and checked inputs, relevant artifacts, and remaining uncertainty
- verification ownership: worker checks, controller integration checks, and
  any final delivery checks owned by the enclosing workflow
- stop conditions: unexpected code shape, repeated command failure, auth/model
  blocker, or need for out-of-scope edits

assign coherent outcomes that can proceed independently. if a consumer depends
on an unsettled interface, resolve that interface first or keep producer and
consumer under one owner. file count alone is not a reason to split work.
tell workers they share the codebase, must preserve others' changes, and must
report interface changes before dependent work proceeds. reuse the same worker
for related integration fixes; send the changed contract and remaining work
instead of a fresh discovery assignment.

## harness projects

when the caller workspace contains `.harness/features.json`, the controller is
the vv-harness lead and workers implement tracked features. batch selection,
packet contents, and the integration order are fixed there, not improvised:
**read [references/harness-mode.md](references/harness-mode.md) in full before
launching any worker in a harness project.**

## worktree isolation

when workers must edit overlapping files in parallel, give each an isolated
worktree — herdr manages them natively, as its own labeled workspace:

```bash
herdr worktree create --workspace "$HERDR_WORKSPACE_ID" \
  --branch <slug> --label "<slug>" --no-focus --json
```

`worktree create` already opens that workspace with one root tab and pane
(`.result.tab.tab_id`, `.result.root_pane.pane_id`), cwd the isolated checkout:
rename that tab to the worker's label and start the worker in its root pane
instead of creating a second tab.

`herdr worktree remove --workspace <worktree_workspace_id>` deletes the whole
worktree workspace — its tabs and any worker tui in them, not just the checkout
(seen live, herdr 0.9.5, 2026-11-06). so a worktree worker needs no separate
`tab close`, but record its screen first. run `herdr workspace list` before
and after the remove, and confirm only that one workspace disappeared.

## track workers

orchestrations are long-running — worker slices take minutes to hours, and a
worker is never abandoned because a wait expired. block on herdr's native agent
status in **check-in intervals** of 3–5 minutes (sized to the slice — longer
for heavier slices) instead of polling screens:

```bash
herdr agent wait fix-loops --timeout 300000  # check-in: returns on idle, done, or blocked
herdr pane wait-output <pane_id> --match "<text>" --timeout 300000
```

controller reads never mark a tab seen, so an unfocused claude worker settles
as `done`, not `idle`; a codex worker was seen settling as `idle`. the plain
`agent wait` default (`idle`, `done`, `blocked`) matches either runtime's finish
**and** a worker stuck on a question or approval, so the controller learns of a
blocked worker at once instead of at the next timeout. on return, `agent get`
tells which state it was. narrow with `--until done --until idle` only for a
worker whose stale `blocked` state would return the wait at once (the
startup-trust case in launch workers). `--timeout` is the check-in interval,
never a deadline on the worker. on expiry, read the screen
(`herdr agent read fix-loops --source recent-unwrapped --lines 120`) and
re-enter the wait — loop until the worker reaches a terminal state, asks a
question, or a stop condition fires. when a reported status looks wrong, debug
detection with `herdr agent explain <pane_id> --json`.

maintain a compact registry in the existing task notes or handoff: controller
identity; worker agent name, tab label, role, model; workspace/tab/pane ids;
objective sent and start time; status (starting, planning, plan-review,
plan-accepted, running, blocked, reported, verified, retired); claimed files or
work slice; worker-reported commands and results; controller verification
performed; final disposition (accepted, rejected, superseded, or blocked).

keep the tab label current as a slice changes, and announce milestones without
stealing focus:

```bash
herdr tab rename <tab_id> "opus: fix loops (verifying)"
herdr notification show "workers done" --body "2/2 verified" --sound done
```

## verify worker output

treat worker output as untrusted until verified. monitor and verify without
changing focus:

- read worker tabs read-only (`agent read <name> --source recent-unwrapped
  --lines 200`); ask a worker for a concise status summary when output is
  unclear.
- re-open cited files locally to verify high-impact findings. for visual output,
  open the delivered images and their named visual references side by side;
  file counts, dimensions, and a worker's verdict do not establish visual fit.
  check one representative image before expanding an expensive generation batch.
- inspect command results, exit status, checked inputs, and evidence paths.
  reuse trustworthy worker evidence for unchanged inputs; rerun missing, stale,
  contradictory, or unverifiable checks. the controller runs integration checks
  the worker did not cover, plus any checks required by project policy. a worker
  handoff alone does not invalidate evidence or require a second full suite.
- review the final diff before accepting any worker patch.
- if a worker edits outside its claim, pause integration and decide: accept,
  request user-approved cleanup, or supersede with controller edits.

when `--lines` stops revealing more of a completed response, the tui is on the
terminal's alternate screen and those rows are gone. ask the worker to write
its full report as markdown in a temp directory and reply with the path, then
read the file.

## retire workers

a delegation ends with the worker retired — tab closed. keep a reported worker
available while reviewing or integrating its outcome when related follow-ups
are expected. retire it once the disposition is recorded and that work is done:

```bash
herdr tab close <tab_id>
```

closing the tab ends the tui session and its scrollback, so record what the
registry needs first — report text, cited line refs, command output. files,
diffs, and worktrees survive on disk.

a worker stays open while mid-run, blocked, awaiting planned integration
follow-ups, while its screen is evidence for an unresolved failure, or when the
user asks to inspect it — record the reason in the registry. the orchestration is complete only when
`herdr agent list` shows none of this controller's workers still running:
every worker retired, or its open tab justified in the registry.

## stop conditions

stop and report instead of improvising when:

- herdr rejects a valid workspace, tab, or pane ref, or `herdr status`
  shows no running server.
- a worker tui rejects the model alias, auth state, working directory, or a
  startup flag — report the exact blocker, and get user approval before
  downgrading a model or switching tools.
- authentication blocks the tui.
- a worker's status stays `unknown` while output scrolls and
  `agent explain` shows no matched rule — detection is broken or the pane is
  running a headless process.
- a plan-mode status line or acceptance menu cannot be confirmed via
  `agent read` or `pane wait-output`, or a worker's plan stays out of scope
  after a re-planning round.
- workers need overlapping edits the controller cannot integrate safely.
- a task needs focus-changing herdr verbs the user has not approved.
