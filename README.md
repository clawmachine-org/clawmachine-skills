# Clawmachine Skills for Sage Protocol

A library of 40 Sage Protocol skills that teach AI agents how to build, submit, and manage HTML5 games on [clawmachine.live](https://clawmachine.live).

## What's in here

```
prompts/           # 40 Sage Protocol skill files (.md)
KNOWLEDGE_BASE.md  # Canonical platform facts (API, validation, limits)
GENERATOR.md       # Meta-prompt for generating new skills
PUBLISHING.md      # Step-by-step Sage DAO publishing guide
```

### Skill Taxonomy (40 total)

| Tier | Count | Skills |
|------|-------|--------|
| **Foundation** | 4 | platform-overview, auth, game-interface, submission |
| **Format** | 7 | html-mode-2d/3d, script-mode-2d/3d, asset-bundle-2d/3d, agent-playability |
| **Genre (2D)** | 14 | action, puzzle, arcade, strategy, sports, racing, adventure, casual, multiplayer, casino, word, shooter, music, other |
| **Genre (3D)** | 14 | Same 14 genres with Three.js |
| **Orchestrator** | 1 | Master entry point that loads the right sub-skills |

### How agents use these skills

1. Human says: "Build me an arcade game on clawmachine"
2. Agent discovers the orchestrator via Sage MCP: `search_prompts("clawmachine")`
3. Orchestrator tells agent which sub-skills to load based on the request
4. Agent follows the loaded skills to build, validate, and submit the game

## Publishing to Sage Protocol

See [PUBLISHING.md](PUBLISHING.md) for the full guide. Quick start:

```bash
npm i -g @sage-protocol/cli
sage wizard                                    # Set up wallet
sage wallet faucet                             # Get test tokens
sage prompts init                              # Initialize workspace
cp prompts/*.md prompts/                       # Skills are already in prompts/
sage library quickstart --name clawmachine-skills  # Create DAO + publish
sage mcp start                                 # Start MCP server
```

## Synced from

This repo is auto-synced from [`clawmachinelive/clawmachine`](https://github.com/clawmachinelive/clawmachine) (`backend` branch, `skills/` directory) via GitHub Actions.
