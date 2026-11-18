# Setting up the session bridge

One-time wiring so a **primary** session on one machine can drive a **secondary**
session on another. Do these steps on each machine once; after that, use the main
skill to dispatch work.

## Requirements

- **Subscription:** Pro, Max, Ultra, Team, or Enterprise. API keys are **not** supported.
  On Team/Enterprise an owner must enable Remote Control in admin settings first.
- **Sign-in:** signed in via claude.ai (`/login`) on both machines, to the **same
  account** — sessions are scoped to the signed-in account, so both ends must share one
  to see each other.
- **Versions** (`claude --version`):
  - Remote Control: 2.1.224+ on macOS and Linux/WSL 2; **2.1.234+ on native Windows**.
  - Sending a task to another machine: sender on 2.1.225+.
  - `@`-mention typeahead for remote sessions: 2.1.232+.
- **Not available** on Amazon Bedrock, Google Vertex, or Microsoft Foundry, when
  `ANTHROPIC_BASE_URL` points at a non-Anthropic gateway/proxy, or in organizations with
  Zero Data Retention or HIPAA configuration.

## Step 1 — Sign in to the same account (both machines)

```
/login
```

Use the same claude.ai account on both. Check with `/status` if unsure which account a
session is on.

## Step 2 — Turn on Remote Control (both machines)

Any one of:

- Inside a running session: `/remote-control` (alias `/rc`).
- Start a session with it: `claude --remote-control` (alias `claude --rc`).
- Run several controllable sessions from one process (server mode): `claude remote-control`.

To make every future session connect automatically, set it once:

- `/config` → **Enable Remote Control for all sessions** → on, or
- add to the user settings file (`~/.claude/settings.json` on macOS/Linux,
  `%USERPROFILE%\.claude\settings.json` on Windows):

```json
{ "remoteControlAtStartup": true }
```

A process must stay running to stay reachable — closing its terminal takes that session
offline. For a session started over SSH, run it inside `tmux` or `screen` so it survives
a disconnect.

## Step 3 — Name the secondary session

Give the session you'll be driving a stable, memorable name so it's easy to address:

- At start: `claude --remote-control "worker"` (or `claude --name "worker"`).
- On a running session: `/rename worker`.

A name set at **startup outranks** a later `/rename`. If `/rename` doesn't seem to take,
relaunch the session with the name instead. After renaming, run `/list-agents` to confirm
the new name shows.

## Step 4 — Set the secondary's autonomy

The secondary acts on messages under **its own** permission mode, and a remotely sent
task **cannot approve permission prompts**. Decide how hands-off it should be:

- **Accept incoming tasks:** set `crossSessionInbound` to `accept` on the secondary
  (options: `accept`, `hold`, `refuse`).
- **Let it act without prompting:** run the secondary in a permission mode that allows
  the work it will be asked to do. Grant the minimum that avoids stalls — a session
  acting on relayed instructions should not be more powerful than the task needs.
- **Guardrail on the primary:** `isolatePeerMachines: true` makes the primary ask you
  before any message leaves the machine.

## Step 5 — Verify the link

On each machine:

```
/list-agents
```

Each should list the other's session, by name, as **online**. Send a trivial task from
the primary ("reply with OK") and confirm the secondary answers. The bridge is up.

## Platform notes

- **Same machine, two platforms:** a WSL 2 session and a native-Windows session on one
  computer **cannot** message each other. Keep a pair on the same platform, or on
  different machines.
- **Network:** Remote Control makes **outbound HTTPS on 443 only** and opens no inbound
  ports. A firewall or proxy that blocks that outbound connection prevents a session from
  coming online.
- **Where messages go:** across machines, through Anthropic's servers; on the same
  machine, over a local socket (a Unix socket, or a named pipe on native Windows).

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Target not in `/list-agents` | Not connected, or a different account. Repeat steps 1–2 on it; confirm the same account. |
| Target shows `offline` | Its process closed or lost connection. Restart the session (step 2). |
| Reply never arrives | Primary isn't connected. `/rc` on the primary, then re-dispatch. |
| Secondary receives but does nothing | `crossSessionInbound` not `accept`, or its permission mode is prompting. See step 4. |
| Works on macOS, not native Windows | Native Windows needs 2.1.234+. Check `claude --version`. |
