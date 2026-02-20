# api2cli

Patterns for building Node.js CLI tools that know their audience. Built with [Commander.js](https://github.com/tj/commander.js).

## The Problem

Most CLIs are built for one audience. Human users get pretty output but scripts can't parse it. Agent-consumed tools return raw JSON but are painful to use in a terminal. The fix is usually a `--json` flag bolted on as an afterthought.

**api2cli** starts with a different question: **who's using this CLI?**

## Three Modes

**Human** -- Readable output, color, tables, spinners. `--json` flag for scripts.

**Agent** -- JSON-only with a standard envelope. Every response includes `next_actions` (HATEOAS-style), errors include a `fix` field, and the root command self-documents the full command tree. Output is context-safe (truncated with pointers to full data).

**Both** -- Auto-detects via `process.stdout.isTTY`. Terminal users get formatted output, piped consumers get the JSON envelope. Same data, different rendering, no flags needed.

## What's Inside

```
skill/
  SKILL.md                             # Full guide with code examples for all three modes
  references/
    commander-patterns.md              # Advanced Commander.js patterns (nested commands, prompts, color, config)
    api-client-template.md             # REST API client boilerplate (pagination, retry, caching, rate limiting)
    agent-first-patterns.md            # Deep dive into agent-first CLI design (HATEOAS, envelopes, context protection)
```

## Using as a Claude Code Skill

Copy the `skill/` folder into your Claude Code skills directory:

```bash
cp -r skill/ /path/to/your/project/.claude/skills/cli-builder/
```

Claude Code will automatically pick up the skill and apply these patterns when building CLI tools.

## The Agent JSON Envelope

The key pattern for agent-consumed CLIs:

```typescript
// Success
{ ok: true, command: "mycli list", result: { items: [...], count: 15 }, next_actions: [...] }

// Error
{ ok: false, command: "mycli list", error: { message: "...", code: "FETCH_FAILED" }, fix: "Check API key", next_actions: [...] }
```

Every response tells the agent what happened and what it can do next. Errors suggest how to fix them. Large outputs are truncated with a path to the full file.

## Inspiration

- [Joel Hooks' agent-first CLI design](https://github.com/joelhooks/joelclaw/blob/main/.agents/skills/cli-design/SKILL.md)

## License

MIT
