# Publishing Clawmachine Skills to Sage Protocol

This guide walks through publishing the generated skill library to a Sage DAO so any AI agent can discover and use them via MCP.

---

## Prerequisites

- **Node.js** 18+ installed
- A terminal/shell environment
- An Ethereum-compatible wallet (Sage CLI can generate one)

---

## Step 1: Install Sage CLI

```bash
npm i -g @sage-protocol/cli
```

Verify installation:
```bash
sage --version
```

---

## Step 2: Initialize Wallet

Run the setup wizard to create or import a wallet:

```bash
sage wizard
```

This will:
- Create a new wallet on **Base Sepolia** testnet (or import an existing one)
- Store your wallet credentials locally
- Set up your Sage identity

---

## Step 3: Get Test Tokens

Request test SXXX tokens from the faucet:

```bash
sage wallet faucet
```

You need tokens to create a DAO and publish prompts to IPFS.

---

## Step 4: Initialize Prompts Workspace

```bash
sage prompts init
```

This creates a `prompts/` directory in your current working directory. This is where your skill files go.

---

## Step 5: Copy Generated Skills

Copy all 40 generated skill files from `skills/generated/` into the `prompts/` directory:

```bash
cp skills/generated/*.md prompts/
```

Verify the files are in place:
```bash
ls prompts/ | wc -l
# Should show 40
```

---

## Step 6: Create DAO and Publish

Use the quickstart command to create a DAO and publish all skills in one step:

```bash
sage library quickstart --name clawmachine-skills
```

This will:
1. Create a new Sage DAO named `clawmachine-skills`
2. Upload all `.md` files in `prompts/` to IPFS
3. Register each skill as a prompt in the DAO
4. Output the DAO address and IPFS hashes

**Expected output:**
```
Created DAO: 0x1234...abcd
Published 40 prompts to IPFS
DAO URL: https://sage.build/dao/clawmachine-skills
```

---

## Step 7: Start MCP Server

Launch the Sage MCP server so agents can discover your skills:

```bash
sage mcp start
```

This starts a local MCP server that exposes:
- `search_prompts(query)` - Search for skills by keyword
- `get_prompt(name)` - Retrieve a specific skill by name
- `list_prompts()` - List all published skills

---

## Step 8: Verify Publication

### View all published prompts
```bash
sage project view-prompts
```

### Test search
```bash
sage prompts search "clawmachine"
```

### Test retrieval
```bash
sage prompts get "clawmachine-orchestrator"
```

---

## Agent Discovery Flow

Once published, any AI agent with Sage MCP access can:

1. **Search:** `search_prompts("clawmachine game")` -> finds the orchestrator
2. **Load orchestrator:** `get_prompt("clawmachine-orchestrator")` -> master skill
3. **Load sub-skills:** Orchestrator instructs agent to load specific skills based on the game request
4. **Build game:** Agent follows the loaded skills to build and submit

### Example MCP Integration

```json
{
  "mcpServers": {
    "sage": {
      "command": "sage",
      "args": ["mcp", "start"]
    }
  }
}
```

---

## Updating Skills

To update a published skill:

1. Edit the `.md` file in `prompts/`
2. Re-publish:
   ```bash
   sage prompts publish --name clawmachine-genre-action-2d
   ```
3. The DAO automatically versions the update

To publish all updates at once:
```bash
sage prompts publish --all
```

---

## Skill File Format Reference

Every skill must use this Sage Protocol frontmatter format:

```yaml
---
title: Clawmachine - [Skill Name]
description: [Brief description of what this skill teaches]
---

[Markdown body with instructions, code examples, checklists]
```

The `title` and `description` fields are indexed by the MCP search function, so use descriptive keywords.

---

## Troubleshooting

### "Insufficient balance" error
Run `sage wallet faucet` again. DAO creation and IPFS uploads require tokens.

### "DAO already exists" error
Use a different name or publish to the existing DAO:
```bash
sage prompts publish --dao clawmachine-skills --all
```

### MCP server won't start
Check that the Sage CLI is up to date:
```bash
npm update -g @sage-protocol/cli
```

### Skills not appearing in search
IPFS propagation can take a few minutes. Wait and retry:
```bash
sage prompts search "clawmachine"
```
