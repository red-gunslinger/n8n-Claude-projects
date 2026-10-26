# Phase 1 Write-Ups

## Project 1: LinkedIn Prospecting Research Assistant

**What it does**
Takes raw prospect text (LinkedIn bio, About section, or recent post) and returns structured prospecting intelligence: likely role seniority, probable pain points, a suggested outreach angle, and a specific personalization hook referencing something concrete the prospect said.

**Architecture**

Webhook (POST /prospect-intake) → HTTP Request (Claude Sonnet 5 analysis) → Code (parse JSON response) → Set (structure output fields) → Respond to Webhook (return result)

Input arrives as `{"prospect_text": "..."}` via webhook POST. Claude is prompted with XML-tagged instructions and a required output schema, then the response is parsed and returned directly to the caller as structured JSON.

**Challenge & lesson learned**
Midway through building, the Code node started throwing `Cannot read properties of undefined (reading 'replace')` — Claude Sonnet 5 introduced extended thinking blocks in its response, so `content[0]` was sometimes a `thinking` block rather than the expected `text` block, breaking the assumption that the answer always lived at index 0.

Fix: instead of indexing by position, the Code node now searches the content array for the block where `type === 'text'`:
```javascript
const blocks = $input.item.json.content;
const textBlock = blocks.find(block => block.type === 'text');
```
Lesson: don't assume a fixed response shape from an LLM API, even one you've tested successfully before — model behavior can change between calls, and defensive parsing (search by type, not position) is worth the extra few lines.

---

## Project 2: Call Notes Follow-Up Pack

**What it does**
Takes rough bullet-point call notes and returns a ready-to-use follow-up pack: key takeaways, action items (with ownership where mentioned), a drafted follow-up email, and a suggested timeframe for the next check-in.

**Architecture**

Webhook (POST /call-notes-intake) → HTTP Request (Claude Sonnet 5 analysis) → Code (parse JSON response) → Set (structure output fields) → Respond to Webhook (return result)

Identical pattern to Project 1 — built by duplicating that workflow and swapping the webhook path, the Claude prompt, and the expected output schema. Confirms the underlying architecture (Webhook → HTTP Request → Code → Set → Respond) is genuinely reusable across different use cases with minimal changes.

**Challenge & lesson learned**
Reused the same extended-thinking-block fix from Project 1 immediately, since the underlying Claude API behavior is the same regardless of which project calls it. This reinforced a broader takeaway from Phase 1: build the parsing/error-handling logic to be generic and resilient once, then treat it as a reusable building block — rather than re-solving the same fragility in every new workflow.

---

## Cross-project notes

- API authentication is handled via N8N's Credential system (Header Auth), not environment variables — avoids exposing keys in exported workflow JSON, so workflows are safe to commit to GitHub.
- All workflows follow a consistent naming convention (`Phase[N]-[Description].json`) tied to Jira epic KAN-9.
- Both workflows ship with `"active": false` — they're prototypes tested via N8N's test webhook URLs, not deployed production automations.

## Phase 3, Step 3: MCP integration (n8n as an MCP server for Claude Code)

**What I built:** An n8n workflow (`mcp-tools`) exposed as an MCP server through the MCP Server Trigger, with two tools: `fetch_audio_news` (HTTP Request Tool pulling a Google News RSS feed on beamforming/microphone AI) and a Calculator. Claude Code connects to it over streamable HTTP (`claude mcp add --scope user --transport http`) and calls it from headless `claude -p` runs.

**Files:** `phase-3/mcp-tools.json` (workflow), `phase-3/n8n-only.json` (MCP config for isolated headless runs), `phase-3/mcp-denied.json` (example of a blocked tool call).

### Key findings

1. **`is_error` is not a success signal.** When the MCP tool was not in `--allowedTools`, the run returned `is_error: false` and `terminal_reason: completed`. The only sign of failure was a non-empty `permission_denials`. A pipeline must treat a run as failed if `is_error` is true, `terminal_reason` is not `completed`, or `permission_denials` is not empty.

2. **Size tool output for the model.** The raw RSS feed was 70KB. Claude Code saved it to a file and tried to search it with PowerShell/Bash (both denied), and used all 6 of 6 turns. Truncating the response in n8n to 8,000 characters cut it to 3 turns with no denials. Trade-off: blunt truncation kept a year-old article as the top result, because the feed isn't sorted by date. The proper fix is to parse, sort and return only titles, dates and links (planned for Project 5).

3. **Fixed context dominates cost.** Each turn carried roughly 40K tokens of context (Claude Code's system prompt plus every connected MCP tool definition) against under 1K output tokens. The claude.ai connectors (about 100 tools) loaded into every run even when unused.

4. **Isolate MCP servers for headless jobs.** `--strict-mcp-config --mcp-config n8n-only.json` loaded only the n8n server and cut cost from $0.100 to $0.057 (about 44%) for the same answer in the same 3 turns. The savings came from cache writes (17.5K to 6.7K tokens).

### Run comparison

| Run | Model | Tool output | Turns | Denials | Est. cost |
|---|---|---|---|---|---|
| 1 | Opus 5.5 | Raw RSS (70KB) | 6/6 | 2 | $0.104 |
| 2 | Sonnet 5 | Truncated (8K chars) | 3 | 0 | $0.100 |
| 3 | Sonnet 5 | Truncated, n8n server only | 3 | 0 | $0.057 |

### Windows gotchas

- Windows PowerShell 5.1 does not support `&&`; run commands separately.
- Use `| Out-File -Encoding utf8` instead of `>` (which writes UTF-16), and set `[Console]::OutputEncoding = [Text.Encoding]::UTF8` first to avoid garbled characters.
- Write JSON config files with `-Encoding ascii`; PowerShell's `utf8` adds a BOM that can break JSON parsing.
- Slash commands (`/login`, `/model`) only work inside interactive Claude Code, not at the PowerShell prompt.
- In this n8n version, a workflow must be **published** (and republished after edits) for the production MCP URL to work.