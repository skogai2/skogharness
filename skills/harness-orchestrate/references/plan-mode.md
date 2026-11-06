# Plan Mode Launch & Acceptance (Claude Code & Codex)

Mechanics to run a worker through an interactive planning phase before a
hands-off implementation: launch the TUI in plan mode, submit the delegation
packet, watch and answer clarifying questions, review the plan, accept it, and
supervise hands-off until the worker reports done. Plan-first is opt-in — the
parent skill's activation gate (explicit user ask or `--plan-mode`) decides
whether this flow runs at all. The parent `harness-orchestrate` SKILL.md
carries the transport verbs and the workers-are-TUIs rule — both TUIs here get
their own named tab and launch through `herdr agent start`, never a headless
runner. Scope everything to the caller workspace; never change focus.

## Contents

- Shared rules (both TUIs)
- Claude Code: launch in plan mode
- Claude Code: accept the plan
- Codex: launch and enter Plan mode
- Codex: accept the plan
- Watch plan mode & answer questions
- Hands-off monitoring & completion

## Shared rules (both TUIs)

- TUI strings (status lines, acceptance menus, option labels) are
  build-specific. Verify them with `agent read` on first use against the
  installed version (`claude --version` / `codex --version`). If a
  newer build renames a mode or menu, adapt the keys/labels before relying on
  the flow rather than failing mid-loop.
- Submit inside a running TUI per the parent skill's *Sending prompts to
  running TUIs* rule: `agent prompt` submits text plus Enter atomically; a
  trailing `\n` alone never submits.
- Detect menus and questions two ways, semantic first: `agent wait <name>
  --until blocked` fires when the worker stops for input;
  `pane wait-output <pane_id> --match "<menu title>"` pins the exact menu. On
  either, `agent read <name> --source visible` before acting.
- Plan mode is interactive. The worker may ask clarifying questions before
  presenting a plan — answer them with the task context. Do not answer
  questions it did not ask.
- Review the plan before accepting: correctness, scope against the delegation
  packet's file claims, and no out-of-scope edits. Read with
  `agent read <name> --source recent-unwrapped --lines 200` if the plan is
  longer than one screen.
- **After acceptance, do not interrupt.** From the moment the plan is accepted
  until the worker reports done, only observe via read-only `agent read` and
  status waits. Send input only to answer a direct blocking question the
  worker surfaces or to handle a stop condition.

## Claude Code: launch in plan mode

`claude` has a first-class flag: `--permission-mode plan`. It replaces the
default `--permission-mode auto` for this launch (one mode per launch; never
add bypass flags, see SKILL.md launch workers). Acceptance then sets the mode
the worker runs in. Verified live with claude 2.1.318 and codex-cli 0.165.0,
herdr 0.9.5 (F003, 2026-11-06).

Create the named tab, then pass the packet as launch argv — the session starts
planning immediately and skips the TUI-ready race entirely:

```bash
herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" \
  --label "opus: fix loops (plan)" --no-focus

herdr agent start fix-loops --kind claude --pane <root_pane_id> -- \
  --permission-mode plan \
  "<packet, single block>"
```

The packet must be one line (SKILL.md launch workers). Expect `agent start` to
return `agent_not_ready` ("blocked during startup") on a small slice: the
worker can finish planning and reach the acceptance menu before herdr sees it
ready (seen live, F003: plan menu up within the 120000 ms start timeout). The
name stays usable; `agent read <name> --source visible` — if the screen shows
the acceptance menu, go straight to review. Any other blocking screen is a
trust prompt for the user (SKILL.md).

To launch without an initial prompt, drop the packet argument and submit with
`agent prompt` once `agent start` returns.

If the worker is not yet at the menu, confirm plan mode via `agent read <name>
--source visible`: the input-box status area must indicate plan mode. If a
session is already running outside plan mode, **shift+tab** cycles the
permission modes (the order is build-specific; not verified live) — press and
re-check until plan mode shows:

```bash
herdr agent send-keys fix-loops shift+tab
```

## Claude Code: accept the plan

When the worker finishes planning it presents an acceptance menu. Live with
claude 2.1.318 (F003):

```
Claude has written up a plan and is ready to execute. Would you like to proceed?
❯ 1. Yes, and use auto mode
  2. Yes, manually approve edits
  3. Tell Claude what to change
     shift+tab to approve with this feedback
ctrl+g to edit in Nvim · $CLAUDE_CONFIG_DIR/plans/<slug>.md
```

Catch it with `herdr pane wait-output <pane_id> --match "Would you like to
proceed?"` (or the `blocked` status wait). Read the full plan from the file
the menu's last line names (`$CLAUDE_CONFIG_DIR/plans/<slug>.md`) — the screen
truncates it — and copy it to the registry's evidence before accepting.
Review it, then select **Yes, and use auto mode** — workers run unattended in
auto mode; accept-edits ("manually approve edits") stalls on per-edit and
shell confirmations. Press Enter while it is highlighted (move with
`agent send-keys <name> down` / `up` if another option is selected):

```bash
herdr agent send-keys fix-loops enter
```

Confirm via `agent read` that the status line shows `auto mode on` and the
worker is working (`agent get <name>` → `working`). If it is not in auto mode
after acceptance, stop and report — never switch to bypass.

To reject and keep planning, select option **3, Tell Claude what to change**
(press `down` twice from option 1). It opens a feedback field in the menu
itself, so type the feedback with `herdr pane send-text <pane_id> "<feedback>"`
and submit with `agent send-keys <name> enter` (send-keys takes key names
only, never text). Do not use shift+tab there:
it approves the plan together with the feedback. Read the screen afterwards.
The worker should be back in plan mode, revising. This path was not exercised
in F003, so verify the field on first use.

## Codex: launch and enter Plan mode

Codex has no CLI flag for plan mode — it is entered inside the TUI with
**shift+tab**. Because of that, do **not** pass the packet as launch argv (the
session would start working in the default mode before the switch). Launch
bare into its named tab, enter Plan mode, then submit the packet:

```bash
herdr tab create --workspace "$HERDR_WORKSPACE_ID" --cwd "$PWD" \
  --label "codex: audit auth (plan)" --no-focus

herdr agent start audit-auth --kind codex --pane <root_pane_id> -- \
  --approve-for-me
```

`agent start` returns only once the TUI is ready for input. Then enter Plan
mode:

```bash
herdr agent send-keys audit-auth shift+tab
```

`agent read` again — the bottom status line must read **`Plan mode`**. If it
does not, send shift+tab again and re-check (the key cycles modes). Do not
submit the packet until `Plan mode` is confirmed. Then submit:

```bash
herdr agent prompt audit-auth "<packet, single block>" --wait --timeout 300000
```

## Codex: accept the plan

When the worker presents the menu:

```
Implement this plan?
> 1. Yes, implement this plan          Switch to Default and start coding.
  2. Yes, clear context and implement  Fresh thread.
  3. No, stay in Plan mode             Continue planning with the model.
```

Catch it with `herdr pane wait-output <pane_id> --match "Implement this
plan?"` (or the `blocked` status wait). Review the plan first, then select
option **1** (keeps the investigation context) by pressing Enter while it is
highlighted:

```bash
herdr agent send-keys audit-auth enter
```

Confirm the status line no longer shows `Plan mode` (live, codex-cli 0.165.0:
one shift+tab from the default mode showed `Plan mode (shift+tab to cycle)`;
after option 1 codex prints "Model changed to … for Default mode" and the
status line drops `Plan mode`). A small run may settle before you read it:
check `agent get` / the transcript rather than waiting to see `Working`. To reject, select option 3 and send feedback via `agent prompt`.

## Watch plan mode & answer questions

Block on `agent wait <name> --until blocked` in 3–5 minute check-in intervals
while the worker investigates (per the parent skill's *Track workers* rule —
the timeout is a check-in, never a deadline); on expiry, `agent read <name>
--source visible` and re-enter the wait. If the worker asks a clarifying
question, answer it via `agent prompt` using the delegation-packet context.
Update the state registry: `planning` while investigating, `plan-review` when
the acceptance menu appears, `plan-accepted` once accepted.

## Hands-off monitoring & completion

After acceptance, only observe. Block on the terminal states in 3–5 minute
check-in intervals — implementation, validation, and any QA regularly span
many intervals, and the worker is never cut off because a wait expired:

```bash
herdr agent wait fix-loops --timeout 300000  # returns on idle, done, or blocked
```

One wait covers both cases: without `--until`, herdr matches `idle`, `done`
and `blocked`, so a finished worker (claude settles `done`, codex `idle`) and a
worker stuck on a question both return it; `agent get` tells which. On expiry,
read the screen and re-enter the wait. On `blocked`, read the
screen and answer only a direct blocking question; on a stop condition (errors
it cannot resolve, out-of-scope request), stop and report. When the worker
reports done, verify its claims per the parent skill: re-open cited files,
inspect current command evidence, and review the diff before accepting the
patch. Rerun only checks whose evidence is missing, stale, or unreliable, plus
uncovered integration checks. Keep the worker for planned integration follow-ups;
record the final disposition and retire it per the parent skill.
