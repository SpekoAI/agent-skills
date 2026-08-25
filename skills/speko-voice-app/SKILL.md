---
name: speko-voice-app
description: Build an application on Speko — hosted voice agents that make and receive phone calls, with speech-to-text and text-to-speech behind one API. Use when the user wants to build, prototype, or wire up an app on Speko, or add Speko STT/TTS to one. Not for placing a single phone call on the user's behalf (use speko-phone-call), and not for STT/TTS work on another vendor. Needs SPEKO_API_KEY.
---

# Building a voice app with Speko

Speko provides telephony, hosted voice agents, speech-to-text, the language model, text-to-speech,
recording, and transcripts. You describe the agent and place calls over HTTPS; Speko routes each
request per language — you never pick or integrate an individual STT/TTS vendor.

Base URL: `https://api.speko.dev`. Auth: `Authorization: Bearer $SPEKO_API_KEY`.
Docs: `https://docs.speko.dev` (start at the quickstart). Every endpoint below has been exercised
against the live API; where a claim comes from docs instead, this file says so.

## Set the project up

1. Account + key: `https://platform.speko.ai/agents/keys` → mint a platform key (`sk_live_…`).
2. Put it in the project's environment as `SPEKO_API_KEY` (`.env`, never committed). Read it from
   the environment at runtime; never hardcode it in source, never log it, never include it in
   error output or a commit.
3. No SDK is required — plain `fetch`/`curl` works. Official SDKs exist; see the quickstart at
   `https://docs.speko.dev` for current install names.

## The core loop

**Before any dial that reaches a real person, the same rules as the `speko-phone-call` skill
apply: show the full E.164 number, say what the call is for, and get an explicit yes. One
confirmation, one call. Never loop this over a list of numbers.** When building and testing, dial
only numbers the user owns, or use a test conversation with no phone leg.

**Start with ad-hoc calls — no agent object needed.** One POST both defines the behaviour and
dials:

```bash
curl -sS --fail-with-body -X POST https://api.speko.dev/v1/sessions/phone \
  -H "Authorization: Bearer $SPEKO_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "to": "+14155550142",
    "intent": {"language": "en"},
    "systemPrompt": "You are the booking assistant for Blue Fin Sushi…",
    "firstMessage": "Hi, this is the Blue Fin booking assistant."
  }'
# → {"sessionId": "…", "status": "dialing", …}
```

It returns before the call ends. Poll `GET /v1/sessions/{sessionId}` every 5 seconds until the
session has ended (cap the loop — 10 minutes, then stop and report), then read the outcome:

- `GET /v1/calls/{id}` — status, `duration_seconds`, `transcript`, `recording_resource_uri`
- `GET /v1/calls/{id}/report` — `summary`, `outcome`, `structured_data`, per-utterance
  `transcript.entries[{text, source: "agent"|"user", startedAt}]`

For production, register a webhook instead of polling — `POST /v1/webhooks` with `name` and `url`
(both required; `GET /v1/webhooks` lists what is registered) — and store the `sessionId` your app
gets back from the dial.

**Promote to a persistent agent when behaviour repeats.** Create it once in the dashboard
(`https://platform.speko.ai/agents`) or via `POST /v1/agents` (requires `name` and
`systemPrompt`), then dial with `{"to": …, "agentId": …}` instead of re-sending the prompt.
Versioning, deploys, monitors, and evals for agents live on the platform side — see
`https://docs.speko.dev`.

**Inbound**: `GET /v1/phone-numbers` (kebab-case — `phone_numbers` 404s) lists the account's
numbers with their allowed `direction` (`both`, `inbound`, `outbound`). Attaching a number to an
agent and buying numbers both happen in the dashboard.

## Speech endpoints inside your app

The same key drives one-shot speech, useful far beyond telephony:

- `POST /v1/synthesize` `{text, intent: {language}, sampleRate: 24000}` → **headerless PCM**
  (`audio/pcm;rate=24000`). It is not a playable file until you wrap it
  (`ffmpeg -f s16le -ar 24000 -ac 1 -i pipe:0 out.wav`). There is no `response_format` parameter.
- `POST /v1/transcribe` — body is raw audio bytes, routing goes in an `x-speko-intent:
  {"language": "en"}` header, and the answer is **server-sent events** — the transcript is the
  `text` field of the final `data:` line. Feed it a mono 24 kHz WAV file — other containers
  return `200` with an empty transcript rather than an error.

Both are demonstrated with working, failure-safe pipelines in the `speko-phone-call` skill in
this repo.

## The traps that cost real time

- **Errors carry `{"error", "code"}`**, plus an `issues: [{path, message}]` array on
  `VALIDATION_ERROR` — fix the named path; do not retry blind.
- **Keys are environment-scoped.** A key minted for one environment answers 401 on another.
  `GET /v1/organization` with your key is the cheapest "is this key alive here" probe.
- **A bogus `agentId` in a dial body returns `404 AGENT_NOT_FOUND` without ringing anyone** —
  the free way to integration-test the dial path in CI.
- **Every real call opens with an AI disclosure applied server-side.** Do not build UX that
  promises to remove it. Calls are recorded by default; surface that in your product's UX.
- **Do not ship unattended or bulk dialing.** Each outbound call to a real person is one
  explicitly confirmed action.
- Test conversations without a phone leg exist on the platform; use them to rehearse prompts
  before dialing people.

## If your agent speaks MCP

Claude Code, Codex, Cursor, and other MCP clients can mount Speko's hosted server instead of raw
HTTP: `https://mcp.speko.ai/mcp` with `"type": "http"` (OAuth sign-in, or a platform API key).
It exposes the same platform — agents, sessions, calls, transcripts, synthesis — as MCP tools,
plus `docs.search` for in-band documentation. The REST surface above stays the portable
lowest-common-denominator.
