# Omnis skills

## What are skills?

Skills are small instruction sets that help an AI coding assistant work with a specific tool or codebase. To install one, copy its skill folder into the assistant's skills directory, or use the assistant's supported skill installer. Keep the folder structure intact so its instructions and references stay together.

## Skills in this repo

- **omnis** — Guidance for writing, reviewing, and debugging Omnis code, including common language and notation pitfalls.
- **omnis-mcp** — Guidance for operating Omnis Studio libraries through the official Omnis developer MCP, including their behavior and limitations.

## How were the skills created?

The skills came from programming with the Omnis MCP server and logging anything that went wrong. The log was later summarized and categorized into errors specific to the MCP server and errors specific to the programming language. That summary was then used by an AI agent to draft the skills. Hallucinations and duplicate information were removed from the skills afterwards.
