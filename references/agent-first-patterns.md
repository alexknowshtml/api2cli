# Agent-First CLI Patterns

Deep dive into designing CLIs that are primarily consumed by AI agents.

Inspired by [Joel Hooks' agent-first CLI design](https://github.com/joelhooks/joelclaw/blob/main/.agents/skills/cli-design/SKILL.md).

## The Core Idea

Agent-first CLIs treat structured JSON as the only output format. The CLI's job is to return data AND tell the caller what it can do next. Every response is self-contained — the agent never needs to read `--help` or guess at available commands.

## HATEOAS: Next Actions

The most valuable pattern. Every response includes `next_actions` — contextual commands the agent can run next. These change based on the current state.

```typescript
// After listing items with zero results
{
  ok: true,
  command: 'mycli list',
  result: { items: [], count: 0 },
  next_actions: [
    { command: 'mycli create', description: 'Create a new item' },
    { command: 'mycli list --include-archived', description: 'Include archived items' },
  ]
}

// After listing items with results
{
  ok: true,
  command: 'mycli list',
  result: { items: [...], count: 15 },
  next_actions: [
    { command: 'mycli show abc123', description: 'View first item details' },
    { command: 'mycli list --status=active', description: 'Filter to active items' },
    { command: 'mycli export', description: 'Export items to file' },
  ]
}
```

The agent never needs to know the CLI's full command tree — each response tells it exactly what's relevant right now.

## Self-Documenting Root

Running the CLI with no arguments returns the full command tree. This is the agent's entry point.

```typescript
program.action(() => {
  const commands = program.commands.map(cmd => ({
    command: `${program.name()} ${cmd.name()}`,
    description: cmd.description(),
    options: cmd.options.map(opt => ({
      flags: opt.flags,
      description: opt.description,
    })),
  }));

  console.log(JSON.stringify({
    ok: true,
    command: program.name(),
    result: {
      description: program.description(),
      version: program.version(),
      commands,
    },
    next_actions: commands.map(c => ({
      command: c.command,
      description: c.description,
    })),
  }));
});
```

## Error Responses with Fix Suggestions

Errors include a `fix` field — plain language guidance on how to resolve the issue. The agent can either follow the fix or present it to the user.

```typescript
function agentError(command: string, message: string, code: string, fix: string, next_actions: any[] = []) {
  console.log(JSON.stringify({
    ok: false,
    command,
    error: { message, code },
    fix,
    next_actions,
  }));
  process.exit(1);
}

// Usage
agentError(
  'mycli deploy',
  'No deployment target specified',
  'MISSING_TARGET',
  'Run "mycli deploy --target=production" or "mycli deploy --target=staging"',
  [
    { command: 'mycli deploy --target=staging', description: 'Deploy to staging' },
    { command: 'mycli deploy --target=production', description: 'Deploy to production' },
    { command: 'mycli config show', description: 'Check current configuration' },
  ]
);
```

## Context-Protecting Output

Large outputs consume agent context window tokens. Truncate by default and point to full data.

```typescript
function truncateResult(items: any[], maxItems = 50) {
  if (items.length <= maxItems) {
    return { items, count: items.length, truncated: false };
  }

  const tmpPath = `/tmp/mycli-results-${Date.now()}.json`;
  writeFileSync(tmpPath, JSON.stringify(items, null, 2));

  return {
    items: items.slice(0, maxItems),
    count: items.length,
    showing: maxItems,
    truncated: true,
    full_results: tmpPath,
  };
}
```

## Reusable Envelope Helpers

Keep envelope construction consistent with shared helpers:

```typescript
interface SuccessResponse {
  ok: true;
  command: string;
  result: Record<string, any>;
  next_actions: Array<{ command: string; description: string }>;
}

interface ErrorResponse {
  ok: false;
  command: string;
  error: { message: string; code: string };
  fix: string;
  next_actions: Array<{ command: string; description: string }>;
}

function success(command: string, result: Record<string, any>, next_actions: Array<{ command: string; description: string }> = []): SuccessResponse {
  return { ok: true, command, result, next_actions };
}

function error(command: string, message: string, code: string, fix: string, next_actions: Array<{ command: string; description: string }> = []): ErrorResponse {
  return { ok: false, command, error: { message, code }, fix, next_actions };
}

// Usage in commands
console.log(JSON.stringify(success('mycli list', { items, count: items.length }, [
  { command: `mycli show ${items[0]?.id}`, description: 'View first item' },
])));
```

## When to Use Agent-First vs Dual-Mode

**Pure agent-first** when:
- The CLI is only ever called by Claude agents or automation
- Human debugging can use `| jq` to read output
- Examples: internal system CLIs, scheduled job scripts, API wrappers for agent consumption

**Dual-mode** when:
- Alex might run it directly in a terminal
- The output needs to be scannable by humans (status checks, health reports)
- Examples: `bd` (beads), project management CLIs, operational tools
