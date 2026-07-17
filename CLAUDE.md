# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of **n8n workflow exports** (`.json`) that call the Claude (Anthropic) API, developed in phases as learning/prototyping exercises. There is no application to build, run, or test locally — the JSON files are imported into an n8n instance and executed there. `hello.py` is a standalone "Phase 0" sanity script unrelated to the workflows.

## Working with the workflows

Each `PhaseN-*.json` file is a complete, self-contained n8n workflow export (nodes + connections + settings). To iterate:
- **Edit** the JSON directly, or import into n8n → modify visually → re-export over the file.
- **Import/export** happens in the n8n UI (Workflows → Import from File / three-dot menu → Download). There is no CLI step in this repo.
- The filename `name` field and the top-level `"id"`/`"versionId"` are set by n8n on export — don't hand-craft them for new files; let n8n assign them.

## Architecture of a workflow

All current workflows follow the same linear chain built around a raw Anthropic API call (no n8n LangChain/AI nodes):

```
Manual Trigger → HTTP Request (Anthropic) → [Code: JSON.parse] → Set (Edit Fields)
```

- **HTTP Request node** POSTs to `https://api.anthropic.com/v1/messages`. Auth is `genericCredentialType` / `httpHeaderAuth` referencing a credential named **"Anthropic API Key"** (the `x-api-key` header lives in the credential, not the workflow). Headers `anthropic-version: 2023-06-01` and `content-type: application/json` are set explicitly in the node.
- The request body is a hand-written JSON string in `jsonBody` (model, `max_tokens`, `messages`). Model in the existing files is `claude-sonnet-4-6`.
- **Code node** (structured-output workflows only) runs `JSON.parse($input.item.json.content[0].text)` to turn Claude's text response into real fields. Claude's text is always at `content[0].text`.
- **Set / Edit Fields node** maps the parsed fields into named workflow outputs.

## Prompt patterns (the actual subject matter here)

The workflows are a progression in prompt engineering for **reliable structured JSON** out of Claude:
- `Phase1-ClaudeAPI-Test` — bare "say hello" call; response consumed directly as text.
- `Phase1-StructuredJSON-Test` — asks for flat JSON with an explicit "ONLY the JSON object, no markdown, no backticks, no preamble" instruction so `JSON.parse` succeeds.
- `Phase1-StructuredJSON-Test2` — adds XML-tagged sections (`<prospect_statement>`, `<instructions>`, `<example>`), step-by-step reasoning guidance, a one-shot example, and an explicit output schema (arrays + enum for `urgency`).

When adding a workflow, keep the "raw JSON, no fences" instruction in the prompt — the downstream Code node does an unguarded `JSON.parse` and will throw on any markdown fencing or preamble.

## Conventions

- Credentials are referenced by n8n internal ID (`"r0C14zXFELMOIUg9"` = "Anthropic API Key"). This ID is instance-specific; on a fresh n8n instance the imported workflow will need the credential re-selected.
- Workflows ship with `"active": false` — they're manual-trigger prototypes, not deployed automations.
