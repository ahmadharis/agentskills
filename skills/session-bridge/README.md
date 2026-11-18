# session-bridge

Hand a task from one Claude Code session to another running on a different machine, and
get the result back — built on Claude Code's **Remote Control** and **cross-session
messaging**.

## What It Does

Lets a **primary** session dispatch work to a **secondary** session on another machine
and collect the reply:

1. **Confirms** the secondary session is connected and online (`/list-agents`)
2. **Dispatches** a task to it by name — plain language or `@mention`
3. **Collects** the result it messages back

The one-time wiring to connect two machines lives in [references/setup.md](references/setup.md).

## When It Triggers

Claude uses this skill when you say things like:

- "Run this on the other machine"
- "Tell the `worker` session to …"
- "Hand this off to the other session"
- "Get the build box to run the tests and report back"

## Prerequisites

- Both sessions connected via **Remote Control**, signed into the **same** claude.ai
  account, on a Pro / Max / Ultra / Team / Enterprise plan (API keys unsupported).
- First-time connection steps: [references/setup.md](references/setup.md).

## Notes

- A remotely dispatched task **cannot approve permission prompts** on the secondary —
  set its autonomy during setup.
- For replies to return, the **primary** must also be connected to Remote Control.
- A WSL and a native-Windows session on the *same* computer cannot message each other.

## License

[MIT](../../LICENSE)
