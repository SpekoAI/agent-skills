# Speko Agent Skills

[Agent Skills](https://agentskills.io) that teach coding agents to build with and act through
[Speko](https://speko.ai) — hosted voice agents that place and answer real phone calls, with
speech-to-text, LLM routing, and text-to-speech behind one API.

| Skill | What it does |
|---|---|
| [`speko-voice-app`](skills/speko-voice-app/SKILL.md) | How to build a voice application on Speko: keys, the dial→poll→report loop, webhooks, speech endpoints, the traps. |
| [`speko-phone-call`](skills/speko-phone-call/SKILL.md) | How to place one real phone call safely and read back what was said — confirmation rules, dial, transcript, recording. |

## Install

```bash
npx skills add SpekoAI/agent-skills
```

pick your agents interactively, or target them directly:

```bash
npx skills add SpekoAI/agent-skills -a claude-code -a cursor -a codex
npx skills add SpekoAI/agent-skills -g -a universal   # global install to ~/.agents/skills
```

Works with any agent that reads the Agent Skills format — Claude Code, Cursor, Codex, Copilot,
Gemini CLI, Cline, Zed, Warp, Amp, Replit, OpenClaw, and others.

### Shelley / exe.dev

[Shelley](https://exe.dev/docs/shelley/intro.md) discovers skills natively from
`~/.config/agents/skills`, `~/.config/shelley`, and project `.skills/` directories — which the
`skills` CLI does not currently target, so install by copying:

```bash
# global, on an exe.dev VM (or anywhere Shelley runs):
git clone --depth 1 https://github.com/SpekoAI/agent-skills /tmp/speko-skills \
  && mkdir -p ~/.config/agents/skills \
  && cp -r /tmp/speko-skills/skills/* ~/.config/agents/skills/

# or per-project: vendor a skill into your repo and every fresh VM gets it,
# including repos launched via https://exe.dev/new?repo=<your-repo>
mkdir -p .skills && cp -r /tmp/speko-skills/skills/speko-phone-call .skills/
```

## Keys

Both skills authenticate with a Speko platform API key: mint one at
[platform.speko.ai/agents/keys](https://platform.speko.ai/agents/keys) and export it as
`SPEKO_API_KEY`. Keys are environment-scoped; `GET https://api.speko.dev/v1/organization` is the
quickest liveness check.

## License

MIT — see [LICENSE](LICENSE).
