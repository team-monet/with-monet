# Dogfood kit — Claude Code

Turnkey setup for running real sessions against the substrate.

## 1. Install the server (once per machine)

```bash
npm i -g @team-monet/local      # provides the `monet-local` command
```

Needs `node` ≥ 22 and network access (the first run downloads the MiniLM model once). Zero-install alternative: use `npx -y @team-monet/local start` as the MCP command instead of `monet-local start`.

## 2. Onboard a test repo

Two ways — both end in the same place.

### A — Agent-first (the product flow)
Open the repo in Claude Code and paste:

> Set up Monet for this project: read https://raw.githubusercontent.com/team-monet/with-monet/main/bootstrap/install.md and follow it, checking with me at each decision point.

The agent wires the MCP server, installs Stig, offers to ingest your docs, then offers to start.

### B — Manual (fastest for your own testing)
From the test repo root (set `WM` to wherever you cloned `with-monet`). Always install the **full team** — Stig is the lead, the workers are its actuators:

```bash
WM=/path/to/with-monet

# 1. MCP server
cp "$WM/claude-code/mcp.json" ./.mcp.json     # or merge into existing .mcp.json

# 2. Worker subagent team (always — orchestrating them is the whole point of Stig)
mkdir -p .claude/agents
for f in explorer researcher analyst developer tester reviewer security reliability aria; do
  { printf '%s\n' '---' "name: $f" "description: $f actuator — Stig injects its context" '---' ''; cat "$WM/agents/$f.md"; } > ".claude/agents/$f.md"
done

# 3. Stig as the lead (main agent delegates to the team above)
cat "$WM/agents/stig.md" >> ./CLAUDE.md       # append; don't clobber existing notes
```

## 3. Run the session

1. **Reload/restart Claude Code** in the repo so it picks up `.mcp.json` (monet-local connects; watch stderr for `semantic embeddings ready`).
2. **Sanity check:** ask the agent to call `agent_context` — a fresh store returns empty workstreams/topConcepts.
3. **Seed (optional):** ask it to ingest `CLAUDE.md` / `docs/` — `memory_store` (dedup is automatic).
4. **Work normally on real code.** Watch the substrate guarantees in action:
   - **prewarm:** `agent_context` at session start restores state with no query (#242)
   - **dedup:** re-storing similar facts resolves into one concept, never duplicates (#239)
   - **correction:** store with `kind="correction"` → concept goes `disputed`; mediate with `memory_resolve` (#240)
   - **survival:** `memory_checkpoint` a workstream at the end → restored next session (#241)
   - search returns **cards** (no content) → it must `memory_fetch` (#232)

## Inspect / reset

```bash
MONET_STORAGE_DIR=<repo>/.monet monet-local status   # Concepts / Observations / Workstreams
rm -rf <repo>/.monet                                 # wipe the store
```
