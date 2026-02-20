# api2cli

A Claude Code skill that generates working Node.js CLIs from any API. Point it at API docs, a live URL, or a [peek-api](https://github.com/alexknowshtml/peek-api) capture and get a dual-mode Commander.js CLI that works for both humans and AI agents.

## What It Does

1. **Discovers endpoints** from API docs pages, live probing, or peek-api network captures
2. **Builds an endpoint catalog** with auth, pagination, and rate limit info
3. **Generates a CLI** with one subcommand per endpoint, a full-featured API client, and dual-mode output (human-readable in terminal, JSON envelope when piped)

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
"Build me a CLI for the Stripe API"
"Generate a CLI from these docs: https://docs.example.com/api"
"Turn this peek-api capture into a CLI"
```

Claude will:
1. Ask what API and how to access it
2. Discover endpoints (parses docs, probes live endpoints, reads peek-api captures)
3. Show you the endpoint catalog for review
4. Generate the CLI -- you choose: scaffold into your project or standalone
5. Test that it works

## What Gets Generated

A Commander.js CLI with:
- **Dual-mode output** -- human-readable in terminal, JSON with `next_actions` when piped
- **Self-documenting root** -- run with no args to see all commands
- **Full API client** -- auth, pagination, retry with backoff, rate limiting, caching
- **One subcommand per endpoint** -- `mycli customers list`, `mycli customers get <id>`
- **Error handling** -- agent-friendly errors with `fix` suggestions

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

The agent-first CLI output patterns (JSON envelope, HATEOAS-style `next_actions`, self-documenting root commands, `fix` suggestions on errors) were inspired by [Joel Hooks' agent-first CLI design](https://github.com/joelhooks/joelclaw/blob/main/.agents/skills/cli-design/SKILL.md).

The audience-aware routing (human-first, agent-first, or auto-detecting dual-mode), the CLI generation patterns, and the full API discovery pipeline (docs parsing, active endpoint probing, peek-api network capture) are by [Alex Hillman](https://github.com/alexknowshtml).

## Related

- [peek-api](https://github.com/alexknowshtml/peek-api) -- Discover internal APIs from any website by monitoring network traffic

## License

MIT
