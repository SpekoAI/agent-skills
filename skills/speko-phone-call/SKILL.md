---
name: speko-phone-call
description: Place a real outbound phone call through Speko and read back what was said. Use when the user wants someone actually called on the telephone — booking, confirming, chasing, asking a business a question — or wants to review a call that already happened. Needs SPEKO_API_KEY.
---

# Placing a phone call with Speko

Speko dials a real phone number with a hosted voice agent, stays on the line, and returns a
transcript and a recording when the call ends. Everything here is plain HTTPS against
`https://api.speko.dev` — no SDK required.

## When this applies

Use it when the user wants a real telephone call made or reviewed: "call them", "ring the clinic",
"book me a table", "chase the quote", "find out if they're open", "what did they say on that call".

Do **not** use it to send a message, an email or a text — Speko is voice on the telephone network.

## Setup

One credential: a Speko platform API key (`sk_live_…`), minted at
`https://platform.speko.ai/agents/keys`. Export it as `SPEKO_API_KEY` and pass it as
`Authorization: Bearer $SPEKO_API_KEY`. Never print the key, never commit it, never paste it into
a file the user did not ask for.

## Before you dial: three things, every time

**1. Read the number back.** Show the full E.164 number — `+14155550142`, not "the dentist".
A wrong digit means a stranger's phone rings.

**2. Say what the call is for.** One sentence, in the user's words, so they can catch a
misunderstanding before it is spoken to a person.

**3. Get an explicit yes.** One confirmation, one call. Never batch, never loop over a list of
numbers, never redial on your own. If the user gives several numbers, take them one at a time
with a separate confirmation each.

If the user has given a business name but no number, ask for the number. Do not guess one or look
one up and dial it without showing it first.

## Making the call

```bash
curl -sS -X POST https://api.speko.dev/v1/sessions/phone \
  -H "Authorization: Bearer $SPEKO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "+14155550142",
    "intent": {"language": "en"},
    "systemPrompt": "You are calling La Taqueria to book a table for 2 at 7pm tonight under the name Sam. Confirm the booking and ask about parking.",
    "firstMessage": "Hi! I am calling to book a table for tonight."
  }'
```

Only `to` is required, plus either an `agentId` (an agent the account already has) or an `intent`
naming the language — an ad-hoc call needs no pre-created agent. Put what the agent should
accomplish in `systemPrompt`. The response returns immediately with
`{"sessionId": "…", "status": "dialing", …}` — the call itself runs in the background.

The call opens by disclosing it is an AI. That is applied server-side and is not yours to remove
or soften; if the user asks you to hide it, say plainly that Speko always discloses.

Calls are recorded by default. Say so when you confirm the call — "this will be recorded" —
because many places require everyone on a call to know.

If the account owns more than one number, `GET /v1/phone-numbers` (kebab-case — `phone_numbers`
404s) lists which can be used as `"from"`. Leave `from` unset to use the default.

### Proving a request body without ringing anyone

A deliberately bogus `agentId` validates everything except the dial:

```bash
curl -sS -X POST https://api.speko.dev/v1/sessions/phone \
  -H "Authorization: Bearer $SPEKO_API_KEY" -H "Content-Type: application/json" \
  -d '{"to": "+14155550142", "agentId": "00000000-0000-0000-0000-000000000000"}'
# → 404 {"error":"Agent not found","code":"AGENT_NOT_FOUND"}  — body and auth are right, nobody was called
```

Malformed bodies return
`{"error":"Invalid request","code":"VALIDATION_ERROR","issues":[{"path":…,"message":…}]}` —
read `issues`, fix the named field.

## Watching and reading back

Poll `GET /v1/sessions/{sessionId}` until the call has ended, then:

```bash
curl -sS https://api.speko.dev/v1/calls/{sessionId} \
  -H "Authorization: Bearer $SPEKO_API_KEY"
# status, duration_seconds, transcript, recording_status, recording_resource_uri

curl -sS https://api.speko.dev/v1/calls/{sessionId}/report \
  -H "Authorization: Bearer $SPEKO_API_KEY"
# summary, outcome, transcript.entries[{text, source: "agent"|"user", startedAt}]
```

Summarise the outcome first — did the thing the user wanted happen? — then offer the transcript.
Do not paraphrase a commitment the other party did not make; quote them.

## One-shot speech, no phone leg

**Text to speech** — `/v1/synthesize` answers with headerless PCM (`audio/pcm;rate=24000`), not a
playable container, so wrap it:

```bash
echo 'Hello from Speko' | jq -Rs '{text: ., intent: {language: "en"}, sampleRate: 24000}' \
  | curl -sS -X POST https://api.speko.dev/v1/synthesize \
      -H "Authorization: Bearer $SPEKO_API_KEY" -H "Content-Type: application/json" --data-binary @- \
  | ffmpeg -hide_banner -loglevel error -f s16le -ar 24000 -ac 1 -i pipe:0 -y out.wav
```

**Speech to text** — `/v1/transcribe` takes raw audio bytes with the routing intent in an
`x-speko-intent` header and answers with server-sent events, not JSON; the transcript is the
`text` field of the final `data:` line. Non-WAV input silently returns an empty transcript, so
normalise first:

```bash
ffmpeg -hide_banner -loglevel error -i input.ogg -ac 1 -ar 24000 -f wav pipe:1 \
  | curl -sS -X POST https://api.speko.dev/v1/transcribe \
      -H "Authorization: Bearer $SPEKO_API_KEY" -H "Content-Type: audio/wav" \
      -H 'x-speko-intent: {"language": "en"}' --data-binary @- \
  | grep '^data:' | tail -n1 | sed 's/^data: //' | jq -r .text
```

Change `language` inside `intent` to route other languages. Speko picks the voice and model per
request from its live benchmarks; you never choose a provider.

## What not to do here

No bulk, scheduled, or unattended calling — every call is one explicit request the user confirmed.
Buying numbers, changing plans, and deleting things belong in the Speko dashboard
(`https://platform.speko.ai`), not in this skill. For building a full voice application rather
than placing a call, use the `speko-voice-app` skill from this same repo.
