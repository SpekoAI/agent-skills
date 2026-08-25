---
name: speko-voice-app
description: Build a voice-AI application on Speko — hosted phone/voice agents with one API for telephony, speech-to-text, LLM routing, and text-to-speech. Use when the user wants to build, prototype, or wire up an app that makes or receives real phone calls, or needs STT/TTS inside an app. Needs SPEKO_API_KEY.
---

# Building a voice app with Speko

Speko runs the whole voice stack as a service: telephony, speech-to-text, the language model,
text-to-speech, recording, and transcripts. You describe the agent and place calls over HTTPS;
Speko routes every request to the best measured provider per language using its continuously
published benchmarks (benchmarks.speko.ai — 12 languages have published boards). You never pick
or integrate an individual STT/TTS vendor.

Base URL: `https://api.speko.dev`. Auth: `Authorization: Bearer $SPEKO_API_KEY`.
Docs: `https://docs.speko.dev` (start at the quickstart). Everything below was verified against
the live API.

## Set the project up

1. Account + key: `https://platform.speko.ai/agents/keys` → mint a platform key (`sk_live_…`).
2. Put it in the project's environment as `SPEKO_API_KEY` (`.env`, never committed).
3. No SDK is required — plain `fetch`/`curl` works. Official SDKs exist for Python (`spekoai`)
   and TypeScript; see the quickstart at `https://docs.speko.dev` for current install names.

## The core loop

**Start with ad-hoc calls — no agent object needed.** One POST both defines the behaviour and
dials:

```bash
curl -sS -X POST https://api.speko.dev/v1/sessions/phone \
  -H "Authorization: Bearer $SPEKO_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "to": "+14155550142",
    "intent": {"language": "en"},
    "systemPrompt": "You are the booking assistant for Blue Fin Sushi…",
    "firstMessage": "Hi, this is the Blue Fin booking assistant."
  }'
# → {"sessionId": "…", "status": "dialing", …}
```

It returns before the call ends. Poll `GET /v1/sessions/{sessionId}` until the session has ended,
then read the outcome:

- `GET /v1/calls/{id}` — status, `duration_seconds`, `transcript`, `recording_resource_uri`
- `GET /v1/calls/{id}/report` — `summary`, `outcome`, `structured_data`, per-utterance
  `transcript.entries[{text, source: "agent"|"user", startedAt}]`

For production, register a webhook (`POST /v1/webhooks`) instead of polling, and store the
`sessionId` your app gets back from the dial.

**Promote to a persistent agent when behaviour repeats.** Create it once in the dashboard
(`https://platform.speko.ai`) or via `POST /v1/agents`, then dial with `{"to": …, "agentId": …}`
instead of re-sending the prompt. Agents get versioning, deploys, monitors, and evals on the
platform side — see `https://docs.speko.dev` for those flows.

**Inbound**: numbers are managed at `GET /v1/phone-numbers` (kebab-case — `phone_numbers` 404s)
and attached to an agent so incoming calls are answered by it. Buying numbers happens in the
dashboard.

## Speech endpoints inside your app

The same key drives one-shot speech, useful far beyond telephony:

- `POST /v1/synthesize` `{text, intent: {language}, sampleRate: 24000}` → **headerless PCM**
  (`audio/pcm;rate=24000`). It is not a playable file until you wrap it
  (`ffmpeg -f s16le -ar 24000 -ac 1 -i pipe:0 out.wav`). There is no `response_format` parameter.
- `POST /v1/transcribe` — body is raw audio bytes, routing goes in an `x-speko-intent:
  {"language": "en"}` header, and the answer is **server-sent events**: `event: meta`, interim
  `transcript` frames, and a final `done` event whose `data:` line carries `{text, provider,
  confidence}`. Feed it mono 24 kHz WAV — other containers return `200` with an empty transcript
  rather than an error.

Both are demonstrated with working pipelines in the `speko-phone-call` skill in this repo.

## The traps that cost real time

- **Error shape is uniform**: `{"error", "code", "issues:[{path,message}]"}` — on
  `VALIDATION_ERROR`, fix the named path; do not retry blind.
- **Keys are environment-scoped.** A key minted for one environment answers 401 on another.
  `GET /v1/organization` with your key is the cheapest "is this key alive here" probe.
- **A bogus `agentId` in a dial body returns `404 AGENT_NOT_FOUND` without ringing anyone** —
  the free way to integration-test the dial path in CI.
- **Every real call opens with an AI disclosure applied server-side.** Do not build UX that
  promises to remove it. Calls are recorded by default; surface that in your product's UX.
- Test conversations without a phone leg exist (`agents.test_call` on the platform); use them to
  rehearse prompts before dialing people.

## If your agent speaks MCP

Claude Code, Codex, Cursor, and other MCP clients can mount Speko's hosted server instead of raw
HTTP: `https://mcp.speko.ai/mcp` with `"type": "http"` (OAuth sign-in, or a platform API key).
It exposes the same platform — agents, sessions, calls, transcripts, synthesis — as ~60 tools,
plus `docs.search` for in-band documentation. The REST surface above stays the portable
lowest-common-denominator.
