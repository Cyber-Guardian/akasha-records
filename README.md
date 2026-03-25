# Akasha Records

Knowledge vault for Akasha / FileScience. Contains project identity, topic notes, research, briefs, decisions, and reference material.

## Structure

```
durable/           Project identity (always-true constraints)
topics/            Warm topic notes (living documents per domain)
thoughts/shared/   Time-stamped knowledge artifacts
  briefs/          Shaped ideas and exploration output
  research/        Deep research and analysis
  decisions/       Architectural and process decisions
  reference/       Reference material and guides
  handoffs/        Session handoff documents
  series/          Recurring content series
```

## Relationship to filescience

This repo is the **knowledge layer** extracted from the [filescience](https://github.com/Cyber-Guardian/filescience) monorepo. The filescience repo retains:

- All code, infrastructure, and tooling
- Implementation plans (`memory-bank/thoughts/shared/plans/`) -- moving here in Wave 2
- Assembly-line work plans (`.work/plans/`)
- Procedural knowledge (`.claude/` skills, rules, agents)

## How Scout indexes this

[Scout](https://github.com/Cyber-Guardian/filescience/tree/main/tools/scout) is an MCP server that provides semantic search, graph traversal, and write tools over this vault.

Configure scout to point at this repo:

```json
{
  "mcpServers": {
    "scout": {
      "command": "uv",
      "args": ["run", "--project", "tools/scout", "scout", "--vault-path", "../akasha-records"]
    }
  }
}
```

Scout tools:
- **Read:** `search`, `vector_search`, `hybrid_search`, `get`, `list_documents`
- **Write:** `put`, `delete` (local working tree)
- **Publish:** `propose` (PR), `commit_direct` (push to main)
- **Graph:** `graph_neighbors`, `graph_path`, `graph_hubs`, `graph_stats`, `graph_rebuild`

## Setup

Clone adjacent to the filescience repo:

```bash
cd /path/to/filescience/..
gh repo clone Cyber-Guardian/akasha-records
```

Scout expects `../akasha-records` relative to the filescience repo root by default.
