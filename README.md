# pi-skill-evolution

Hermes Agent-style automated skill creation and evolution for Pi Agent.

Pi Agent already has a great skills system and extension API, but lacks the
closed-loop "auto-create, auto-modify, evolve skills" capability that Hermes
Agent achieves through `background_review` and `skill_manage`. This repo fills
that gap on top of Pi's event-driven architecture.

## What it does

| Capability | Description |
|------------|-------------|
| Auto-create skills | After a complex task, the agent saves the workflow as a skill via `skill_manage` |
| Auto-patch skills | When an existing skill can be improved, uses `patch` for minimal, safe edits |
| Background review loop | After each session, the agent self-audits and decides whether to save/update skills |
| Progressive disclosure | Only skill names + descriptions sit in the system prompt; full content loads on demand |

## Files

```
pi-skill-evolution/
├── README.md                      ← this file (EN)
├── README_ZH.md                   ← 中文说明
├── extensions/
│   └── skill-evolution.ts         ← Pi extension: skill_manage tool + review loop
├── skill-authoring/
│   └── SKILL.md                   ← Skill authoring playbook for the agent
└── LICENSE
```

## Installation

### Direct copy (easiest)

```bash
# Extension
cp extensions/skill-evolution.ts ~/.pi/agent/extensions/

# Authoring skill (recommended)
mkdir -p ~/.pi/agent/skills/skill-authoring
cp skill-authoring/SKILL.md ~/.pi/agent/skills/skill-authoring/
```

Pi auto-discovers both — no settings changes needed.

### As a Pi package

```bash
pi install git:github.com/<your-org>/pi-skill-evolution
```

Requires a `package.json` with `pi.extensions` and `pi.skills` entries (see [Dev section](#development) below).

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PI_SKILL_EVOLUTION_DIR` | `~/.pi/agent/skills/` | Directory where skills are stored |

## How it works

### `skill_manage` tool

The extension registers a `skill_manage` tool callable by the LLM:

| Operation | Required params | What it does |
|-----------|----------------|--------------|
| `create` | `skillName`, `description`, `content?` | Creates a new skill |
| `edit` | `skillName`, `description`, `content` | Full rewrite of an existing skill |
| `patch` | `skillName`, `find`, `replace?` | Exact find-and-replace inside SKILL.md (preferred — safest) |
| `delete` | `skillName` | Removes a skill |
| `list` | — | Lists all available skills |
| `inspect` | `skillName`, `content?` | Reads current skill content |

Skill names are validated per the Agent Skills spec: 1–64 chars, lowercase letters,
digits, hyphens only, no leading/trailing/consecutive hyphens.

### System prompt injection

On `before_agent_start`, appends:

> After completing a complex task (5+ tool calls), fixing a tricky error, discovering
> a non-trivial workflow, or recovering from an unexpected failure, save the approach
> as a skill using the skill_manage tool so you can reuse it next time.

### Background review loop

On `agent_settled` (every 3rd settled event to prevent spam), sends a follow-up
message that asks the agent to review the completed session and create or patch
skills with `skill_manage`:

> Review this session. Did you just complete a complex task, fix a tricky error,
> discover a new workflow, or recover from an unexpected failure? If yes, use
> skill_manage to create or patch a skill. Be proactive — most sessions produce
> at least one skill update.

### Skill authoring playbook

`skill-authoring` is a model-invoked skill that auto-loads when the agent uses
`skill_manage`. Covers:

- Name rules, description best practices, leading words
- Information hierarchy (step / reference / external reference)
- Progressive disclosure and when to split skills
- Failure modes: premature completion, duplication, sediment, sprawl, no-op
- `skill_manage` operation cheat sheet with preference order: patch loaded → patch existing → create

## Hermes Agent vs pi-skill-evolution

| Dimension | Hermes Agent | pi-skill-evolution |
|-----------|-------------|-------------------|
| Review trigger | Daemon thread (`spawn_background_review`) | `agent_settled` event + `sendUserMessage` followUp |
| Skill management | Python `skill_manager_tool.py` | TypeScript `pi.registerTool()` |
| Prompt injection | `prompt_builder.py` hardcoded layer | `before_agent_start` event chain |
| Skill format | agentskills.io `SKILL.md` | agentskills.io `SKILL.md` |
| Review agent | Separate sub-agent instance | Reuses main agent loop |

Pi's implementation is cleaner — no extra agent instances needed, fully leverages
Pi's existing event-driven architecture and session system.

## Development

### Test the extension standalone

```bash
pi --extension /path/to/extensions/skill-evolution.ts

pi --extension /path/to/extensions/skill-evolution.ts \
   -p "List all skills with skill_manage"
```

### Package as a Pi npm/git package

Create `package.json` at the repo root:

```json
{
  "name": "pi-skill-evolution",
  "version": "0.1.0",
  "type": "module",
  "pi": {
    "extensions": ["./extensions/skill-evolution.ts"],
    "skills": ["./skill-authoring"]
  },
  "dependencies": {}
}
```

Then install with `pi install .` or publish to npm / a git repo.

## Compatibility

Requires Pi Agent ≥ 0.80 (needs `pi.registerTool()` dynamic registration and the
`agent_settled` event).