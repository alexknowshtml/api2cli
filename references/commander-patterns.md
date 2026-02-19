# Commander.js Advanced Patterns

## Nested Subcommands

```typescript
const inbox = program.command('inbox').description('Inbox operations');

inbox.command('list')
  .description('List conversations')
  .option('-s, --status <status>', 'Filter by status', 'open')
  .option('-l, --limit <n>', 'Max results', '20')
  .action(listInbox);

inbox.command('search <query>')
  .description('Search conversations')
  .option('--from <email>', 'Filter by sender')
  .action(searchInbox);
```

## Global Options

```typescript
program
  .option('-v, --verbose', 'Verbose output')
  .option('--config <path>', 'Config file path');

// Access in any command via program.opts()
```

## Interactive Prompts

```typescript
import { input, select, confirm } from '@inquirer/prompts';

async function interactiveCreate() {
  const name = await input({ message: 'Name:' });
  const type = await select({
    message: 'Type:',
    choices: [
      { name: 'Task', value: 'task' },
      { name: 'Bug', value: 'bug' }
    ]
  });
  const proceed = await confirm({ message: 'Create?' });
  if (proceed) await createItem({ name, type });
}
```

## Progress Indicators

```typescript
import ora from 'ora';

async function longOperation() {
  const spinner = ora('Fetching data...').start();
  try {
    const result = await fetchData();
    spinner.succeed(`Fetched ${result.length} items`);
    return result;
  } catch (err) {
    spinner.fail('Failed to fetch data');
    throw err;
  }
}
```

## Colored Output

```typescript
import chalk from 'chalk';

console.log(chalk.green('Success:'), 'Operation completed');
console.log(chalk.yellow('Warning:'), 'Rate limit approaching');
console.log(chalk.red('Error:'), 'Authentication failed');
console.log(chalk.dim('Hint:'), 'Use --help for options');
```

## Config File Support

```typescript
import { cosmiconfigSync } from 'cosmiconfig';

function loadConfig() {
  const explorer = cosmiconfigSync('mycli');
  const result = explorer.search();
  return result?.config ?? {};
}

// Supports: .myclirc, .myclirc.json, mycli.config.js, package.json "mycli" key
```

## Piping & Stdin

```typescript
import { stdin } from 'process';

async function readStdin(): Promise<string> {
  if (stdin.isTTY) return '';

  const chunks: Buffer[] = [];
  for await (const chunk of stdin) {
    chunks.push(chunk);
  }
  return Buffer.concat(chunks).toString('utf8');
}

// Usage: echo "data" | mycli process
```

## Tab Completion (using tabtab)

```typescript
import tabtab from 'tabtab';

if (process.argv.includes('--completion')) {
  tabtab.install({
    name: 'mycli',
    completer: 'mycli'
  });
}
```

## Exit Codes

```typescript
// Standard exit codes
const EXIT_SUCCESS = 0;
const EXIT_ERROR = 1;
const EXIT_INVALID_ARGS = 2;
const EXIT_AUTH_FAILED = 3;

process.exit(EXIT_SUCCESS);
```

## Testing CLIs

```typescript
import { execSync } from 'child_process';

describe('mycli', () => {
  it('lists items', () => {
    const output = execSync('npx tsx scripts/mycli.ts list --json').toString();
    const items = JSON.parse(output);
    expect(items).toBeInstanceOf(Array);
  });
});
```
