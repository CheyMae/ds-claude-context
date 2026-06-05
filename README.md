# ds-claude-context

Persistent context repo for Claude sessions working on design systems in Figma.

## Why this repo exists

Working with Claude across multiple sessions requires re-establishing context each time: which design system, which Figma file, which variable architecture, which working conventions. Manual copy-paste is fragile and grows stale.

This repo solves that. A single bootstrap URL gives Claude everything it needs to hit the ground running.

## How it works

1. In a new Claude chat, paste one of the bootstrap prompts (see [Bootstrap prompts](#bootstrap-prompts) below)
2. Claude fetches `CONTEXT.md` from this repo
3. `CONTEXT.md` instructs Claude to load relevant skill files and a scenario-specific kickoff prompt
4. Claude proceeds with full context — no copy-paste needed

When the design system changes (new tokens, new components, new conventions), update the relevant skill files here. The next Claude chat picks up the changes automatically.

## Repo structure

```
ds-claude-context/
├── CONTEXT.md                        ← Claude reads this first
├── README.md                         ← this file
├── skills/
│   └── clean-spiel-collection.md    ← audit + restructure string variable collections
├── kickoff-prompts/
│   ├── default.md                   ← catch-all when no scenario is named
│   └── spiel-cleanup.md             ← clean up a string variable collection
└── docs/
    └── sessions/                    ← chronological session notes (optional)
```

## Bootstrap prompts

Paste one of these at the start of a new Claude chat. Replace `YOUR_GITHUB_USERNAME` with your username.

### General (Claude picks scenario based on conversation)

```
Load my design system context from:
https://raw.githubusercontent.com/YOUR_GITHUB_USERNAME/ds-claude-context/main/CONTEXT.md

I'll tell you what we're working on next.
```

### Specific scenario

```
Load my design system context from:
https://raw.githubusercontent.com/YOUR_GITHUB_USERNAME/ds-claude-context/main/CONTEXT.md

Scenario: spiel-cleanup
```

Available scenarios: `spiel-cleanup`, `default`

### Private repo variant

If this repo is private, generate a GitHub Personal Access Token (PAT) with `repo` scope and paste it inline:

```
Load my design system context from:
https://raw.githubusercontent.com/YOUR_GITHUB_USERNAME/ds-claude-context/main/CONTEXT.md?token=YOUR_PAT

Scenario: spiel-cleanup
```

> Note: PAT-in-URL will appear in chat history. For higher security, use Claude Projects with manual file uploads instead.

## Skills in this repo

| Skill | File | What it does |
|---|---|---|
| clean-spiel-collection | `skills/clean-spiel-collection.md` | Full pipeline for auditing, deduplicating, renaming, and binding a Figma string variable collection |

## Adding a new skill

1. Create `skills/your-skill-name.md` following the structure of an existing skill
2. Add a row to the skill table in `CONTEXT.md` and in this README
3. Create a matching `kickoff-prompts/your-scenario.md` if it warrants its own scenario
4. Add the scenario to the kickoff prompt table in `CONTEXT.md`

## Updating context

When the design system changes:

1. Update the relevant `skills/*.md` file(s)
2. If Figma file key or variable architecture changed, update `CONTEXT.md`
3. Add a session note in `docs/sessions/YYYY-MM-DD-topic.md` for significant changes
