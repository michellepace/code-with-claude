---
description: Set up the Agent Battle stack (deps, server, bot, relay) and get the participant to "ready"
---

You are setting up the Agent Battle workshop stack for a participant.
Your job: get them from a fresh clone to `▸ ready` with no manual
debugging on their part. Be adaptive — every machine is a little
different.

## Steps

**1. Check the participant env vars** that must be exported:

```bash
for v in PARTICIPANT MINECRAFT_EULA; do
  if [ -z "${!v:-}" ]; then echo "MISSING: $v"; else echo "OK: $v"; fi
done
[ -f .env.event ] && grep EVENT_URL .env.event || echo "NO .env.event"
```

If any are MISSING, stop and tell them what to export:
- `ANTHROPIC_API_KEY` — their own key from console.anthropic.com
  (optional if their environment has SDK OAuth/workload-identity auth)
- `PARTICIPANT` — any unique name they pick
- `MINECRAFT_EULA` — must be `accept`. Show them
  https://www.minecraft.net/eula and ask whether they agree.
  If yes, they run `export MINECRAFT_EULA=accept`. **You must
  not set this for them or assume agreement** — the user has to
  affirmatively accept the EULA themselves.

`EVENT_URL`, `LEADERBOARD_KEY`, and `MC_SEED` come from the
`.env.event` file in this directory — `setup.sh` reads it
automatically. Do NOT block on these. (If `.env.event` has an empty
`EVENT_URL`, setup runs in SOLO mode — local event server + one
quick-tunnel — which is correct for at-home practice.)

Do NOT proceed until PARTICIPANT and MINECRAFT_EULA are set
(ANTHROPIC_API_KEY is optional).

**2. Run setup** and capture the full output:

```bash
./setup.sh 2>&1
```

**3. Decide based on what you see:**

If the last line is the boxed `OPEN THIS IN YOUR BROWSER` block →
**success.** Tell them:
- Open `http://localhost:8088/view` in a browser (their bot's camera)
- Open `my_agent.py` and find the `AGENT = dict(...)` block — that's
  what they'll edit
- Run `python3 my_agent.py` — every run is 5 min and posts to the
  leaderboard; best run counts, so iterate freely
- `python3 my_agent.py --eval` for a ~30-60s scorecard with no run
- You're standing by if anything breaks — they can type `/cwc-fix`

If you see `✗` (a `die()` line) → **failure.** Match the error to the
playbook in `.claude/CLAUDE.md` and remediate. Common cases:

| Error pattern | Fix |
|---|---|
| `java not found` | macOS: `brew install openjdk@21 && sudo ln -sfn $(brew --prefix openjdk@21)/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/`. Linux: `sudo apt install openjdk-21-jdk`. Then re-run. |
| `npm install (bot) failed` | Likely a corp registry. Run `(cd bot && rm -rf node_modules package-lock.json && npm install --registry=https://registry.npmjs.org)` then re-run `./setup.sh`. |
| `server failed — see /tmp/mc-server.log` | Read that log. If `Address already in use`: stale java holding :25565 → see `.claude/CLAUDE.md` "Killing leftovers". If `Unsupported class file`: Java too old, install 17+. |
| `bot failed — see /tmp/mc-bot.log` | Read that log. Usually `Cannot find module` (npm install incomplete) or `ECONNREFUSED` (server not up yet — wait 10s, re-run). |
| `bot did not register with the relay` | Read `/tmp/mc-bot.log`, look for `[relay]` lines. Check `curl $EVENT_URL/api/config` works from their machine. If the event server is fine but WebSockets can't connect, the venue network may block them — escalate to the facilitator. |
| `event server failed` (SOLO mode) | Read `/tmp/event-local.log`. Usually a missing `(cd event && npm install)`. |
| `tunnel failed` (SOLO mode only) | Transient cloudflared issue. Re-run `./setup.sh`. If `/tmp/cloudflared` is missing or zero-byte, delete it so `tunnel.sh` re-downloads. |
| `ANTHROPIC_API_KEY not set` warning | Fine to proceed if their environment has SDK auth; otherwise have them export it. |

After remediating, re-run `./setup.sh` (it's idempotent). Loop until
`▸ ready` or you hit something not in the playbook — then explain what
you see and ask if they want you to dig in.

## Don't

- Don't edit `my_agent.py` for them. Setup only.
- Don't suggest what to put in `AGENT["system"]`. That's the workshop.
- Don't run `python3 my_agent.py` for them. Hand off after `▸ ready`.
- Never echo their `ANTHROPIC_API_KEY` value into the chat.
