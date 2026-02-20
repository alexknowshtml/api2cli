# api2cli

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that turns any API into a working CLI, then wraps that CLI in a skill. Point it at API docs, a live URL, or a [peek-api](https://github.com/alexknowshtml/peek-api) capture and get a dual-mode Commander.js CLI plus a ready-to-use Claude Code skill -- so any future Claude session can pick it up and use it without reading the code.

> **Requires [Claude Code](https://docs.anthropic.com/en/docs/claude-code)** -- this is a skill, not a standalone tool. Claude handles the discovery, generation, and wiring.

## What It Does

1. **Discovers endpoints** from API docs pages, live probing, or peek-api network captures
2. **Builds an endpoint catalog** with auth, pagination, and rate limit info
3. **Generates a CLI** with one subcommand per endpoint, a full-featured API client, and dual-mode output (human-readable in terminal, JSON envelope when piped)
4. **Generates a skill** -- a SKILL.md that teaches Claude how to use the CLI, with commands, examples, and common workflows

## Example: Resend API

Here's what it looks like when you point api2cli at the [Resend](https://resend.com) email API.

**You say:**
```
Build me a CLI for the Resend API. Here are the docs: https://resend.com/docs/api-reference
```

**Claude discovers 15 endpoints across 5 resources:**
```
Found 15 endpoints across 5 resources:
  emails (3 endpoints): send, get, batch
  domains (4 endpoints): list, get, create, delete
  api-keys (3 endpoints): list, create, delete
  audiences (3 endpoints): list, get, create
  contacts (2 endpoints): list, create

Ready to generate the CLI?
```

**Claude generates a CLI at `scripts/resend.ts`:**
```bash
# See all commands
$ npx tsx scripts/resend.ts
{
  "ok": true,
  "command": "resend",
  "result": {
    "description": "Resend email API CLI",
    "commands": [
      { "command": "resend emails send", "description": "Send an email" },
      { "command": "resend emails get <id>", "description": "Get email by ID" },
      { "command": "resend domains list", "description": "List all domains" },
      ...
    ]
  }
}

# Send an email
$ npx tsx scripts/resend.ts emails send \
    --to user@example.com \
    --subject "Hello" \
    --html "<p>Hi there</p>"

# List domains (human-readable in terminal)
$ npx tsx scripts/resend.ts domains list
ID          Domain              Status
re_abc123   example.com         verified
re_def456   staging.example.com pending

# Same command piped to an agent (auto-detects, returns JSON)
$ npx tsx scripts/resend.ts domains list | cat
{
  "ok": true,
  "command": "resend domains list",
  "result": { "domains": [...], "count": 2 },
  "next_actions": [
    { "command": "resend domains get re_abc123", "description": "View domain details" },
    { "command": "resend emails send --from noreply@example.com", "description": "Send from verified domain" }
  ]
}
```

**Claude also generates a skill at `.claude/skills/resend/SKILL.md`:**
```markdown
---
name: resend
description: Interact with the Resend email API via CLI. Use when user wants to
  send emails, manage domains, create API keys, manage audiences and contacts.
  Commands: resend emails send, resend domains list, resend contacts create.
---

# Resend CLI

## Setup
export RESEND_API_KEY=re_your_key_here

## Commands
### emails send --to <email> --subject <text> --html <html>
### domains list | get <id> | create --name <domain>
### contacts list --audience-id <id> | create --email <email>
...
```

From here, any Claude session in your project can send emails, manage domains, and work with contacts -- without knowing anything about the Resend API.

## Install

Copy the `skill/` folder into your Claude Code project:

```bash
# Clone this repo
git clone https://github.com/alexknowshtml/api2cli.git

# Copy the skill into your project
cp -r api2cli/skill/ /path/to/your/project/.claude/skills/api2cli/
```

## Usage

In Claude Code, tell Claude what API you want to wrap:

```
"Build me a CLI for the Resend API"
"Generate a CLI from these docs: https://docs.example.com/api"
"Turn this peek-api capture into a CLI"
```

Claude will:
1. Ask what API and how to access it
2. Discover endpoints (parses docs, probes live endpoints, reads peek-api captures)
3. Show you the endpoint catalog for review
4. Generate the CLI -- you choose: scaffold into your project or standalone
5. Test that it works
6. Generate a skill folder so Claude can use the CLI in future sessions

## What Gets Generated

**A CLI** -- Commander.js with:
- **Dual-mode output** -- human-readable in terminal, JSON with `next_actions` when piped
- **Self-documenting root** -- run with no args to see all commands
- **Full API client** -- auth, pagination, retry with backoff, rate limiting, caching
- **One subcommand per endpoint** -- `mycli customers list`, `mycli customers get <id>`
- **Error handling** -- agent-friendly errors with `fix` suggestions

**A skill** -- a `.claude/skills/{service}/SKILL.md` that:
- Lists all available commands with examples
- Includes common multi-step workflows
- Has the right trigger phrases so Claude auto-activates when relevant
- Documents auth setup and agent usage

## Discovery Methods

| Method | Best For |
|--------|----------|
| **Docs parsing** | Public APIs with documentation pages |
| **Active probing** | APIs where you have a base URL and credentials |
| **peek-api capture** | Sites where you need to sniff network traffic to find endpoints |

## What's Inside

```
skill/
  SKILL.md                                # Main skill instructions
  references/
    discovery-strategies.md               # Endpoint discovery patterns and probing techniques
    api-client-template.md                # API client with pagination, retry, rate limiting, caching
    agent-first-patterns.md               # Agent JSON envelope, HATEOAS, context-safe output
    commander-patterns.md                 # Commander.js advanced patterns
```

## Credits

This came from a pattern of wanting to give my AI agent access to apps and discovering it loved working with CLIs. I wanted an easy way to turn any API into a CLI, then wrap that CLI in a skill. The audience routing (human-first, agent-first, dual-mode), the API discovery pipeline ([peek-api](https://github.com/alexknowshtml/peek-api), docs parsing, active probing), and the generation approach all came from that.

The agent-side output patterns -- JSON envelope, HATEOAS-style `next_actions`, self-documenting root commands, and `fix` suggestions on errors -- were inspired by [Joel Hooks' agent-first CLI design](https://github.com/joelhooks/joelclaw/blob/main/.agents/skills/cli-design/SKILL.md).

## Related

- [peek-api](https://github.com/alexknowshtml/peek-api) -- Discover internal APIs from any website by monitoring network traffic

## License

MIT
