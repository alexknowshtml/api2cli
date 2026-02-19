# api2cli

A guide for building Node.js CLI tools with [Commander.js](https://github.com/tj/commander.js) that are audience-aware -- optimized for humans, AI agents, or both.

## The First Question: Who's Using This CLI?

Before writing any CLI, decide who's consuming it:

- **Human** -- Terminal users who read output and appreciate color and formatting
- **Agent** -- AI agents parsing structured output to decide next steps
- **Both** -- Auto-detects context and renders accordingly

This choice drives output format, error handling, and discoverability patterns.

---

## Quick Start: Human CLI

For tools people run directly in a terminal.

```typescript
#!/usr/bin/env npx tsx
import { Command } from 'commander';

const program = new Command();

program
  .name('mycli')
  .description('What this CLI does')
  .version('1.0.0');

program
  .command('list')
  .description('List items')
  .option('-j, --json', 'Output as JSON')
  .action(async (options) => {
    const items = await fetchItems();
    if (options.json) {
      console.log(JSON.stringify(items, null, 2));
    } else {
      items.forEach(i => console.log(`${i.id}: ${i.name}`));
    }
  });

program.parse();
```

**Human CLI principles:**
1. Human-readable output by default, `--json` flag for structured output
2. Color, tables, spinners, progress bars are welcome
3. Interactive prompts for complex operations
4. `--help` for discoverability

## Quick Start: Agent CLI

For tools consumed by Claude agents or automated pipelines.

```typescript
#!/usr/bin/env npx tsx
import { Command } from 'commander';

const program = new Command();

program
  .name('mycli')
  .description('What this CLI does')
  .version('1.0.0');

// Root command returns full command tree (self-documenting)
program.action(() => {
  console.log(JSON.stringify({
    ok: true,
    command: 'mycli',
    result: {
      description: 'What this CLI does',
      version: '1.0.0',
      commands: [
        { command: 'mycli list', description: 'List items' },
        { command: 'mycli list --status=active', description: 'List active items' },
      ]
    },
    next_actions: [
      { command: 'mycli list', description: 'List all items' },
      { command: 'mycli health', description: 'Check system health' },
    ]
  }));
});

program
  .command('list')
  .description('List items')
  .option('-s, --status <status>', 'Filter by status')
  .action(async (options) => {
    try {
      const items = await fetchItems(options.status);
      console.log(JSON.stringify({
        ok: true,
        command: 'mycli list',
        result: { items, count: items.length },
        next_actions: [
          { command: `mycli show ${items[0]?.id}`, description: 'View first item details' },
          { command: 'mycli list --status=active', description: 'Filter to active items' },
        ]
      }));
    } catch (err) {
      console.log(JSON.stringify({
        ok: false,
        command: 'mycli list',
        error: { message: err.message, code: 'FETCH_FAILED' },
        fix: 'Check that the API is running and MYSERVICE_API_KEY is set',
        next_actions: [
          { command: 'mycli health', description: 'Check API connectivity' },
        ]
      }));
      process.exit(1);
    }
  });

program.parse();
```

**Agent CLI principles:**
1. **JSON always** -- Every command returns a JSON envelope. No plain text, no color codes, no tables
2. **HATEOAS** -- Every response includes `next_actions` telling the agent what to do next
3. **Self-documenting root** -- Running the command with no args returns the full command tree
4. **Errors suggest fixes** -- Error responses include a `fix` field with actionable guidance
5. **Context-protecting output** -- Truncate large results, include file paths to full data

## Quick Start: Dual-Mode (Both)

For tools used by both humans and agents. Auto-detects based on terminal context.

```typescript
#!/usr/bin/env npx tsx
import { Command } from 'commander';

// Detect if output is being piped/consumed by another program
const isAgent = !process.stdout.isTTY;

// Standard envelope for agent output
function respond(data: { command: string; result: any; next_actions?: any[] }) {
  if (isAgent) {
    console.log(JSON.stringify({ ok: true, ...data }));
  } else {
    // Human-friendly rendering -- customize per command
    return data.result;
  }
}

function respondError(data: { command: string; error: string; code: string; fix: string; next_actions?: any[] }) {
  if (isAgent) {
    console.log(JSON.stringify({
      ok: false,
      command: data.command,
      error: { message: data.error, code: data.code },
      fix: data.fix,
      next_actions: data.next_actions ?? []
    }));
  } else {
    console.error(`Error: ${data.error}`);
    console.error(`Fix: ${data.fix}`);
  }
  process.exit(1);
}

const program = new Command();
program.name('mycli').description('What this CLI does').version('1.0.0');

program
  .command('list')
  .description('List items')
  .action(async () => {
    const items = await fetchItems();

    const result = respond({
      command: 'mycli list',
      result: { items, count: items.length },
      next_actions: [
        { command: `mycli show ${items[0]?.id}`, description: 'View first item' },
      ]
    });

    // Human rendering (only runs in TTY mode)
    if (result) {
      items.forEach(i => console.log(`${i.id}: ${i.name}`));
    }
  });

program.parse();
```

**Dual-mode principles:**
1. `process.stdout.isTTY` determines rendering -- no flags needed
2. Agent callers get the JSON envelope with `next_actions`
3. Terminal users get readable, formatted output
4. Error handling covers both paths
5. Both audiences get the same underlying data

---

## Agent JSON Envelope

Standard response format for agent-consumed CLIs:

```typescript
// Success
{
  ok: true,
  command: string,         // The command that was run
  result: object,          // Command-specific payload
  next_actions: Array<{    // What the agent can do next
    command: string,
    description: string
  }>
}

// Error
{
  ok: false,
  command: string,
  error: {
    message: string,
    code: string           // Machine-readable error code
  },
  fix: string,             // Plain language: how to resolve this
  next_actions: Array<{
    command: string,
    description: string
  }>
}
```

**Context-protecting output rules (agent mode):**
- Truncate lists beyond 50 items by default
- Include total count and a file path to full results when truncated
- Never dump unbounded logs or raw API responses

## Core Principles

1. **Shebang + tsx** -- Use `#!/usr/bin/env npx tsx` for direct execution without build step
2. **Subcommands** -- Group related operations: `cli inbox list`, `cli inbox search`
3. **Fail fast** -- Validate config/auth early, exit with clear error messages

## Anti-Patterns

**Human CLIs:**
- Don't require `--json` to get structured data when the primary consumer is a script
- Don't output only JSON with no human formatting

**Agent CLIs:**
- Don't output plain text, ANSI colors, or table formatting
- Don't require `--help` reading for discoverability -- root command shows everything
- Don't output errors without a `fix` suggestion
- Don't dump unbounded output -- truncate and point to full file

**Dual-Mode CLIs:**
- Don't force users to pass `--json` or `--human` flags -- detect automatically
- Don't have divergent behavior between modes (same data, different rendering)

## Naming Conventions

- **Commands:** lowercase nouns/verbs, no hyphens (`send`, `status`, `health`)
- **Subcommands:** natural hierarchy (`mycli inbox list`, `mycli memory search`)
- **Flags:** `--kebab-case` (`--max-results`, `--follow`)
- **Short flags:** common options (`-s` for `--status`, `-f` for `--follow`)

## File Structure

```
scripts/
  mycli.ts           # Main entry point
  mycli/
    commands/        # Subcommand implementations
    lib/             # Shared utilities (api client, formatters, envelope helpers)
```

## Checklist for New Commands

- [ ] Audience identified (human / agent / both)
- [ ] Returns appropriate format for audience
- [ ] Agent mode: JSON envelope with `next_actions`
- [ ] Agent mode: errors include `fix` field
- [ ] Agent mode: root command lists full command tree
- [ ] Agent mode: output is context-safe (truncated if large)
- [ ] Human mode: readable formatting with optional `--json`
- [ ] Dual mode: auto-detects via `isTTY`
- [ ] Works when piped
- [ ] Builds/runs without compilation step

## Reference Docs

- [Commander.js Patterns](references/commander-patterns.md) -- Advanced Commander.js patterns for human CLIs
- [REST API Client Template](references/api-client-template.md) -- Boilerplate for CLIs that wrap REST APIs
- [Agent-First Patterns](references/agent-first-patterns.md) -- Deep dive into agent-first CLI design

## Using as a Claude Skill

Drop the contents of this repo into your `.claude/skills/` directory:

```
.claude/skills/cli-builder/
  SKILL.md              # Copy of README.md (or symlink)
  references/
    commander-patterns.md
    api-client-template.md
    agent-first-patterns.md
```

Claude Code will automatically pick up the skill and use these patterns when building CLI tools.

## Inspiration

- [Joel Hooks' agent-first CLI design](https://github.com/joelhooks/joelclaw/blob/main/.agents/skills/cli-design/SKILL.md)

## License

MIT
