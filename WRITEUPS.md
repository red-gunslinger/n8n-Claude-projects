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