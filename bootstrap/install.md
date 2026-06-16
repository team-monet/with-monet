# Monet — agent-first install playbook

**You are the user's coding agent.** The user pasted a one-line request to set up Monet. Your job is to install and configure it *for* them — running the commands yourself, conversing at each decision point, and diagnosing/fixing failures as you go. **Do not ask the user to clone a repo or run shell scripts themselves.** Adapt to their host and preferences.

Work through the phases in order. After each, tell the user what happened in one line. If a step fails, show the error, explain the likely cause, propose a fix, and retry — never leave a half-configured state.

**This playbook is self-contained — no clone or pre-install needed.** You install the substrate yourself (Phase 2), and you read any prompt file you need (`roster.json`, `agents/<name>.md`) from one of:
- a local `with-monet` checkout if one exists (prefer it), else
- this repo's raw URLs, base `https://raw.githubusercontent.com/team-monet/with-monet/main/` (e.g. `…/agents/stig.md`).

(Raw fetch needs the repo to be **public**; if it's private and you can't fetch, that's the one time to ask the user to clone `with-monet` and point you at the local path.)

---

## Updating an existing install (team files only)

Already set up and just want the latest team (Stig + workers)? You don't need to re-run onboarding — re-run **Phase 4 only** against the latest sources:
1. Re-read `roster.json`, `agents/*.md`, and `agents/stig.md` from the raw-URL base (or your local `with-monet` checkout).
2. Re-apply Phase 4's write step **to the same scope the team was installed in** — detect that from where the existing `<!-- BEGIN with-monet:stig -->` / `<!-- with-monet:agent -->` markers already live in your host's lead-persona target and worker locations (Phase 1 maps these per host). (Claude Code example: user scope `~/.claude/CLAUDE.md` + `~/.claude/agents/<name>.md`; per-repo `./CLAUDE.md` + `./.claude/agents/<name>.md`.) Rewrite the Stig block (between the `<!-- BEGIN with-monet:stig -->` / `<!-- END with-monet:stig -->` markers) and each agent file **in that location** — never silently switch a per-repo install to global. **Reconcile, don't clobber:** preserve your local edits, apply the new changes, guard the invariants, and keep `.bak` (see Phase 4).

Skip Phases 1–3 (substrate + MCP already configured) and 5–7 — except: if your host only loads agents/MCP at startup (Claude Code, Cursor, and most IDE-based hosts do — see the per-host reload note in Phase 1), **reload or restart it afterward** so the updated Stig + workers take effect; a running session keeps the old prompts until you do. Phase 4 is idempotent, so this is safe to repeat.

---

## Phase 1 — Orient

You install Monet **globally for this user** (every project), not just the current repo — so the team and memory work everywhere without re-installing per project. (If the user prefers, you can scope it to just this repo instead — confirm in step 2.)

1. Identify your host (Claude Code, Cursor, Continue, Aider, …) and note where it reads, **at user scope**, (a) **MCP server config** and (b) **agent/subagent prompts**.
   - Claude Code → MCP: user scope via `claude mcp add --scope user …` (or `~/.claude.json`); subagents: `~/.claude/agents/<name>.md`; lead persona: `~/.claude/CLAUDE.md` (global memory). *(Project scope, if requested: `./.mcp.json`, `./.claude/agents/<name>.md`, `./CLAUDE.md`.)* Reload: Claude Code picks up user-scope servers and `~/.claude/agents/` on startup — reload or restart the session after any install or update.
   - Cursor → MCP: `~/.cursor/mcp.json`; agent rules: `.cursor/rules/*.mdc` — **repo-scoped, not global** (Cursor's user-scope "User Rules" live in Settings → Rules, a UI you can't write as a file). For an all-projects install, ask the user to paste the lead persona into Settings → Rules; the file-based `.cursor/rules` path is per-repo only. (`.cursor/rules` are always-on instructions — fine for the lead persona, but **not** delegatable workers; see Tier B.) Reload: Cursor loads MCP + rules on startup — restart after changes.
   - Aider (or any host with **no MCP support**) → Monet can't register as an MCP server, and Stig + the workers depend on Monet's MCP tools (`agent_context`, `memory_store`, `memory_checkpoint`) — so the harness **can't run as designed** here. **Stop**: tell the user Monet needs an MCP-capable host, and don't install a lead/worker prompt that would call tools the host can't provide. (Monet's whole value is the MCP memory substrate.)
   - OpenCode / Cline / Windsurf → MCP + subagents supported; config locations vary by host and version — confirm the exact paths in the host's docs before writing. Reload: consult host docs for whether a restart is needed after config changes.
   - Unknown host → ask the user where its user-level MCP config and custom-agent-prompt location live, and whether a restart is needed after changes.
2. Confirm: *"You're on **<host>**. I'll install Monet **globally** so it works across all your projects — or scoped to just this repo if you'd rather. Anything special about your setup?"*

## Phase 2 — Get Monet

Goal: the `monet` MCP server (the state-centric memory substrate) available on this machine.

- Install from npm (published):
  ```
  npm i -g @team-monet/monet      # provides the `monet` command
  ```
  Zero-install alternative: `npx -y @team-monet/monet start`. Requires `node` ≥ 22 and network access — the first run downloads the MiniLM embedding model once.
- Dev / unpublished fallback: clone `team-monet/monet`, then `pnpm install && pnpm build`, and use `node <abs>/dist/index.js`.

## Phase 3 — Configure the monet MCP server (user scope)

Register `monet` in your host's MCP config at **user scope** so it's available in every project (canonical template: `with-monet/mcp/monet.json`; ready-to-merge per-host snippets in `with-monet/examples/<host>/mcp.json`, e.g. `examples/cursor/mcp.json`). The entry to merge is:
```jsonc
{ "mcpServers": { "monet": { "command": "monet", "args": ["start"] } } }
```
*(Dev/unpublished fallback: use `"command": "node", "args": ["<abs>/dist/index.js"]` in place of `"command": "monet"`.)*

Host examples:
- **Claude Code:** `claude mcp add --scope user monet -- monet start`, or merge the entry above into `~/.claude.json` under `mcpServers`. *(Per-repo install instead: write `./.mcp.json`.)*
- **Cursor:** merge the entry into `~/.cursor/mcp.json` under `mcpServers`. *(Per-repo: `.cursor/mcp.json` in the repo root, if Cursor supports it — confirm in Cursor's docs.)*
- **Aider / any no-MCP host:** MCP isn't supported, so Monet is unavailable — **stop here** (see the Phase 1 no-MCP note) and don't continue to Phase 4. Installing Stig without Monet hands the lead instructions to call tools that don't exist.
- **Other host:** merge the entry into your host's user-level MCP config file. Consult your host's docs for the exact path and format.

**Storage — one global brain, organized per project.** By default the store lives at `~/.monet` (shared across all the user's projects) and Monet organizes each project into its own *circle* automatically — no config required. If the user prefers a hard filesystem split per repo, offer it: set `env.MONET_STORAGE_DIR` to a per-project path (e.g. `<repo>/.monet`) in the MCP config block you write for them. Default to the shared global store unless they ask.

Merge into the host's **user-level** MCP config **without clobbering existing servers** — read it, add `monet`, write it back, and back up the file first. Verify it comes up: `monet start` logs `semantic embeddings ready (MiniLM, 384-dim)` then `Monet started`, and also prints the active project circle (`Circle: <name>`), confirming per-project organization. The server exposes 14 tools: `agent_context`, `memory_store`, `memory_search`, `memory_overview`, `memory_gather`, `memory_fetch`, `memory_synthesize`, `memory_checkpoint`, `memory_flag_contradiction`, `memory_resolve`, `memory_list`, `memory_reassign_circle`, `memory_detach`, `memory_circle_manage`. Recall is store-wide, with each result card carrying its home circle, so reorganizing circles never breaks findability. `memory_detach` splits observations out of a concept for wrong-merge repair — or, given `destConceptId`, folds them into another concept; detaching all observations with `destConceptId` consolidates the source away, carrying its identity as an alias on the keeper so references keep resolving. `memory_fetch` returns observation entries as `{id, content}`, pages with `observationsOffset`, and includes `lastConfirmedAt` (Unix ms of last evidence-based confirmation) as the staleness signal. `memory_store` accepts `resolution: "forceNew"`/`attachTo` for bulk import and directed attach. `memory_overview` surfaces `possibleDuplicates`. `memory_resolve` handles two verdict families: contradiction resolution (retrying an already-closed contradiction returns `alreadyClosed: true` with zero mutations) and duplicate-pair dismissal (pass `conceptAId` + `conceptBId` to assert two concepts are not duplicates, clearing the pair from `possibleDuplicates`; idempotent when no live edge exists). `memory_circle_manage` handles circle lifecycle — rename/merge/archive with stable aliases (the interactive curation ritual lives in [`curate-memory.md`](curate-memory.md)); `memory_reassign_circle` accepts `ids` batching and `resolution: "auto"|"forceNew"`. The consolidation flow uses `memory_list` and `memory_reassign_circle` (see [`consolidate-memory.md`](consolidate-memory.md)). The server also provides an in-band session lifecycle with zero host configuration: on the first successful tool response, it appends a delimited block (`=== MONET SESSION CONTEXT (auto-prewarm) ===`) as an additional content item carrying active workstreams, top concepts, and a curation advisory — suppressed when the first call is `agent_context` (no double-inject), and opt-out server-side via `MONET_NO_AUTOPREWARM`; after 10 mutating calls (then every 20) it appends a checkpoint nudge, silenced by a checkpoint that saves a workstream; and the server's `instructions` field describes the memory loop so an agent without the Stig persona gets the loop on first use.

## Phase 4 — Install the agent team (user scope)

**Always install the full team.** Stig is a context engine whose entire purpose is to orchestrate the workers — never install Stig alone. Stig is the **lead** (the one the user talks to, the only one that delegates, the only one that touches Monet); the workers are its **subagent actuators**.

**Tier A — Lead persona (ask first, highest-impact write).** This is the install's highest-impact write, so it gets its own decision point. Ask: *"Install Stig as the lead persona in your [host's lead-persona location]? This changes how every session on this machine starts."* A general "go ahead with the install" doesn't cover this — wait for an explicit yes (your host's permission system will likely insist on the same). On a no, offer to scope Stig to the current repo's equivalent per-repo location instead (the per-repo option from Phase 1).

Write the body of `agents/stig.md` (wrapped in the `<!-- BEGIN with-monet:stig -->` / `<!-- END with-monet:stig -->` markers) into your host's lead-persona target so the **main agent** acts as Stig and can delegate to workers. A key constraint that must survive regardless of host: the lead is the only agent that delegates and the only one that touches Monet — sub-contexts cannot spawn further sub-contexts, so the lead must be the main agent.

Host examples for lead-persona target:
- Claude Code → `~/.claude/CLAUDE.md` (global); or `./CLAUDE.md` (per-repo scope).
- Cursor → `.cursor/rules/stig.mdc` with `alwaysApply: true` (repo-scoped; for a global install use Settings → Rules — see Phase 1).
- No-MCP hosts (e.g. Aider) → not reached: they stop at Phase 1/3 (Monet needs MCP).
- Unknown host → ask the user where its lead-persona / always-active system prompt lives.

Wrap the body in idempotent markers so re-running doesn't duplicate it. If the host's lead-persona file carries required frontmatter (e.g. a Cursor `.mdc` rule's `alwaysApply: true`), the markers and body go **below** that frontmatter — a leading HTML comment would break the rule's activation metadata:
```
<!-- BEGIN with-monet:stig -->
…agents/stig.md body…
<!-- END with-monet:stig -->
```
If the markers already exist, replace the block in place; never append a second copy, and never clobber the user's other content in that file (back it up first).

**Tier B — Workers (named sub-contexts, where the host supports them).** Only install workers where the host has a **real named-subagent primitive** that gives each worker a *fresh, isolated context* the lead can delegate to. Write one worker prompt per worker into your host's agent-prompt location — `explorer, researcher, analyst, developer, tester, reviewer, auditor, security, reliability, aria`. Hosts whose "agents" are really always-on instruction *rules* (e.g. Cursor's `.cursor/rules`) do **not** qualify — rules bleed every worker persona into the main context, breaking the "workers execute separately, never touch Monet" invariant. For any host without a true subagent primitive, **skip this tier**: the lead hands off to workers manually in-conversation.

Each worker file is any required host frontmatter + the body from `agents/<name>.md`. Pull the frontmatter fields from that worker's entry in `roster.json`:
- `name` → frontmatter `name`
- `description` → frontmatter `description` — **use it verbatim; do not replace it with a bare role label.** This is your host's dispatch trigger: your host's dispatch mechanism maps the `description` onto its trigger so the right worker is invoked automatically (Claude Code: auto-dispatch on the frontmatter description; other hosts: consult host docs for how task-to-agent routing is configured). The descriptions contain colons, so **quote them** in YAML (`description: "…"`) or the frontmatter won't parse.
- `model` → frontmatter `model` (`haiku`/`sonnet`/`opus`, or omit to inherit the session model). These are sensible defaults — offer to retune (e.g. cheaper models for read-only workers, stronger for `reviewer`/`security`).

Add the `<!-- with-monet:agent -->` marker right after any leading frontmatter — it lets a later memory-consolidation pass tell Monet's installed workers apart from the user's *own* custom sub-contexts, so it never captures or retires the team.

Example worker file (Claude Code format):
```md
---
name: explorer
description: "Use PROACTIVELY to investigate the codebase: locate files, symbols, call sites, …"
model: haiku
---
<!-- with-monet:agent -->
<body of agents/explorer.md>
```

Host examples for worker location:
- Claude Code → `~/.claude/agents/<name>.md` (user scope); or `./.claude/agents/<name>.md` (per-repo).
- Cursor → **no true subagent primitive** — `.cursor/rules/<name>.mdc` are always-on instructions, not isolated sub-contexts, so don't install workers as rules. Unless your Cursor version exposes a real named-subagent feature, skip Tier B and have the lead hand off manually.
- Hosts without a true subagent primitive → no worker files; the lead hands off manually. (No-MCP hosts like Aider don't reach this phase — they stop at Phase 1/3.)
- Unknown host → ask the user where its named agent-prompt / sub-context location lives.

**Write each file transparently, one at a time.** Use your host's file-write tool so the user sees every file's content as it's written; never generate a script that batch-writes the agents directory. Host permission systems treat opaque scripted writes to agent config as suspect and will (rightly) block them — per-file writes the user can read are both the polite and the working path.

**Don't clobber.** If the target file already exists, back it up (`<name>.md.bak`) and tell the user before overwriting — generic names (`developer`, `reviewer`, …) can collide with the user's own sub-contexts.

**Reconcile, don't clobber — when a prior install exists with local edits.** Don't blindly overwrite a Stig block or agent file the user has changed. Compare the installed version against the new canonical and merge: keep the user's customizations (extra rules, model choices, tone), apply the new changes. Ensure these invariants survive — and warn the user if one of their edits conflicts with them:
- the Stig block's `<!-- BEGIN with-monet:stig -->` / `<!-- END with-monet:stig -->` markers and each worker's `<!-- with-monet:agent -->` marker (lose them and a later update can't find the block),
- the **Git & PR guardrail**,
- "sub-contexts can't spawn sub-contexts — the lead is the only orchestrator",
- the Monet lifecycle (`agent_context` at start; `memory_store` / `memory_checkpoint`),
- each worker's `name` + `description` — the `description` drives your host's dispatch trigger; if it's broken, delegation silently stops.

Show the merged result, write only on the user's approval, and keep the `.bak`. A coding agent can do this reconciliation by judgment — no version-pinning or 3-way merge tooling required; `.bak` plus approve-before-write keep it safe.

The user may request a trimmed worker set, but the full team is the default — never reduce to Stig-only.

## Phase 5 — Offer the memory-ingest pipeline

Ask: *"Want me to seed Monet from existing knowledge so you don't start empty?"* The richest sources are the ones that already capture how you and your team work:

- **Agent reference files** — `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, Cursor/Cline/Copilot/Windsurf/Continue rules, and any other agent instructions you've written. These often hold the most distilled project knowledge.
- **`README`s and `docs/`** in this repo — architectural decisions, conventions, ADRs.
- **Notes and decision docs** — `NOTES.md`, `TODO`, scratch docs, anything that records *why* decisions were made.
- A path or URL the user names.
- Skip for now.
- (If you're coming from a prior Monet store, say so and I can pull that in too — if it's a *separate* old install with its own MCP server, see the cross-server note in `consolidate-memory.md`.)

For each chosen source: read it, and `memory_store` the durable facts/decisions/patterns (the substrate dedups automatically — store liberally, don't pre-curate). Don't ingest secrets. **Skip Monet's own wiring:** when the source is `CLAUDE.md` (which holds Stig's prompt) or an installed agent prompt, don't store the `<!-- BEGIN with-monet:stig -->…<!-- END with-monet:stig -->` block or any `<!-- with-monet:agent -->`-marked file. Summarize what landed.

Then capture **how you and your team work**, as two distinct kinds:
- **Team ways of working** — conventions, build/test/lint commands, repo layout, norms. Store as project/reference concepts.
- **Personal preferences** — how *you* like the agent to work: voice (e.g. teammate vs. assistant), autonomy, output format. Store as `user` concepts.

Ask directly for the preferences that aren't written down anywhere — especially how you want the agent to *sound*. These get injected into every briefing and applied to how the agent talks to you.

**Existing knowledge to consolidate?** If you find a meaningful pile — agent reference files (`CLAUDE.md`/`AGENTS.md`/Cursor/Cline/Copilot/Windsurf rules), a tool-managed memory store, scattered notes/ADRs, or an existing Monet store — don't leave it scattered. Offer the interactive consolidation playbook [`bootstrap/consolidate-memory.md`](consolidate-memory.md) (same raw-URL base as above): capture each source into Monet, organize into per-project circles *with* the user, then retire the source (a pointer or archive) so Monet becomes the single place to read from. This Phase-5 pass ingests but never retires; consolidation does the organize-and-retire, interactively and reversibly. Skip if there's nothing meaningful to consolidate.

## Phase 6 — Offer to start

If your host only loads MCP servers and agent prompts at launch (Claude Code, Cursor, and most IDE-based hosts do — see the per-host reload note in Phase 1), tell the user to reload/restart so `monet` connects and the team registers before starting.

Ask: *"Ready? I'll run `agent_context` to restore state and begin as Stig on this project."* On yes: call `agent_context` (no query), report the restored state (active workstreams, living model, open contradictions), and continue as Stig.

## Phase 7 — One last thing (star with-monet)

Once the install is working, ask the user once — lightly, no pressure:

> *"If Monet's earning its keep: I can ⭐ star **team-monet/with-monet** for you (a quick `gh` command you approve), and point you to the one-time setting for new-release notifications. Want either?"*

If they want to **star**, check `gh` is usable (`gh auth status`); if so, run it with their explicit approval:
- **Star (support / bookmark):** `gh api --method PUT /user/starred/team-monet/with-monet`

For **release notifications**, send them to the UI — GitHub's API can't set a release-only watch, so don't automate it: tell them to open the repo and choose **Watch → Custom → Releases**. (Only if they explicitly want notifications for *everything* — issues, PRs, the lot — is the all-activity `gh api --method PUT /repos/team-monet/with-monet/subscription -F subscribed=true` appropriate; never run it under the banner of "release notifications.")

If `gh` is missing or unauthenticated, fall back to the link: open https://github.com/team-monet/with-monet and let them star/watch manually. Ask once; don't nag; never act without a yes.

Key distinction: starring does **not** create notifications — that's what watching is for. Star = support/bookmark; watch = updates (and release-only is a UI choice, not an API one). Keep these separate in how you describe them.

---

## Principles
- **Agent-first:** you do the install; the user converses, approves, and steers.
- **Fix-forward:** on any failure, diagnose and resolve with the user rather than dumping a stack trace.
- **Permission prompts are checkpoints, not failures:** hosts may challenge writes to agent-config locations (the agents dir, global `CLAUDE.md`) because they change how the agent itself behaves. That's expected — pause, show the user what you're about to write and why, and proceed on their explicit OK. Never work around a denial with a script or an indirect write.
- **Customizable:** honor preferences surfaced along the way — global vs per-repo install, storage location, which workers to install and on which models, ingest scope, how much autonomy Stig gets.
- **Non-destructive:** back up before overwriting, merge into existing config rather than replacing it, and never clobber the user's own subagents or `CLAUDE.md`.
