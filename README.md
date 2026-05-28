# nark-action

GitHub Action that checks TypeScript projects for missing dependency error handling against 169+ npm package profiles.

Wraps [`nark`](https://github.com/nark-sh/nark) so you can add a single line to your CI workflow.

## Quick start

```yaml
- uses: nark-sh/nark-action@v1
```

That's it. nark auto-detects your `tsconfig.json` and scans your project.

## Full example

```yaml
name: CI
on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Lint
        run: npx eslint .

      - name: Test
        run: npx vitest run

      - name: Check error handling
        uses: nark-sh/nark-action@v1
```

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `tsconfig` | Path to tsconfig.json (auto-detected if omitted) | `''` |
| `fail-threshold` | Exit 1 if violations at or above this severity (`error` \| `warning` \| `info`) | `error` |
| `report-only` | Always exit 0 regardless of violations | `false` |
| `diff-base` | If set, only report violations on lines added/modified by `git diff $diff-base..HEAD`. Pair with `github.event.pull_request.base.sha` for PR scans. | `''` |
| `version` | nark version to use | `latest` |

## Examples

### Custom tsconfig path

```yaml
- uses: nark-sh/nark-action@v1
  with:
    tsconfig: ./packages/api/tsconfig.json
```

### Report-only (don't fail the build)

```yaml
- uses: nark-sh/nark-action@v1
  with:
    report-only: 'true'
```

### Fail on warnings too

```yaml
- uses: nark-sh/nark-action@v1
  with:
    fail-threshold: warning
```

### PR diff mode (only flag what the PR changed)

```yaml
- uses: nark-sh/nark-action@v1
  with:
    diff-base: ${{ github.event.pull_request.base.sha }}
```

In `--diff` mode nark only reports violations on lines the PR actually added or modified. Pre-existing violations on untouched lines in touched files are excluded. Same posture CodeRabbit and Greptile take.

### Pin to a specific version

```yaml
- uses: nark-sh/nark-action@v1
  with:
    version: '1.3.0'
```

## What nark checks

nark scans your TypeScript code against profiles for 169+ npm packages (axios, prisma, stripe, pg, ioredis, etc.) and reports places where your code calls library functions without proper error handling.

Learn more: [nark on GitHub](https://github.com/nark-sh/nark) | [nark.sh](https://nark.sh)

## License

AGPL-3.0
