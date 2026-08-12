# Agent Profiles — File Locations, Tool Mappings, and Chat AI Sources

Reference for the Joule Onboarding & Migration skill. Used in Phases 1 and 4.

`~` = `pathlib.Path.home()`. `%APPDATA%` = `os.environ.get('APPDATA')` (Windows only).

---

## Technical Agent File Locations

### Claude Code
- **Memory:** `~/CLAUDE.md`, `~/.claude/CLAUDE.md`
- **Skills dir:** `~/.claude/skills/*/SKILL.md`
- **MCP config:** `~/.claude/settings.json` → `mcpServers`
- **Shared format:** `{project}/AGENTS.md`

### Cursor
- **Rules (current):** `{project}/.cursor/rules/*.mdc`
- **Rules (legacy):** `{project}/.cursorrules`
- **Global:** No user-level rules file → paste mode
- **MCP (global):** `~/.cursor/mcp.json`
- **MCP (project):** `{project}/.cursor/mcp.json`
- **Skills dir:** None
- **Note:** `.mdc` files are Markdown with optional frontmatter (`description`, `globs`, `alwaysApply`)

### Windsurf
- **Rules (current):** `{project}/.windsurf/rules/*.md`
- **Rules (legacy):** `{project}/.windsurfrules`
- **Global:** No user-level rules file → paste mode
- **MCP (Windows):** `%USERPROFILE%\.codeium\windsurf\mcp_config.json`
- **MCP (macOS/Linux):** `~/.codeium/windsurf/mcp_config.json`
- **Skills dir:** None

### Cline (VS Code extension)
- **Rules:** `{project}/.clinerules` or `{project}/.clinerules/` directory
- **Global:** No user-level rules file → paste mode
- **MCP (Windows):** `%APPDATA%\Code\User\globalStorage\saoudrizwan.claude-dev\settings\cline_mcp_settings.json`
- **MCP (macOS):** `~/Library/Application Support/Code/User/globalStorage/saoudrizwan.claude-dev/settings/cline_mcp_settings.json`
- **Skills dir:** None

### Roo Code (VS Code, Cline fork)
- **Rules:** `{project}/.roorules` or `{project}/.clinerules`
- **Global:** No user-level rules file → paste mode
- **MCP (Windows):** `%APPDATA%\Code\User\globalStorage\rooveterinaryinc.roo-cline\settings\cline_mcp_settings.json`
- **MCP (macOS):** `~/Library/Application Support/Code/User/globalStorage/rooveterinaryinc.roo-cline/settings/cline_mcp_settings.json`
- **Skills dir:** None

### GitHub Copilot
- **Project rules:** `{project}/.github/copilot-instructions.md`
- **Scoped instructions:** `{project}/.github/instructions/*.instructions.md`
- **CLI global:** `~/.copilot/`
- **CLI agents:** `~/.copilot/agents/*.agent.md`
- **MCP (CLI):** `~/.copilot/mcp-config.json`
- **Skills dir:** `~/.copilot/agents/` (CLI only)

### Codex (OpenAI)
- **Rules:** `{project}/AGENTS.md`
- **Memory:** `~/.codex/memories/*.md`
- **Config:** `~/.codex/config.toml`
- **Skills dir:** None

### Gemini CLI
- **Rules:** `GEMINI.md` (project root)
- **MCP:** `~/.gemini/settings.json`
- **Skills dir:** None

### Aider
- **Config:** `.aider.conf.yml` or `CONVENTIONS.md` (project root)
- **MCP:** Not natively supported
- **Skills dir:** None

### Continue.dev
- **Config / system message:** `~/.continue/config.json` → `systemMessage`
- **Context providers:** `~/.continue/config.json` → `contextProviders`
- **Skills dir:** None

---

## Chat AI Sources (Branch B)

### ChatGPT (OpenAI)

**Custom Instructions** (NOT in data export — must retrieve manually):
- chatgpt.com → Profile → Settings → Personalization → Custom Instructions

**Memory items:**
- chatgpt.com → Profile → Settings → Personalization → Manage Memories

**Conversation History Export:**
- chatgpt.com → Profile → Settings → Data Controls → Export Data → Confirm
- Receive ZIP via email (link expires in 24 hours)
- Unzip → find `conversations.json` (or `conversations-000.json` etc. for large histories)
- Place in Documents/ folder

**Size-adaptive analysis script** (write to scratch dir, run via execute):
```python
import json, pathlib, sys, re
from collections import Counter
sys.stdout.reconfigure(encoding='utf-8')

conv_file = pathlib.Path.home() / 'Documents' / 'conversations.json'
if not conv_file.exists():
    conv_file = pathlib.Path.home() / 'Documents' / 'conversations-000.json'
if not conv_file.exists():
    print("ERROR: conversations.json not found in Documents/")
    sys.exit(1)

size_mb = conv_file.stat().st_size / (1024 * 1024)
print(f"File size: {size_mb:.1f} MB")

if size_mb > 200:
    # ---- TITLE-ONLY MODE (>200MB) ----
    print("Mode: TITLE-ONLY (regex extraction)")
    with open(conv_file, 'r', encoding='utf-8', errors='replace') as f:
        content = f.read(40 * 1024 * 1024)  # First 40MB
    titles = re.findall(r'"title"\s*:\s*"([^"\\]*(?:\\.[^"\\]*)*)"', content)
    print(f"Titles extracted: {len(titles)}")
    stopwords = {'the','a','an','and','or','is','in','of','to','for','with','how','what','why','can','i','my','you','this','that'}
    words = [w.lower() for t in titles for w in t.split() if w.lower() not in stopwords and len(w) > 3]
    print(f"\nTop topics from {len(titles)} conversation titles:")
    for word, n in Counter(words).most_common(15):
        print(f"  {word}: {n}")
    print("\nNOTE: File too large for full message analysis. Showing title patterns only.")

elif size_mb > 50:
    # ---- STREAMING MODE (50-200MB) ----
    print("Mode: STREAMING (one conversation at a time)")
    decoder = json.JSONDecoder()
    user_messages = []
    titles = []
    with open(conv_file, 'r', encoding='utf-8', errors='replace') as f:
        raw = f.read()
    idx = raw.find('[')
    if idx == -1:
        print("ERROR: Could not find JSON array start")
        sys.exit(1)
    idx += 1
    conv_count = 0
    while idx < len(raw):
        while idx < len(raw) and raw[idx] in ' \t\n\r,':
            idx += 1
        if idx >= len(raw) or raw[idx] == ']':
            break
        try:
            conv, end_idx = decoder.raw_decode(raw, idx)
            idx = end_idx
            conv_count += 1
            if conv.get('title'):
                titles.append(conv['title'])
            for node in conv.get('mapping', {}).values():
                msg = node.get('message') or {}
                if msg.get('author', {}).get('role') == 'user':
                    for part in msg.get('content', {}).get('parts', []):
                        if isinstance(part, str) and part.strip():
                            user_messages.append(part.strip())
        except (json.JSONDecodeError, ValueError):
            next_obj = raw.find(',\n  {', idx)
            if next_obj == -1:
                break
            idx = next_obj + 1
    print(f"Conversations processed: {conv_count} | User messages: {len(user_messages)}")

else:
    # ---- FULL MODE (<50MB) ----
    print("Mode: FULL")
    with open(conv_file, 'r', encoding='utf-8') as f:
        data = json.load(f)
    user_messages = []
    titles = [c.get('title', '') for c in data if c.get('title')]
    for conv in data:
        for node in conv.get('mapping', {}).values():
            msg = node.get('message') or {}
            if msg.get('author', {}).get('role') == 'user':
                for part in msg.get('content', {}).get('parts', []):
                    if isinstance(part, str) and part.strip():
                        user_messages.append(part.strip())
    print(f"Conversations: {len(data)} | User messages: {len(user_messages)}")

# ---- ANALYSIS (streaming + full modes) ----
if 'user_messages' in dir() and user_messages:
    intro_starters = ("i am ", "i'm ", "i work ", "my role", "my job", "i'm a ", "as a ")
    intros = [m[:250] for m in user_messages if m.lower().startswith(intro_starters) and len(m) < 500]
    print(f"\nSelf-introductions ({len(intros)} found):")
    for i in intros[:8]:
        print(f"  {i[:120]}")
    pref_words = ["always", "never", "please", "i prefer", "i like", "i want", "format", "don't", "avoid"]
    prefs = [m[:250] for m in user_messages if any(w in m.lower() for w in pref_words) and len(m) < 400]
    print(f"\nPreference statements ({len(prefs)} found):")
    for p in prefs[:8]:
        print(f"  {p[:120]}")
    if 'titles' in dir() and titles:
        stopwords = {'the','a','an','and','or','is','in','of','to','for','with','how','what','why','can','i','my','you','this','that'}
        words = [w.lower() for t in titles for w in t.split() if w.lower() not in stopwords and len(w) > 3]
        print(f"\nTop topics from {len(titles)} conversation titles:")
        for word, n in Counter(words).most_common(12):
            print(f"  {word}: {n}")
```

### Google Gemini
- gemini.google.com → Settings & help → Personal Intelligence → Instructions for Gemini
- gemini.google.com/saved-info for saved preferences
- No chat history export available.

### Microsoft Copilot
- copilot.microsoft.com → Settings → Personalization → Custom Instructions
- Ask Copilot: "What do you know about me?"
- No automated export.

---

## Tool Pattern → Joule Equivalent

| Capability pattern | Joule equivalent | Transfer |
|---|---|---|
| Terminal / shell command execution | `execute` | Full |
| File read | `read_file` | Full |
| File write / edit | `write_file` / `edit_file` | Full |
| Directory listing / file search | `ls` / `glob` / `grep` | Full |
| Public web fetch / search | `web_search` | Full |
| Spawning parallel subagents | `task` | Full |
| LLM reasoning and text generation | Native | Full |
| MCP tool calls (if Joule has equivalent) | Joule connector | Full |
| Browser navigation to public URL | `web_search` (partial) | Partial |
| Authenticated web session / SSO | No equivalent | None |
| CDP / Playwright / Puppeteer | No equivalent | None |
| Keyboard / mouse / clipboard automation | No equivalent | None |
| VS Code / IDE extension API calls | No equivalent | None |
| Background daemon / scheduled process | No equivalent | None |

---

## Common MCP Servers → Joule Equivalents

| MCP server | Joule equivalent |
|---|---|
| `server-filesystem` | Built-in file tools (no setup) |
| `server-brave-search` or any search MCP | Built-in `web_search` (no setup) |
| `server-github` | Check Joule connector catalog |
| Any HTTP/SSE server with a public URL | Add via Extensions → Connectors → + New |
| Local stdio MCP (runs a local command) | No direct equivalent — needs hosted HTTP version |
| Playwright / CDP browser MCP | No equivalent in Joule |
| `agentmemory`, `mem0`, `memories.sh` | Joule `memory/memory.md` via joule-memory skill |
| Telegram / Slack / Teams MCPs | No equivalent; Joule has built-in Outlook email |
