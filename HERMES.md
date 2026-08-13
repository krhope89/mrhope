# Hermes: Running a Self-Hosted AI Agent on a Raspberry Pi 5

Build notes from running [Hermes](https://github.com/NousResearch/hermes-agent) (NousResearch) in
Docker on a Pi 5, reachable over Telegram. Started May 2026, still running.

I wanted an agent I actually controlled: my hardware, my data, my kill switch. Most of what I
learned had nothing to do with the model. It was about where the agent's authority stops.

---

## Architecture

```
Telegram  <->  Telegram API (cloud)
                    |  polling, no inbound port needed
              hermes container (Pi 5)
                    |
              OpenRouter API  ->  model
                    |  fallback
              Ollama (local, on the Pi)
```

A few deliberate choices:

- **Polling, not webhooks.** The agent polls Telegram's servers, so nothing inbound is open on my
  home network. No port forwarding, no reverse proxy, no attack surface.
- **The gateway API stays internal.** It is never exposed to the internet.
- **Secrets live only on the Pi**, in a data volume that is gitignored and never pushed.
- **A user whitelist is the real security control.** The agent only answers messages from
  explicitly listed user IDs. Without this, a Telegram bot token is effectively a public endpoint
  to your agent.

---

## Boundaries: what the agent is allowed to do

This is the part worth reading, and it is the part I would get right first if I were doing it again.

The agent runs *inside* its container and has no host or Docker access. It cannot run `docker`,
cannot see the real `docker-compose.yml`, and cannot edit the environment file that configures it.

That matters because of a failure mode I did not anticipate. There is a copy of the deployment
config *inside* the container image. When the agent tried to fix its own problems, it would edit
those in-container files, and the edits looked like they worked. The file changed. Behaviour
sometimes changed. But those files are vendored image content, not the real deployment config, so
the changes did nothing to how the container actually launches, and they vanished on the next
update or container recreate. They survived a plain restart, which is exactly what made them seem
durable right up until they weren't.

So the rule became: when the agent identifies a fix that needs a host-side change, it reports the
exact variable, file and value required, and then stops. It does not patch in-container files and
it does not relaunch its own supervised processes as a workaround.

That rule is written up as a persistent skill the agent loads every session, so the boundary is
part of its context rather than something I have to re-explain.

**The general lesson:** an agent that can partially reach its own infrastructure is more dangerous
than one that cannot reach it at all, because the partial access produces changes that look
successful and silently aren't. Decide what the agent owns and what it hands back to a human, then
make the boundary structural rather than advisory.

---

## The root versus agent-user problem

The container has two users: `root`, and the low-privilege user the agent is designed to run as
(UID 10000). On startup the entrypoint hands control from root to that user, which is correct. You
do not want an AI agent running as root.

But `docker exec` skips the entrypoint. Open a shell that way and you land as root, and every file
you touch gets written as `root:root`. Later, the agent (running as UID 10000) cannot read them.

This caused essentially every problem I had in week one, and each one failed quietly rather than
loudly:

- auth file written as root, so OAuth tokens could not be read and the model failed silently
- config file written as root, so model and fallback config never loaded, producing 400s
- skill files written as root, so custom skills silently did not load
- history file written as root, so the CLI crashed on every keystroke

**Fix going forward:** always pass `-u <agent-user>` to `docker exec` so you land as the right UID.

**Fix for already-broken files:** recursively chown the whole data directory back to the agent's
UID. Safe to re-run, and it is the first thing I try whenever behaviour goes strange.

---

## Two gotchas that cost me real time

**Auxiliary models default to expensive.** Setting an OpenRouter key silently activated an
auxiliary provider for background work like context compression and title generation, and it
defaulted to a frontier model. Housekeeping tasks were running on the most expensive thing
available. The fix is to pin auxiliary tasks to a cheap model in `config.yaml`. Note that the
environment variables for this are deprecated, so config file is the only supported route.

**Environment changes need a recreate, not a restart.** `docker restart` reuses the existing
container's baked-in environment and will not pick up changed variables. You need
`docker compose up -d --force-recreate`. The symptom if you forget is maddening: the variable is
plainly correct in your env file, and `printenv` inside the container returns nothing.

---

## What I would tell someone starting this

- Decide the agent's boundaries before you give it capabilities, not after it surprises you.
- Prefer failures that are loud. Most of my week-one problems were permission errors that
  presented as silence.
- Whitelist users on day one. A bot token with no allow-list is a public door.
- Keep the expensive model on the work that needs it and check what your tooling picked by default.
- Write down what broke. Six months later that log is worth more than the code.

---

Longer write-ups at [mrhope.ca](https://mrhope.ca).
