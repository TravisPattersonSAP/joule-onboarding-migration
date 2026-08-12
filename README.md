# Joule Onboarding & Migration

A skill for [Joule Work Desktop](https://joule.only.sap) that guides anyone
through setting up Joule for the first time or migrating from another AI tool.

## What it does

Supports three paths:

- **Branch A — Technical Agent:** Migrates from Claude Code, Cursor, Windsurf,
  Cline, Roo Code, GitHub Copilot, Codex, Gemini CLI, Aider, or Continue.dev.
  Reads your existing memory/rules files, inventories your skills, maps your
  MCP connections, and writes a full migration report.

- **Branch B — Chat AI:** Imports custom instructions and conversation history
  from ChatGPT, Google Gemini, or Microsoft Copilot into Joule memory.

- **Branch C — New User:** Guided interview that builds your Joule memory from
  scratch — no prior AI tool required.

Progress is saved to a checkpoint file after every action, so the skill can
resume exactly where it left off if interrupted.

## How to trigger it

Say any of:
- `migrate from Claude` / `migrate from Cursor` / `migrate from ChatGPT`
- `help me set up Joule`
- `I'm new to AI agents`
- `set up Joule from scratch`

## Install

```bash
npx skills add TravisPattersonSAP/joule-onboarding-migration

