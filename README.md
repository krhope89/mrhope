# Mr. Hope: Public Builds & Experiments

Practical AI and automation experiments. Mostly self-hosted, mostly things I
wanted to exist and couldn't buy.

## What's here

**[Hermes](HERMES.md).** A self-hosted AI agent running in Docker on a Raspberry
Pi 5, reachable over Telegram, with OpenRouter for inference and a local Ollama
fallback. The build notes cover the architecture, the agent's boundaries and why
they're structural rather than advisory, and the two gotchas that cost me the
most time.

More as I go.

## Ground rules

What I try to hold to when building this stuff:

- Ship the rough version and fix it against real use, not against an imagined user
- Decide up front what the agent owns and what it hands back to a person
- Prefer failures that are loud. The expensive ones present as silence
- Measure adoption, not just time saved. A workflow nobody uses saves nothing
- Write down what didn't work, since that's the part nobody publishes

## Elsewhere

Longer write-ups at [mrhope.ca](https://mrhope.ca).
