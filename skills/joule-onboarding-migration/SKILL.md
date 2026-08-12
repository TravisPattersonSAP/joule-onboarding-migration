---
name: joule-onboarding-migration
description: >-
  Guides anyone through setting up Joule Desktop for the first time or migrating from another AI tool. Supports three paths: (A) technical agent migration from Claude Code, Cursor, Windsurf, Cline, Roo Code, GitHub Copilot, or Codex; (B) switching from ChatGPT, Gemini, or Microsoft Copilot — imports custom instructions and analyses conversation history; (C) first-time AI setup with a guided interview that builds your Joule memory from scratch. Works for any user on any OS. No config files required. Trigger phrases: 'migrate from Claude', 'migrate from Cursor', 'migrate from ChatGPT', 'I use ChatGPT and want to switch to Joule', 'help me set up Joule', 'I'm new to AI agents', 'import my memory', 'set up Joule from scratch', 'migrate my AI setup'.
allowed-tools: execute read_file write_file ls glob task
metadata:
  version: 2.3.1
  tags: migration onboarding setup memory chatgpt cursor claude new-user skills joule
---

# Joule Onboarding & Migration

Helps anyone get the most out of Joule Desktop — whether you're migrating from a technical AI agent, switching from a chat AI like ChatGPT, or setting up Joule for the very first time.

**No config files required.** Works for every experience level.

---

## Tone Calibration

Adapt your language based on the user's background:

- **Branch A (Technical Agent):** Concise and technical. Skip concept explanations. Use precise file paths, tool names, and jargon freely.
- **Branch B (Chat AI) or C (New User):** Friendly and plain. Define terms on first use in parentheses. Use short sentences. Be encouraging. Never show raw error output without explaining what it means in plain language.

---

## Phase Routing

Use this table to determine which phases to run once the branch is known. If the user only wants part of the migration (e.g., memory only, or connections only), confirm scope before starting and skip irrelevant phases.

| Branch | Phases to run |
|--------|---------------|
| A — Technical Agent | 0 → 1a → 1b → 1c → 1d → 1e → 2 → 3 → 4 → 5 → 6 |
| B — Chat AI | 0 → 1a → 1b → B1 → B2 → B3 → B4 → 2 → 3 → 6 |
| C — New User | 0 → 1a → 1b → C1 → C2 → C3 → 3 → 6 |

---

## Checkpoint Management

The skill maintains a `.joule-migration-checkpoint.json` file alongside the migration report. This enables exact resume — even mid-phase — without re-doing completed work.

**Checkpoint path:** Determined at end of Phase 1a. Use the same directory as the migration report. Store as `CHECKPOINT_PATH` for the session.

**Structure:**
```json
{
  "version": "2.3.1",
  "source_agent": "<agent or ChatGPT/New>",
  "branch": "A|B|C",
  "phases_complete": [],
  "memory_rules_written": 0,
  "memory_rules_discarded": 0,
  "triage_chunks_done": 0,
  "skills": {
    "<name>": { "status": "assessed|pending|skipped", "feasibility": "full|partial|none" }
  },
  "connections": {
    "<name>": { "status": "mapped|no-equivalent", "joule_equivalent": "..." }
  },
  "last_updated": "ISO timestamp",
  "last_error": null
}
```

**Write/update checkpoint after every significant action** — write this script to scratch and run via `execute` each time, substituting the actual updates:

```python
import pathlib, json, datetime, sys
sys.stdout.reconfigure(encoding='utf-8')
cp_path = pathlib.Path("CHECKPOINT_PATH")
cp = {}
if cp_path.exists():
    try:
        with open(cp_path, 'r', encoding='utf-8') as f:
            cp = json.load(f)
    except: pass
# Apply your specific updates here, for example:
# cp.setdefault("phases_complete", []).append("3")
# cp["memory_rules_written"] = 14
# cp.setdefault("skills", {})["skill-name"] = {"status": "assessed", "feasibility": "full"}
cp["last_updated"] = datetime.datetime.now().isoformat()
with open(cp_path, 'w', encoding='utf-8') as f:
    json.dump(cp, f, indent=2, ensure_ascii=False)
print("Checkpoint saved.")
```

**Trigger checkpoints after:**
- Phase 1 complete → record `source_agent`, `branch`
- Each memory rule batch written → update `memory_rules_written`
- Each triage chunk processed → increment `triage_chunks_done`
- Each skill assessed → update `skills[name]`
- Each connection mapped → update `connections[name]`
- Any error → set `last_error` with the error message before surfacing it

---

## Phase 0 — Resume Check

Check for an existing checkpoint (most granular) or migration report (fallback):

```python
import pathlib, json, glob as glob_mod, sys
sys.stdout.reconfigure(encoding='utf-8')
home = pathlib.Path.home()
cp_found = None
for pattern in [
    str(home / "Obsidian Vault" / "02 - Areas" / "Work" / ".joule-migration-checkpoint.json"),
    str(home / "Obsidian Vault" / ".joule-migration-checkpoint.json"),
    str(home / "Documents" / ".joule-migration-checkpoint.json"),
    ".joule-migration-checkpoint.json"
]:
    matches = glob_mod.glob(pattern)
    if matches:
        cp_found = sorted(matches)[0]
        break
if cp_found:
    try:
        with open(cp_found, 'r', encoding='utf-8') as f:
            cp = json.load(f)
        skills = cp.get("skills", {})
        assessed = [k for k,v in skills.items() if v.get("status") == "assessed"]
        pending  = [k for k,v in skills.items() if v.get("status") == "pending"]
        print("CHECKPOINT:", cp_found)
        print("AGENT:", cp.get("source_agent","?"), "| BRANCH:", cp.get("branch","?"))
        print("PHASES_DONE:", cp.get("phases_complete", []))
        print("MEMORY_RULES:", cp.get("memory_rules_written",0), "written,", cp.get("memory_rules_discarded",0), "discarded")
        print(f"SKILLS: {len(assessed)} assessed, {len(pending)} pending")
        print("PENDING_SKILLS:", pending)
        print("LAST_UPDATED:", cp.get("last_updated","?"))
    except Exception as e:
        print("CHECKPOINT_PARSE_ERROR:", e)
else:
    found = []
    for pattern in [
        str(home / "Obsidian Vault" / "02 - Areas" / "Work" / "Joule Migration Report*.md"),
        str(home / "Obsidian Vault" / "Joule Migration Report*.md"),
        str(home / "Documents" / "joule-migration-report*.md"),
        "joule-migration-report*.md"
    ]:
        found.extend(sorted(glob_mod.glob(pattern)))
    print("REPORT:", '\n'.join(found) if found else "NOT_FOUND")
```

- **Checkpoint found:** Show summary. Ask: *"I found a checkpoint from [date] — [N] memory rules written, [M] skills still pending. Continue where you left off or start fresh?"*
- **Report found (no checkpoint):** Read it, offer to continue or start fresh.
- **Nothing found:** Proceed to Phase 1.

---

## Phase 1 — Discover Environment

### 1a. Find home directory and OS

```python
import pathlib, platform, os, sys
sys.stdout.reconfigure(encoding='utf-8')
print("HOME:", pathlib.Path.home())
print("OS:", platform.system())
print("APPDATA:", os.environ.get('APPDATA', 'N/A'))
```

If `python` fails, try `python3`, then `py` (Windows Python Launcher). If all fail, ask the user for their home directory path and OS.

Once HOME is known, set `CHECKPOINT_PATH` to: `HOME/Documents/.joule-migration-checkpoint.json` (or the Obsidian vault path if one is detected later).

### 1b. Detect source — who are you migrating from?

First, check whether the user's message already tells you (e.g. "I'm coming from ChatGPT", "migrating from Cursor"). If clear, use that.

Otherwise, run detection:

```python
import pathlib, os, sys
sys.stdout.reconfigure(encoding='utf-8')
home = pathlib.Path.home()
cwd = pathlib.Path.cwd()
appdata = os.environ.get('APPDATA', '')
markers = {
    'Claude Code':    [home/'.claude', home/'CLAUDE.md'],
    'Cursor':         [home/'.cursor', cwd/'.cursor', cwd/'.cursorrules'],
    'Windsurf':       [home/'.codeium', cwd/'.windsurf', cwd/'.windsurfrules'],
    'Cline':          [cwd/'.clinerules', pathlib.Path(appdata)/'Code'/'User'/'globalStorage'/'saoudrizwan.claude-dev' if appdata else pathlib.Path('/x')],
    'Roo Code':       [cwd/'.roorules', pathlib.Path(appdata)/'Code'/'User'/'globalStorage'/'rooveterinaryinc.roo-cline' if appdata else pathlib.Path('/x')],
    'GitHub Copilot': [home/'.copilot', cwd/'.github'/'copilot-instructions.md'],
    'Codex':          [home/'.codex'],
    'Gemini CLI':     [cwd/'GEMINI.md', home/'.gemini'],
    'Aider':          [cwd/'.aider.conf.yml', cwd/'CONVENTIONS.md'],
    'Continue.dev':   [home/'.continue'],
}
detected = [a for a, paths in markers.items() if any(p.exists() for p in paths)]
print('DETECTED:', ', '.join(detected) if detected else 'NONE')
```

**If exactly one technical agent is detected:** confirm with user → store as SOURCE_AGENT → **go to Branch A**.

**If multiple agents are detected:** list them and ask:
> "I found [A] and [B] on this machine. Which would you like to migrate from?"
Wait for the answer, store as SOURCE_AGENT, then proceed to Branch A.

**If nothing is detected:** ask:
> "I couldn't find an AI agent config on this machine. Which best describes you?
>
> 1. I use **ChatGPT, Gemini, or Microsoft Copilot** and want to bring my preferences over
> 2. I'm **new to AI assistants** and want to set Joule up from scratch
> 3. I use a coding agent (**Cursor, Windsurf, Claude Code**, etc.) but keep my rules in a project folder"

- Option 1 → **Branch B** (Chat AI Migration)
- Option 2 → **Branch C** (New User Onboarding)
- Option 3 → ask for project path or offer paste mode → **Branch A**

**Sensitive data reminder (always show before reading any file):**
> "⚠️ Before I read your configuration file — if it contains credentials, customer data, API keys, or confidential business information, please remove or replace those sections first. I'll also flag anything that looks sensitive during triage."

After branch is confirmed, write initial checkpoint: `source_agent`, `branch`.

---

## Branch A: Technical Agent Migration

*For users migrating from Claude Code, Cursor, Windsurf, Cline, Roo Code, GitHub Copilot, Codex, Gemini CLI, Aider, Continue.dev, or similar.*

### 1c. Find memory / rules file

Using SOURCE_AGENT and `references/agent-profiles.md`, search for the user-level memory file.

For agents with project-level-only rules (Cursor, Windsurf, Cline, Roo Code):
> "[Agent] stores rules at the project level, not globally. You can either paste your rules here directly, or tell me your project folder path and I'll look for the rules file there."

**For large files (>300 lines): read and categorise each chunk immediately — do not accumulate all chunks before categorising.**
Use `read_file(path, offset=0, limit=200)`, then `offset=200, limit=200`, repeating until done. After each chunk, output your running category tallies and merge across chunks as you go.

Also check `{cwd}/AGENTS.md` regardless of agent.

### 1d. Inventory skills / commands

Using `references/agent-profiles.md`, find the skills directory for SOURCE_AGENT. Only Claude Code has a dedicated skills dir (`~/.claude/skills/`). For others, treat Category B items from Phase 2 triage as skill candidates.

```python
import pathlib, sys
sys.stdout.reconfigure(encoding='utf-8')
skills_path = pathlib.Path.home() / '.claude' / 'skills'
if skills_path.exists():
    dirs = [p.name for p in sorted(skills_path.iterdir()) if p.is_dir()]
    print('\n'.join(dirs) if dirs else 'NO_SKILLS_FOUND')
else:
    print('NOT_FOUND')
```

### 1e. Read MCP config

Using `references/agent-profiles.md`, find and read the MCP config for SOURCE_AGENT and OS. Extract `mcpServers` entries.

For Cline / Roo Code on Windows: the path starts with `%APPDATA%` — substitute APPDATA_DIR from Phase 1a.

Write checkpoint: phases_complete includes `1a`, `1b`, `1c`, `1d`, `1e`.

*→ Proceed to Phase 2 (Triage)*

---

## Branch B: Chat AI Migration

*For users coming from ChatGPT, Google Gemini, or Microsoft Copilot.*

Substitute `[CHAT_AI_NAME]` with the actual product name:

> "Welcome to Joule Desktop! You're switching from [CHAT_AI_NAME].
>
> Here's the key difference: **[Chat AI] starts fresh every conversation** — you have to re-explain who you are each time. **Joule remembers things permanently** and can run repeatable workflows for tasks you do every day.
>
> In the next few steps, I'll capture what's been working for you and turn it into Joule's long-term memory — so you never have to re-introduce yourself again."

### B1. Retrieve custom instructions

**ChatGPT:**
> "Go to: Profile → Settings → Personalization → Custom Instructions. Copy both text boxes and paste them here."

**Google Gemini:**
> "Go to: Settings & help → Personal Intelligence → Instructions for Gemini. Copy all instructions and paste here. Also check gemini.google.com/saved-info."

**Microsoft Copilot:**
> "Go to: Settings → Personalization → Custom Instructions. Copy and paste here. You can also ask Copilot 'What do you know about me?' and paste that too."

If the user has no custom instructions: proceed to B3.

### B2. Retrieve memory items (ChatGPT users only)

> "Go to: Profile → Settings → Personalization → Manage Memories. Paste a summary here, or ask ChatGPT: 'List everything you remember about me.'"

Skip for Gemini / Copilot users.

### B3. Analyze conversation history (optional, ChatGPT only)

> "Would you like me to analyze your ChatGPT conversation history? Go to: Profile → Settings → Data Controls → Export Data → Confirm. Place **conversations.json** in your Documents folder when the download arrives, then let me know."

If they provide the file, run the **size-adaptive analysis** from `references/agent-profiles.md`:
1. Read `references/agent-profiles.md` and extract the size-adaptive Python analysis script.
2. Write it to the scratch directory using `write_file`.
3. Run it using `execute`. The script auto-selects one of three modes based on file size:
   - **< 50 MB** — full analysis (messages, preferences, titles)
   - **50–200 MB** — streaming mode (processes one conversation at a time, never fully in memory)
   - **> 200 MB** — title-only mode (regex extraction of titles only; completes in seconds)
4. Present findings as: topics discussed most, context re-explained repeatedly (memory rule candidates), recurring task types (skill candidates).

If they skip: proceed to B4.

### B4. Gap-fill interview

Based on what's missing from B1–B3, ask up to 3 targeted questions, one at a time:

- If no role/domain context found: *"What's your job or field? What do you primarily use AI for day-to-day?"*
- If no style preferences found: *"How do you like AI to respond — concise and to the point, or detailed with examples? Formal or casual?"*
- If no constraints found: *"Are there things you'd want Joule to always be careful about?"*

Wait for each answer before asking the next. Follow up gently if answers are very brief.

*→ Pass all collected content (B1 + B2 + B3 findings + B4 answers) to Phase 2 (Triage).*

---

## Branch C: New User Onboarding

*For users new to AI agents or setting up Joule for the first time.*

> "Welcome to Joule Desktop! Let me explain what makes Joule different from chatbots like ChatGPT:
>
> 🧠 **Memory** — You teach Joule facts about yourself once, and it applies them in every future conversation.
>
> ⚡ **Skills** — Reusable mini-workflows for tasks you do repeatedly.
>
> 🔌 **Connectors** — Joule can connect to your work tools: email, calendar, Jira, internal systems, and more.
>
> Let's spend 3 minutes setting up your memory so Joule knows you from the start."

### C1. Guided interview

Ask these questions **one at a time**:

1. *"What's your name, and what do you do for work?"*
2. *"What kinds of tasks do you most want AI help with?"*
3. *"How do you prefer AI to respond — short and direct, or detailed? Formal or casual?"*
4. *"Are there things you'd want Joule to always be careful about?"*
5. *"Anything else you'd want Joule to always know about you? No pressure."*

If Q1 or Q2 answers are very brief, ask a gentle follow-up before moving on.

### C2. Convert answers to memory rules

Draft rules directly from interview answers:
- Q1 → role and domain context rule
- Q2 → primary use case rule
- Q3 → response style rule(s)
- Q4 → constraint / guardrail rule(s)
- Q5 → any additional context rules

Show drafts and ask: *"Here are [N] things I'll have Joule remember about you. Anything to change or add?"*

### C3. Suggest starter skills

Based on Q2 answers, suggest 2–3 relevant skill types and direct the user to find them:

| If they mention... | Look for a skill that... |
|---|---|
| Writing, drafting, emails, messages | Drafts responses or communications |
| Data analysis, Excel, spreadsheets | Works with spreadsheets and tabular data |
| Presentations, slides | Creates or formats slide decks |
| Investigations, troubleshooting, root cause | Analyses logs, errors, or system issues |
| Word documents, reports | Creates or edits Word documents |
| PDFs | Reads, extracts, or creates PDFs |

Say: *"Based on what you told me, these are the types of skills worth looking for. Browse Extensions → Skills or search [skills.cloud.sap](https://skills.cloud.sap) to find matching ones for your setup."*

*→ Routing: Skip Phase 2, Phases 4–5. Go directly to Phase 3 then Phase 6.*

---

## Phase 2 — Triage Content

*(Branch A and B only. Skip for Branch C — see Phase Routing table.)*

**Categorise each chunk as it arrives — never accumulate all chunks before categorising.**

For files > 300 lines: read one 200-line chunk, categorise its contents immediately, record running tallies, then read the next chunk. After each chunk output:
> *"Chunk [N] done — running totals: A:[n] memory rules, B:[n] workflows, C:[n] references, D:[n] discards"*

Merge tallies across all chunks. Present the final triage summary only after all chunks are processed.

If resuming from checkpoint: read `triage_chunks_done` and start from `offset = triage_chunks_done * 200`.

After each chunk, write checkpoint: increment `triage_chunks_done`.

**Category A — Memory Rules:** Compliance guardrails, behavioral preferences, role/domain context, tool gotchas, workflow conventions.

**Category B — Skill Workflows:** Recurring procedures with 3+ steps, a clear trigger, and consistent output.

**Category C — Reference Material:** Technical tables, field mappings, service inventories too detailed for memory.

**Category D — Discard:** Agent-specific hooks, CDP/browser automation setup, plugin config, one-time environment setup.

Flag any sensitive data (credentials, API keys, customer names) and remind the user not to include it in memory.

Present triage summary. Wait for confirmation. If the user says "just do it", proceed without further confirmation.

---

## Phase 3 — Write Memory Rules

### Volume cap — 15–20 rules maximum

Priority: compliance/security → daily workflow conventions → technical gotchas → context.

### Entry format

```
## [YYYY-MM-DD] <short label>
RULE: <clear, actionable rule — "Always X", "Never Y", "When Z use W">
SOURCE: migrated from <SOURCE_AGENT or interview> on YYYY-MM-DD
CONFIRMATIONS: 0
VIOLATIONS: 0
```

### Batched write-as-you-go — never hold more than 5 unwritten rules at a time

1. Draft the first batch of up to 5 rules.
2. Show the batch and wait for confirmation (or edits).
3. Write the confirmed batch to `memory/memory.md` **immediately** — do not wait for all batches.
4. Write checkpoint: `memory_rules_written += batch_size`.
5. Confirm: *"[N] rules saved. [M] to go."*
6. Repeat for the next batch.

If the 20-rule cap is reached mid-batch: complete and write the current batch, then list remaining candidates as discarded and offer swaps.

If `memory/memory.md` does not exist, create it with a `# Memory` header before the **first** batch write. If it exists, read it first and skip duplicates.

After all batches: *"[total] rules saved to memory/memory.md."* List discarded items: *"I also dropped [M] lower-priority items: [brief list]. Let me know if any matter and I'll include them instead."*

---

## Phase 4 — Skills Inventory

*(Branch A only. Skip for Branches B and C.)*

### Step 4a — Load work queue (resume-aware)

If a checkpoint exists, read `skills`. Skills with `status: "assessed"` are already done — skip them. Collect `status: "pending"` (or unrecorded) skills as the work queue. Show: *"[N] skills to assess, [M] already done from previous session."*

### Step 4b — Parallel assessment in batches of 4

Use the `task` tool to assess up to 4 skills simultaneously. Each task description must include:
- The absolute path to the skill's SKILL.md
- The tool pattern → Joule equivalent table (paste inline from `references/agent-profiles.md`)
- Instructions to return: `name`, `feasibility` (Full / Partial / None), `description` (1–2 sentences), `joule_adaptation`

Dispatch batch, await all results, then dispatch the next batch. Show progress after each batch: *"[N]/[total] skills assessed."*

### Step 4c — Save immediately after each batch

After each batch of tasks returns:
1. Append each skill's briefing card to the migration report file immediately.
2. Write checkpoint: `skills[name] = {status: "assessed", feasibility: "..."}` for each skill in the batch.

**Briefing card format (appended to report after each batch):**
```
### Skill: <name>
**Feasibility:** Full / Partial / None
**What it does:** <1-2 sentences>
**Joule adaptation:** <what changes>
**To migrate:** In Joule, say "create a skill that does [X]" - the skill-creator activates automatically.
```

---

## Phase 5 — Connections Mapping

*(Branch A only. Skip for Branches B and C.)*

Map each MCP server from Phase 1e using `references/agent-profiles.md`.

For MCPs that can be added to Joule: *"Extensions icon (left rail) → Connectors → + New → Add Connector → Name: [name], URL: [url] → OK."*

If it's a local stdio server: *"This MCP runs as a local command — Joule requires HTTP/SSE endpoint. Check if a hosted version is available."*

After each connection mapped, write checkpoint: `connections[name] = {status, joule_equivalent}`.

---

## Phase 6 — Migration Report

Save to the first available path (absolute — substitute HOME from Phase 1a):
1. `HOME/Obsidian Vault/02 - Areas/Work/Joule Migration Report - YYYY-MM-DD.md`
2. `HOME/Obsidian Vault/Joule Migration Report - YYYY-MM-DD.md`
3. `HOME/Documents/joule-migration-report-YYYY-MM-DD.md`
4. `joule-migration-report-YYYY-MM-DD.md` (working directory fallback)

Set `status` based on remaining work:
- **Branch C** or nothing remaining: `status: complete`
- Pending skills/connections: `status: in-progress`

```markdown
---
tags: [type/work-log, area/ai-tools, tool/joule-migration]
created: <YYYY-MM-DD>
status: <complete or in-progress>
---

# Joule Desktop Migration — <YYYY-MM-DD>

Source: **<SOURCE_AGENT or ChatGPT/Gemini/Copilot/New Setup>**

## Memory Rules
**Written:** <N> rules → `memory/memory.md` (<M> discarded)
**To review:** say "recall memory" in any Joule conversation.

## Skills / Workflows
*(Briefing cards appended here by Phase 4 as each batch is assessed)*

| Item | Feasibility | Status | Notes |
|---|---|---|---|
| `<name>` | Full/Partial/None | Migrated / Pending / Skipped | |

## Connections
| Source MCP | Joule equivalent | Status |
|---|---|---|
| `<name>` | `<equivalent>` | Connected / Needs setup / No equivalent |

## Remaining Work
### Skills to migrate
- [ ] `<skill>` (Partial) — <what it does and adaptation needed>
### Connections to set up
- [ ] `<mcp>` → <steps>

## See Also
- [[MOC - AI Tools & Automation]]
```

After saving: write final checkpoint with all completed phases and `status: "complete"` if nothing remains.

*"All done! Report saved to [path]. Checkpoint saved to [checkpoint path]. You can run this skill again anytime to continue — it picks up exactly where you left off."*

---

## Error Handling

| Scenario | Action |
|---|---|
| Python not found | Try python → python3 → py. If all fail, ask user for Python path. Provide python.org/downloads if needed. |
| No agent detected, user unsure | Ask: "Do you use chat.openai.com regularly? Or any coding tools like VS Code, Cursor?" Guide from answer. |
| User has no custom instructions in chat AI | Reassure and proceed to B4. |
| conversations.json not found | Ask: "Did you unzip the file? Place conversations.json in your Documents folder." |
| conversations.json split into numbered files | Parse conversations-000.json first. Note: repeat for each file for full analysis. |
| conversations.json < 50 MB | Normal json.load() — full analysis. |
| conversations.json 50–200 MB | Streaming mode from agent-profiles.md — one conversation at a time. |
| conversations.json > 200 MB | Title-only mode — regex extraction, completes in seconds. |
| Crash during Phase 3 | Checkpoint preserves all written batches. Resume picks up from next unwritten batch. |
| Crash during Phase 4 | Checkpoint shows assessed vs pending. Resume skips assessed skills. |
| Checkpoint file corrupted/missing | Fall back to migration report for resume; start checkpoint fresh. |
| Sensitive data spotted during triage | Flag: "I noticed [type] in section [X]. Excluding from memory. Verify this is intentional." |
| Branch C interview answer too brief | Follow up: "Could you say a bit more? For example, [specific prompt]." Don't write a vague rule. |
| MCP is local stdio server | Note: no direct Joule equivalent — needs hosted HTTP/SSE version. |
| Obsidian vault not found | Save report to working directory root. |
| memory/memory.md already exists | Read it first; append only new topics; explain skipped entries. |