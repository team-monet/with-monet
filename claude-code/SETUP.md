# Dogfood kit — Claude Code

Turnkey setup for running real sessions against the substrate.

## 1. Install the server (once per machine)

```bash
npm i -g @team-monet/monet      # provides the `monet` command
```

Needs `node` ≥ 22 and network access (the first run downloads the MiniLM model once). Zero-install alternative: use `npx -y @team-monet/monet start` as the MCP command instead of `monet start`.

## 2. Onboard a test repo

Two ways — both end in the same place.

### A — Agent-first (the product flow)
Open the repo in Claude Code and paste:

> Set up Monet for this project: read https://raw.githubusercontent.com/team-monet/with-monet/main/bootstrap/install.md and follow it, checking with me at each decision point.

The agent wires the MCP server, installs Stig, offers to ingest your docs, then offers to start.

### B — Manual (fastest for your own testing)
Installs the **full team globally** (user scope — every project), non-destructively. Needs `jq` and the `claude` CLI. Set `WM` to your `with-monet` checkout:

```bash
WM=/path/to/with-monet                 # your with-monet checkout
AG="$HOME/.claude/agents"; mkdir -p "$AG"

# 1. MCP server at USER scope — available in every project
claude mcp add --scope user monet -- monet start   # or merge {command:"monet",args:["start"]} into ~/.claude.json

# 2. Worker subagents (user scope). name/description/model come from roster.json — the `description`
#    is the trigger text Claude Code matches to delegate, so don't water it down.
for f in explorer researcher analyst developer tester reviewer auditor security reliability aria; do
  out="$AG/$f.md"
  [ -e "$out" ] && [ ! -e "$out.bak" ] && cp "$out" "$out.bak"    # back up the user's ORIGINAL once; don't clobber it on reruns
  desc=$(jq -c --arg n "$f" '.agents[]|select(.name==$n).description' "$WM/roster.json")  # JSON-quoted ⇒ YAML-safe (descriptions contain ':')
  model=$(jq -r --arg n "$f" '.agents[]|select(.name==$n).model' "$WM/roster.json")
  # `<!-- with-monet:agent -->` marks these as Monet's installed workers, so a later memory-consolidation
  # pass excludes them by marker (not by filename) and never captures/retires the team.
  { printf -- '---\nname: %s\ndescription: %s\nmodel: %s\n---\n<!-- with-monet:agent -->\n\n' "$f" "$desc" "$model"; cat "$WM/agents/$f.md"; } > "$out"
done

# 3. Stig as the lead, in GLOBAL memory (~/.claude/CLAUDE.md) — idempotent, original backed up once
CM="$HOME/.claude/CLAUDE.md"; mkdir -p "$(dirname "$CM")"
[ -e "$CM" ] && [ ! -e "$CM.bak" ] && cp "$CM" "$CM.bak"          # preserve the original, never overwrite it on reruns
touch "$CM"
sed -i.tmp '/<!-- BEGIN with-monet:stig -->/,/<!-- END with-monet:stig -->/d' "$CM" && rm -f "$CM.tmp"
{ printf '\n<!-- BEGIN with-monet:stig -->\n'; cat "$WM/agents/stig.md"; printf '\n<!-- END with-monet:stig -->\n'; } >> "$CM"
```

Storage is one global brain at `~/.monet`, isolated per project by circle. Per-repo instead? Write `./.mcp.json` (project scope) and set `MONET_STORAGE_DIR=<repo>/.monet`.

## 3. Run the session

1. **Reload/restart Claude Code** so it picks up the user-scope `monet` server and `~/.claude/agents/` (monet connects; watch stderr for `semantic embeddings ready`).
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
monet status            # global store (~/.monet): Concepts / Observations / Workstreams
rm -rf ~/.monet         # wipe the global store   (per-repo: MONET_STORAGE_DIR=<repo>/.monet monet status)
```
