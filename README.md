# RunApps skill for AI agents

One [Agent Skill](https://agentskills.io) that teaches an AI coding agent (Claude Code, Codex, Cursor, Gemini CLI, … anything that follows the format) how to build and publish AI web apps on [RunApps](https://www.runapps.ai) through its MCP tools, and how to write the app code with the RunApps SDK.

## Install

```bash
npx skills add runapps/skills --global --yes -a claude-code
```

Set `-a` to your agent (`npx skills add --help` lists them).

Connecting an agent to the platform (device-code authorization, MCP URL, scopes) is described at https://www.runapps.ai/agent/setup.md and in the skill itself; the agent can read either and do it by itself. The one line to give your agent:

> Read https://www.runapps.ai/agent/setup.md and connect to RunApps.

## Layout

```
runapps/
├── SKILL.md                 entry point: connecting, the platform model, every MCP tool, the workflow
│                            from create_app to a reviewed public app, safety rules
└── references/
    ├── sdk.md               the SDK and platform docs (the README of github.com/runapps/sdk-js):
    │                        sign-in, chat, images, audio, video, files, sync, private prompts, HTTP API
    └── tools.md             MCP tool parameters and result shapes
```

The agent loads `SKILL.md` when a task concerns RunApps and reads the reference files that task needs. `references/sdk.md` is a copy of the SDK README; update it from there when the SDK changes.

## License

MIT.
