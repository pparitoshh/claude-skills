---
name: brain-worker
description: Set up a Herdr brain + worker pair. Use when the user says "brain worker", "brain_worker", "/brain-worker", or asks for a supervisor agent and a worker agent in Herdr. Checks that this session runs inside a Herdr terminal, asks setup questions, then creates two windows (Herdr tabs or panes) - a "brain" supervisor and a "worker" executor. Brain and worker can each be Claude Code or OpenCode.
---

# Brain + worker in Herdr

The **brain** is a supervisor only. It plans, sends tasks to the worker, reads the results and checks them. It does not edit files or run builds itself.
The **worker** does the work: it edits code, runs commands and reports back.

The brain controls the worker with the `herdr agent` CLI. The installed binary is the authority for syntax: if a command fails, run `herdr agent` or `herdr pane` (no subcommand) to read the current help. Never run bare `herdr` - it attaches the TUI.

## Step 1 - Check the Herdr terminal

Run these checks. Stop at the first failure and tell the user what to do.

```bash
command -v herdr && herdr --version
test "${HERDR_ENV:-}" = 1 && echo inside-herdr || echo NOT-inside-herdr
herdr status
```

| Result | Action |
|---|---|
| `herdr` not found | Tell the user to install it: `curl -fsSL https://herdr.dev/install.sh \| sh`. Stop. |
| `NOT-inside-herdr` | Tell the user: start `herdr` in a terminal, then start Claude Code inside a Herdr pane and run this skill again. Stop. Do not control a Herdr session from outside. |
| `inside-herdr` | Continue. Record `$HERDR_WORKSPACE_ID`, `$HERDR_TAB_ID`, `$HERDR_PANE_ID`. |

Also check which agent CLIs exist: `command -v claude opencode`. Offer only installed CLIs in Step 2.

Check that the names `brain` and `worker` are free: `herdr agent list`. If one is in use, ask the user to reuse it, or use a suffix (for example `brain2`, `worker2`).

## Step 2 - Ask the setup questions

Use **one** `AskUserQuestion` call with these questions:

1. **Brain agent** (header "Brain"): `Claude Code` / `OpenCode`. Brain is supervisor only.
2. **Worker agent** (header "Worker"): `Claude Code` / `OpenCode`.
3. **Layout** (header "Layout"):
   - `Two tabs (Recommended)` - a new tab "brain" and a new tab "worker" in the current workspace.
   - `Two panes in one new tab` - brain on the left, worker on the right.
4. **Integrations** (header "State hooks"): `Install (Recommended)` / `Skip`. Herdr integrations let Herdr see the exact idle / working / blocked state of each agent. They write a hook file under `~/.claude/hooks/` or `~/.config/opencode/plugins/`.

Then ask in plain text (not a forced choice): "What is the first task for the brain? (Leave empty to start idle.)"

Map answers to Herdr kinds: Claude Code = `claude`, OpenCode = `opencode`.

## Step 3 - Install integrations (only if the user chose Install)

For each chosen kind that `herdr integration status` reports as `not installed`:

```bash
herdr integration install <kind>
```

## Step 4 - Create the two windows

Use `--no-focus` and the current directory. Read every ID from the JSON reply (for example with `jq -r`). Never guess IDs.

**Two tabs:**

```bash
B=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --label brain  --cwd "$PWD" --no-focus | jq -r .result.root_pane.pane_id)
W=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --label worker --cwd "$PWD" --no-focus | jq -r .result.root_pane.pane_id)
```

**Two panes in one new tab:**

```bash
B=$(herdr tab create --workspace "$HERDR_WORKSPACE_ID" --label brain-worker --cwd "$PWD" --no-focus | jq -r .result.root_pane.pane_id)
W=$(herdr pane split --pane "$B" --direction right --cwd "$PWD" --no-focus | jq -r .result.pane.pane_id)
```

If `jq` is missing, print the raw JSON and read the IDs from it.

## Step 5 - Start the agents

Start the worker first, so the brain can find it when it starts:

```bash
herdr agent start worker --kind <worker-kind> --pane "$W"
herdr agent start brain  --kind <brain-kind>  --pane "$B"
```

For an OpenCode agent, always select a model from the OpenCode Go provider (`opencode-go/...`). Do not use `opencode/...` models: their API key expired. Pass the model after `--`:

```bash
herdr agent start worker --kind opencode --pane "$W" -- --model opencode-go/glm-5.3-flash
```

List the available models with `opencode models | grep '^opencode-go/'`.

If a start returns `agent_not_ready` (for example a trust or login dialog), run `herdr agent read <name> --source visible`, show the screen to the user and ask what to do. Do not answer dialogs for the user.

## Step 6 - Give each agent its role

Send the worker role:

```bash
herdr agent prompt worker "You are the WORKER in a brain/worker pair run by Herdr. A supervisor agent named 'brain' sends you tasks. Do each task completely: edit files, run commands and tests. When you finish, end with a short report: what you changed, the files, the commands you ran and their results, and any open problems. Do not start new work that the brain did not ask for." --wait --timeout 120000
```

Send the brain role (replace `<TASK>` with the user's first task, or with "No task yet. Wait for the user."):

```bash
herdr agent prompt brain "You are the BRAIN, a supervisor only, in a brain/worker pair run by Herdr. Do NOT edit files or run builds yourself. Your worker is the Herdr agent named 'worker'. Control it only with these commands:
- Send a task and wait:  herdr agent prompt worker \"<task>\" --wait --timeout 1800000
- Check state:           herdr agent get worker
- Read its output:       herdr agent read worker --source recent-unwrapped --lines 200
- Stop it:               herdr agent send-keys worker esc
Your loop: 1) split the user goal into small, testable tasks; 2) send one task; 3) read the report; 4) verify it (read files, read diffs, ask the worker to run tests); 5) send a fix or the next task. If the worker is 'blocked' on an approval dialog, show it to the user and ask - do not approve it yourself. Report progress to the user after each task. First task: <TASK>"
```

Do not use `--wait` on the brain prompt when there is a real task, because the brain can work for a long time.

## Step 7 - Report to the user

Tell the user:

| Item | Value |
|---|---|
| Brain | `<kind>` in pane `$B` (agent name `brain`) |
| Worker | `<kind>` in pane `$W` (agent name `worker`) |
| Layout | tabs or panes |

Then give these commands:

- Switch to the brain: `herdr agent focus brain`
- Watch the worker: `herdr agent read worker --source recent-unwrapped --lines 80`
- Give the brain a new goal: `herdr agent prompt brain "<goal>"`

## Rules

- Never close tabs, panes or workspaces that this skill did not create.
- Never run `herdr server stop`.
- Keep the user's focus in the current pane (`--no-focus` everywhere).
